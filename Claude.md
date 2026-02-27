# HAMi 核心代码列表与技术细节详解

## 项目概述

HAMi (Heterogeneous AI Computing Virtualization Middleware) 是 CNCF 沙盒项目，是一个 **Kubernetes 异构设备（GPU/NPU/etc）虚拟化中间件**。它通过三大组件实现 GPU 等设备的共享、隔离与调度。

---

## 一、系统架构总览

```
[Pod 提交]
    |
    v
[MutatingWebhook] ← pkg/scheduler/webhook.go
    | 注入调度器名称、修改资源请求
    v
[Scheduler Extender] ← pkg/scheduler/scheduler.go
    | Filter (选节点) + Bind (绑定节点)
    v
[Device Plugin] ← cmd/device-plugin/nvidia/
    | 向 Kubelet 注册设备资源、响应 Allocate 请求
    v
[vGPU Monitor] ← cmd/vGPUmonitor/
    | 通过 mmap 共享内存监控容器 GPU 用量，实现优先级与限流反馈
    v
[Container] ← lib/nvidia/ld.so.preload → libvgpu.so (C 库)
    内核级显存/算力隔离
```

---

## 二、核心代码文件清单

### 🔷 1. 设备抽象层（Device Interface）

#### `pkg/device/devices.go` — **最核心的接口定义文件**

这是整个异构设备抽象的基石：

```go
type Devices interface {
    CommonWord() string
    MutateAdmission(ctr *corev1.Container, pod *corev1.Pod) (bool, error)
    CheckHealth(devType string, n *corev1.Node) (bool, bool)
    NodeCleanUp(nn string) error
    GetResourceNames() ResourceNames
    GetNodeDevices(n corev1.Node) ([]*DeviceInfo, error)
    LockNode(n *corev1.Node, p *corev1.Pod) error
    ReleaseNodeLock(n *corev1.Node, p *corev1.Pod) error
    GenerateResourceRequests(ctr *corev1.Container) ContainerDeviceRequest
    PatchAnnotations(pod *corev1.Pod, annoinput *map[string]string, pd PodDevices) map[string]string
    ScoreNode(node *corev1.Node, podDevices PodSingleDevice, previous []*DeviceUsage, policy string) float32
    AddResourceUsage(pod *corev1.Pod, n *DeviceUsage, ctr *ContainerDevice) error
    Fit(devices []*DeviceUsage, request ContainerDeviceRequest, pod *corev1.Pod, nodeInfo *NodeInfo, allocated *PodDevices) (bool, map[string]ContainerDevices, string)
}
```

**关键数据结构：**

| 类型 | 说明 |
|------|------|
| `DeviceInfo` | 节点上单块设备的静态信息（UUID、总显存、核数、NUMA 亲和性） |
| `DeviceUsage` | 调度时的设备实时使用状态（已用显存、核数、占用的 Pod） |
| `ContainerDevice` | 分配给某个容器的一块设备（UUID + 使用量） |
| `PodDevices` | `map[devType]PodSingleDevice`，一个 Pod 的全部设备分配结果 |
| `ContainerDeviceRequest` | 容器的资源请求（数量 Nums、显存 Memreq、算力 Coresreq） |

---

#### `pkg/device/devices.go` — Annotation 编解码系列函数

设备分配信息通过 **Kubernetes Pod/Node Annotations** 传递，编码协议是自定义的分隔符格式：

```go
// 节点设备编码：UUID,Count,Mem,Core,Type,Numa,Health,Index,Mode:UUID,...
func EncodeNodeDevices(dlist []*DeviceInfo) string

// Pod 容器设备编码：UUID,Type,UsedMem,UsedCores:UUID,...
func EncodeContainerDevices(cd ContainerDevices) string

// 容器间用 ; 分隔，设备间用 : 分隔
const OneContainerMultiDeviceSplitSymbol = ":"
const OnePodMultiContainerSplitSymbol   = ";"
```

---

### 🔷 2. NVIDIA GPU 设备实现

#### `pkg/device/nvidia/device.go` — **最复杂的设备实现**

**关键常量（Annotation Keys）：**
```go
const HandshakeAnnos       = "hami.io/node.nvidia.registry.time"
const RegisterAnnos        = "hami.io/node.nvidia.device-register"
const RegisterGPUPairScore = "hami.io/node.nvidia.device-pair-score"
const NvidiaGPUDevice      = "NVIDIA"
const MigMode              = "mig"
const HamiCoreMode         = "hami-core"
const MpsMode              = "mps"
```

**`Fit()` 函数** — 核心设备分配算法（L749-885），逐个检查设备是否满足请求：

```
检查顺序：
1. 设备健康状态 (!dev.Health → skip)
2. 类型/UUID/NUMA 亲和性过滤
3. 时间片配额检查 (Count > Used)
4. Quota 检查 (fitQuota)
5. 显存检查 (Totalmem - Usedmem >= memreq)
6. 算力检查 (Totalcore - Usedcores >= Coresreq)
7. 独占模式检查 (Coresreq=100 → 排他)
8. MIG CustomFilterRule
9. 拓扑感知 (topology-aware) 最优组合选择
```

**MIG（Multi-Instance GPU）支持** — `AddResourceUsage()` (L677-726)：
- 自动选择 MIG Profile（vir02/vir04/vir08/vir16）
- UUID 追加 `[templateIdx-instanceIdx]` 格式标记 MIG 实例

**拓扑感知调度** — `computeBestCombination()` / `computeWorstSingleCard()`：
- 多卡请求时选择 NVLink 连接分数最高的组合
- 单卡请求时选择与其他卡连接最差的（降低干扰）

---

### 🔷 3. 调度器核心

#### `pkg/scheduler/scheduler.go` — **Scheduler Extender 核心**

```go
type Scheduler struct {
    *nodeManager           // 管理节点设备信息
    podManager             // 跟踪已分配 Pod
    quotaManager           // ResourceQuota 管理
    leaderManager          // HA 主从选举
    cachedstatus           // Filter 时返回的节点状态缓存
    overviewstatus         // 监控用全量节点状态
    // ...
}
```

**两大入口：**

| 函数 | 职责 |
|------|------|
| `Filter()` (L644-716) | 从候选节点中选出最优节点，打 Annotation，将设备分配写入 Pod |
| `Bind()` (L584-642) | 加 NodeLock → Patch Pod Annotation(allocating) → 调用 k8s Bind API |

**`Filter()` 完整流程：**
```
1. 解析 Pod 资源请求 Resourcereqs()
2. 删除旧的 PodManager 缓存（防止重调度污染）
3. getNodesUsage() 构建所有候选节点的设备用量快照
4. calcScore() 并发计算每个节点得分 + 设备适配
5. 按策略(binpack/spread)排序，取最高分节点
6. PatchAnnotations 写入设备分配结果
7. AddPod 到 PodManager 内存缓存
8. PatchPodAnnotations 持久化到 k8s
```

---

#### `pkg/scheduler/score.go` — 评分引擎

**`calcScore()`** — 并发评分（每个节点一个 goroutine）：

```
对每个节点：
1. ComputeDefaultScore() 计算节点基础分 = Weight*(used/total + core/total + mem/total)
2. SnapshotDevice() 快照当前设备状态（用于回滚）
3. fitInDevices() 尝试在节点上分配设备，成功则加入候选
4. OverrideScore() 加上设备级别附加分（当前 nvidia 返回 0）
```

**`fitInDevices()`** — 设备适配：
```
1. 计算每块设备的 ComputeScore (GPU 级别打分)
2. 按策略排序设备列表 (binpack：分高排前；spread：分低排前)
3. 调用 device.Fit() 尝试分配
4. 成功后调用 AddResourceUsage() 更新内存中的设备使用量
```

---

#### `pkg/scheduler/nodes.go` — 节点状态管理

```go
type NodeUsage struct {
    Node    *corev1.Node
    Devices policy.DeviceUsageList  // 带排序策略的设备列表
}

type nodeManager struct {
    nodes map[string]*device.NodeInfo
    mutex sync.RWMutex
}
```

`ListNodes()` 返回深拷贝，防止并发调度时数据竞争。

---

### 🔷 4. 调度策略层

#### `pkg/scheduler/policy/gpu_policy.go` — GPU 级别排序

```
评分公式（Weight=10）：
Score = 10 * ((req+used)/count + (coreReq+usedCore)/totalCore + (memReq+usedMem)/totalMem)
binpack: 分高的排前面（优先打满）
spread:  分低的排前面（优先分散）
```

#### `pkg/scheduler/policy/node_policy.go` — 节点级别排序

```
节点基础分 = 10 * (used/total + usedCore/totalCore + usedMem/totalMem)
binpack: NodeScoreList.Less → 分低的排前（最终取最后一个 = 分最高 → 打满）
spread:  分高的排前（分最低的节点 = 最空闲 → 分散）
```

注意：`sort.Sort()` 后取 `NodeList[len-1]`，binpack 取最高分（最满），spread 取最低分（最空）。

---

### 🔷 5. MutatingWebhook

#### `pkg/scheduler/webhook.go` — Pod 准入变更

```
处理流程：
1. 解码 Pod
2. 跳过已有其他调度器的 Pod
3. 遍历容器，调用各设备的 MutateAdmission()
   - 注入环境变量（CUDA_TASK_PRIORITY, GPU_CORE_UTILIZATION_POLICY）
   - 补全默认 GPU 数量（只有显存/算力请求时）
   - 设置 RuntimeClassName
4. 修改 Pod.Spec.SchedulerName
5. 返回 JSON Patch
```

---

### 🔷 6. 配额管理

#### `pkg/device/quota.go` — ResourceQuota 感知

```
FitQuota() 在 Fit() 中被调用，检查 namespace 级别配额：
- 显存配额：limits.nvidia.com/gpumem
- 算力配额：limits.nvidia.com/gpucores
QuotaManager 是单例（sync.Once），线程安全
```

---

### 🔷 7. NodeLock 机制

#### `pkg/util/nodelock/nodelock.go` — 防止并发分配冲突

```
锁格式：Annotation "hami.io/mutex.lock" = "RFC3339时间,namespace,podName"
细粒度：每个 node 有独立的内存 mutex (nodeLockManager)
超时：5 分钟后过期自动释放（HAMI_NODELOCK_EXPIRE 可配置）
悬空锁检测：上一个 Pod 不存在时自动释放
```

**用途：** `Bind()` 前锁定节点，防止多个调度器同时向同一节点绑定多个 GPU 任务。

---

### 🔷 8. HA 主从选举

#### `pkg/util/leaderelection/leaderelection.go` — 基于 Lease 的主从

```
监听 Coordination/v1/Lease 资源变化
判断自己是否 Leader：HolderIdentity.HasPrefix(hostname)
且 Lease 未过期 (observedTime + LeaseDuration > now)
OnStartedLeading: 通知 scheduler 开始注册节点
OnStoppedLeading: 将 synced 置 false，停止服务调度请求
```

---

### 🔷 9. vGPU 监控与反馈

#### `cmd/vGPUmonitor/feedback.go` — 优先级调度反馈

```
通过共享内存（mmap）读写容器内 libvgpu.so 的控制块：
- GetRecentKernel() / SetRecentKernel()  → 内核活跃度（负数=被阻塞）
- GetUtilizationSwitch() / SetUtilizationSwitch() → 算力限制开关
- GetPriority() → 任务优先级（0-1）

Observe() 每 5 秒：
1. 统计每块 GPU 上各优先级的活跃任务数
2. 高优先级任务存在时，低优先级任务的 UtilizationSwitch 置 1（触发限流）
3. 无法调度时将 RecentKernel 置 -1（触发阻塞）
```

#### `pkg/monitor/nvidia/cudevshr.go` — 共享内存映射

```
通过 syscall.Mmap + unsafe.Pointer 直接读写容器的 .cache 文件
支持 v0（固定大小 1197897 字节）和 v1（majorVersion=1）两种格式
magic flag = 19920718 验证文件有效性
监控路径：$HOOK_PATH/containers/{podUID}_{containerName}/
```

---

### 🔷 10. Device Plugin 层

#### `cmd/device-plugin/nvidia/main.go` — Kubelet Device Plugin

```
启动流程：
1. inotify 监听 /var/lib/kubelet/device-plugins/ (kubelet socket)
2. kubelet 重启时（socket 重建）自动重启所有插件
3. SIGHUP → 重启；SIGTERM → 优雅退出
4. 通过 NVML 库发现 GPU 设备
5. 支持 CDI（Container Device Interface）现代化设备注入
6. 支持 MIG(none/single/mixed)、MPS、时间片共享
```

#### `pkg/device-plugin/nvidiadevice/nvinternal/plugin/` — gRPC 实现

实现 `kubeletdevicepluginv1beta1` 接口（`ListAndWatch`、`Allocate`、`GetPreferredAllocation`）。

---

### 🔷 11. 全局配置注册

#### `pkg/scheduler/config/config.go` — 设备注册总线

```
InitDevicesWithConfig() 统一初始化所有异构设备：
NVIDIA / Cambricon / HYGON / Iluvatar / MThreads /
MetaX / Kunlun / AWSNeuron / AMD / Ascend (HUAWEI)

每个设备注册到全局 device.DevicesMap[commonWord]
默认配置内嵌在代码中（InitDefaultDevices），也可通过 YAML 文件覆盖
```

---

## 三、关键技术细节总结

### 🔑 T1. Annotation-Driven 状态传递

所有调度状态通过 Pod/Node Annotation 传递，**没有 CRD**，这是核心设计决策：

| Annotation | 位置 | 含义 |
|---|---|---|
| `hami.io/vgpu-devices-to-allocate` | Pod | 待分配设备列表 |
| `hami.io/vgpu-devices-allocated` | Pod | 已分配设备列表 |
| `hami.io/vgpu-node` | Pod | 目标节点名 |
| `hami.io/bind-phase` | Pod | allocating/failed/success |
| `hami.io/node.nvidia.device-register` | Node | GPU 设备列表（JSON） |
| `hami.io/node.nvidia.registry.time` | Node | 握手时间戳 |
| `hami.io/mutex.lock` | Node | 节点并发锁 |

### 🔑 T2. 双层评分体系

```
节点层(NodeScore):
  Score = 10 * (used/total + usedCore/totalCore + usedMem/totalMem)
  binpack → 优先选高分（最满的节点）
  spread  → 优先选低分（最空的节点）

GPU 层(DeviceListsScore):
  Score = 10 * ((reqCount+used)/count + (reqCore+usedCore)/totalCore + (reqMem+usedMem)/totalMem)
  binpack → 优先分配高分 GPU（打满单卡）
  spread  → 优先分配低分 GPU（分散使用）
```

### 🔑 T3. 共享内存隔离机制

```
libvgpu.so (通过 ld.so.preload 注入容器)
  ↕ mmap 共享内存文件 (.cache)
vGPUmonitor (宿主机 DaemonSet)
  - 读取：GPU 使用量（显存、SM 利用率）
  - 写入：UtilizationSwitch（限速开关）、RecentKernel（阻塞信号）
```

这是**无侵入式**隔离的关键：应用程序不需要修改，通过 `LD_PRELOAD` 劫持 CUDA 调用。

### 🔑 T4. 设备健康检查握手协议

```
device-plugin 启动 → 向 Node Annotation 写 "Requesting_<时间戳>"
scheduler 检测到 "Requesting_" → 60 秒内认为健康，等待上报
device-plugin 完成注册 → 写入设备列表 JSON
scheduler 检测到变化 → 更新 nodeManager 内部缓存
device-plugin 停止 → 写 "Deleted_<时间戳>"
scheduler 检测到 "Deleted_" → 标记 needUpdate=false
```

### 🔑 T5. 并发调度安全

```
三层并发保护：
1. nodeManager.mutex (RWMutex) - 保护节点缓存读写
2. PodManager.mutex (RWMutex) - 保护 Pod 缓存
3. nodelock (per-node Mutex + k8s Annotation) - 跨实例分布式锁

calcScore() 中每个节点用独立 goroutine 并发计算
fitInDevices() 操作的是 node 的本地拷贝（SnapshotDevice 快照）
最终只有选中的节点的 Score 被实际写入 Pod Annotation
```

### 🔑 T6. MIG（Multi-Instance GPU）动态分配

```
MIG UUID 格式：{物理GPU_UUID}[{templateIdx}-{instanceIdx}]
例如：GPU-abc123[0-2] 表示第0号模板的第2个实例

PlatternMIG()       - 将 MIG 模板展开为使用列表
migNeedsReset()     - 检测是否需要重置 MIG 使用列表
AddResourceUsage()  - 分配时更新 MIG 实例的 InUse 状态
CustomFilterRule()  - 调度时检查 MIG 模板是否有空闲实例
```

### 🔑 T7. 拓扑感知调度（Topology-Aware）

```
Node Annotation "hami.io/node.nvidia.device-pair-score" 存储 NVLink 矩阵：
[{"uuid":"GPU-A","score":{"GPU-B":120,"GPU-C":80}}]

单卡请求 → computeWorstSingleCard()：选与其他卡连接最弱的（减少争用）
多卡请求 → computeBestCombination()：遍历所有组合，选 NVLink 得分总和最高的
```

---

## 四、需要重点学习的技术栈

| 领域 | 技术点 | 对应文件 |
|------|--------|---------|
| **Kubernetes 扩展机制** | Scheduler Extender、MutatingWebhook | `pkg/scheduler/` |
| **Informer/Lister 模式** | SharedInformerFactory、ResourceEventHandler | `pkg/scheduler/scheduler.go` |
| **Client-go 高级用法** | MergePatch、Retry、WaitForCacheSync | `pkg/util/util.go`, `nodelock.go` |
| **Device Plugin gRPC** | kubeletdevicepluginv1beta1 协议 | `pkg/device-plugin/nvinternal/plugin/` |
| **NVML 库** | GPU 信息查询 (go-nvml) | `cmd/device-plugin/nvidia/` |
| **CDI 规范** | Container Device Interface | `pkg/device-plugin/nvinternal/cdi/` |
| **mmap 共享内存** | syscall.Mmap + unsafe.Pointer | `pkg/monitor/nvidia/cudevshr.go` |
| **LD_PRELOAD 注入** | 动态库劫持 CUDA 调用 | `lib/nvidia/ld.so.preload` |
| **Prometheus 监控** | 自定义 Collector、Registry | `cmd/vGPUmonitor/metrics.go` |
| **Leader Election** | Coordination/v1 Lease | `pkg/util/leaderelection/` |
| **接口抽象设计** | 多异构设备统一接口 | `pkg/device/devices.go` |
| **并发调度安全** | 分布式锁 + 内存锁组合 | `pkg/util/nodelock/nodelock.go` |

---

## 五、Upstream 同步变更记录（2026-02 批次，共 31 commits）

> 同步自 `Project-HAMi/HAMi@cb077d5`，合并到本 fork `master` 分支。

### 🐛 Bug 修复

#### BUG-1：`calcScore()` 中 `ctrfit` 初始值错误导致 panic（#1626）
- **文件**：`pkg/scheduler/score.go`
- **问题**：`ctrfit` 初始值为 `false`，当 Pod 没有任何设备请求（`resourceReqs` 为 nil）时，`range` 循环不执行，节点被错误排除，并可能触发 panic（issue #1327）。
- **修复**：将初始值改为 `true`。Go 的 `range nil` 安全跳过循环，无设备需求的 Pod 应能调度到任意节点。
```
// 修复前
ctrfit := false
// 修复后
ctrfit := true  // Pod 无设备需求时，默认节点适配
```

#### BUG-2：Webhook 中调度器名称判断运算符优先级错误（#1627）
- **文件**：`pkg/scheduler/webhook.go`
- **问题**：`||` 和 `&&` 混用时缺少括号，导致 `ForceOverwriteDefaultScheduler=false` 时逻辑短路异常，具有资源请求的 Pod 被错误放行而不注入调度器名称。
- **修复**：添加括号明确优先级。
```go
// 修复前（有歧义）
if pod.Spec.SchedulerName != "" &&
    pod.Spec.SchedulerName != corev1.DefaultSchedulerName || !config.ForceOverwriteDefaultScheduler &&
    ...
// 修复后（明确）
if pod.Spec.SchedulerName != "" &&
    (pod.Spec.SchedulerName != corev1.DefaultSchedulerName || !config.ForceOverwriteDefaultScheduler) &&
    ...
```

#### BUG-3：LeaderElection 中 nil 指针 panic（#1603）
- **文件**：`pkg/util/leaderelection/leaderelection.go`
- **问题**：`OnStartedLeading`/`OnStoppedLeading` callback 为 nil 时直接调用会 panic；`isHolderOf()` 未检查 `lease == nil`；`isLeaseValid()` 未检查 `LeaseDurationSeconds == nil`。
- **修复**：所有调用点前加 nil guard；`isHolderOf()` 和 `isLeaseValid()` 加防御性检查。

#### BUG-4：Iluvatar 设备 binpack/spread 策略反转（#1631）
- **文件**：`pkg/device/iluvatar/device.go`
- **问题**：Iluvatar 设备实现中 binpack 和 spread 排序逻辑写反，导
致 binpack 时实际执行 spread 行为。
- **修复**：交换排序比较方向，与 NVIDIA 实现保持一致。

#### BUG-5：统一显存 GPU（GB10/DGX Spark）`GetMemoryInfo` 返回 NOT_SUPPORTED 导致 panic（#1637）
- **文件**：`pkg/device-plugin/nvidiadevice/nvinternal/plugin/register.go`、`cmd/vGPUmonitor/metrics.go`、`pkg/device/nvidia/device.go`
- **问题**：NVIDIA GB10 等统一内存架构 GPU 调用 `nvmlDeviceGetMemoryInfo()` 返回 `ERROR_NOT_SUPPORTED`，原代码直接 panic。
- **修复**：
  - `register.go`：捕获 `NOT_SUPPORTED`，回退到配置文件中的 `PreConfiguredDeviceMemory` 值；若未配置则跳过该设备（`continue`）而非 panic。
  - `metrics.go`：跳过不支持显存查询的设备的内存指标采集。
  - `device.go`：`NodeDefaultConfig` 新增 `PreConfiguredDeviceMemory` 字段，支持 Helm values 按节点配置。

#### BUG-6：`Device_memory_desc_of_container` 指标基数爆炸（#1628）
- **文件**：`cmd/vGPUmonitor/metrics.go`
- **问题**：每个容器的每块设备都生成独立 label 维度组合，随容器数量线性增长，导致 Prometheus 基数爆炸（cardinality explosion），内存占用急剧上升。
- **修复**：统一显存和利用率指标的 label 集合，合并冗余维度，减少时间序列数量。

---

### 🔒 安全加固

#### SEC-1：HTTP 请求体大小限制，防止 DoS（#1620）
- **文件**：`pkg/scheduler/routes/route.go`
- **问题**：`/predicate` 和 `/bind` 两个 HTTP 端点未限制请求体大小，攻击者可发送超大 payload 耗尽内存（issue #554）。
- **修复**：用 `io.LimitReader` 包装 `r.Body`，限制为 1MB。
```go
const maxRequestSize = 1024 * 1024 // 1MB
limitedReader := io.LimitReader(r.Body, maxRequestSize)
body := io.TeeReader(limitedReader, &buf)
```

---

### ✨ 新功能

#### FEAT-1：Webhook 阶段提前检查 ResourceQuota（#1605）
- **文件**：`pkg/scheduler/webhook.go`
- **新增**：`fitResourceQuota()` 函数，在 Pod 准入阶段（Webhook）即检查 namespace 级别的显存/算力配额，配额不足时直接 Deny，避免 Pod 进入调度队列后才失败。
- **当前限制**：仅支持 NVIDIA GPU 设备。
- **流程位置**：位于 `MutateAdmission` 之后、`json.Marshal` 之前。
```go
if !fitResourceQuota(pod) {
    return admission.Denied("exceeding resource quota")
}
```

#### FEAT-2：Ascend 910C SuperPod 模块对儿分配（#1610）
- **文件**：`pkg/device/ascend/device.go`
- **背景**：Ascend 910C 的物理架构中，最小分配单元是一个物理模块（2 个 NPU）。请求 1 个 NPU 实际需要占用 2 个。
- **实现**：
  - `MutateAdmission()` 中检测 `Ascend910C` 设备类型：请求数为 1 时自动扩展为 2；奇数请求（3、5、7…）直接拒绝并返回错误。
  - `Fit()` 中新增 `computeBestCombination910C()`：按物理卡（每卡 2 NPU）分组，选择同一物理模块内的 NPU 对进行分配，保证模块内局部性。
```go
const Ascend910CType = "Ascend910C"
// MutateAdmission 中
if reqNum == 1 {
    reqNum = 2  // 自动扩展到最小分配单元
} else if reqNum%2 != 0 {
    return false, errors.New("Ascend910C device request must be 1 or 2*n")
}
```

#### FEAT-3：Prometheus ServiceMonitor 支持（#1614、#1633）
- **文件**：`charts/hami/templates/scheduler/servicemonitor.yaml`、`charts/hami/templates/device-plugin/servicemonitor.yaml`
- **新增**：Helm chart 中为 scheduler 和 device-plugin 分别添加 `ServiceMonitor` CRD 资源，配合 Prometheus Operator 实现自动服务发现和指标采集。
- **配置**：通过 `values.yaml` 中 `scheduler.serviceMonitor.enabled` 和 `devicePlugin.serviceMonitor.enabled` 开关控制。

#### FEAT-4：指标新增 `device_type` 标签（#1612）
- **文件**：`cmd/scheduler/metrics.go`
- **变更**：调度器指标中所有与设备相关的 Gauge/Counter 新增 `device_type` label，便于在 Grafana 中按设备类型（NVIDIA/Ascend/Cambricon 等）分组展示。

#### FEAT-5：vGPUmonitor 支持自定义 `metrics-bind-address`（#1613）
- **文件**：`cmd/vGPUmonitor/main.go`
- **变更**：新增 `--metrics-bind-address` 命令行参数，允许自定义监控端点监听地址，不再硬编码 `:9394`。

#### FEAT-6：`checkUUID` 提取为公共函数复用（#1622）
- **文件**：`pkg/device/devices.go`（新增）、`pkg/device/nvidia/device.go`（删除私有方法）
- **变更**：将 NVIDIA 私有的 `checkUUID()` 重构为包级公共函数 `device.CheckUUID()`，其他设备厂商实现可直接复用 UUID 白名单/黑名单过滤逻辑。
```go
// 从 nvidia 私有方法提升为 device 包公共函数
func CheckUUID(annos map[string]string, deviceID, useUUIDAnno, noUseUUIDAnno, commonWord string) bool
```

---

### ⚡ 性能优化

#### PERF-1：NodeLock 重试策略改用指数退避（#1663）
- **文件**：`pkg/util/nodelock/nodelock.go`
- **变更**：`DefaultStrategy` 中 `Factor` 从 `1.0`（线性）改为 `2.0`（指数），`Jitter` 从 `0.1` 改为 `0.5`（更大随机抖动），减少高并发场景下多调度器实例争抢节点锁时的惊群效应。
```go
DefaultStrategy = wait.Backoff{
    Steps:    5,
    Duration: 100 * time.Millisecond,
    Factor:   2.0,  // 指数退避：100ms → 200ms → 400ms → 800ms → 1600ms
    Jitter:   0.5,  // ±50% 随机抖动
}
```

---

### 📦 依赖升级

| 依赖 | 旧版本 | 新版本 |
|------|--------|--------|
| `google.golang.org/grpc` | 1.78.0 | 1.79.1 |
| `github.com/NVIDIA/k8s-device-plugin` | — | 最新 |
| `github.com/NVIDIA/nvidia-container-toolkit` | — | 最新 |
| `golang.org/x/tools` | 0.41.0 | 0.42.0 |
| `github.com/onsi/gomega` | 1.39.0 | 1.39.1 |
| `github.com/onsi/ginkgo/v2` | 2.27.5 | 2.28.1 |
| `docker/build-push-action` (CI) | 6.18.0 | 6.19.2 |
| `docker/login-action` (CI) | 3.6.0 | 3.7.0 |
| `aquasecurity/trivy-action` (CI) | 0.33.1 | 0.34.1 |

---

### 🧪 测试覆盖增强

| 新增测试文件/函数 | 覆盖场景 |
|---|---|
| `pkg/scheduler/routes/route_test.go` | `LimitReader` 超大请求体触发 EOF |
| `pkg/scheduler/webhook_test.go` `TestSchedulerNameEmptyNoOverwrite` | `ForceOverwriteDefaultScheduler=false` 时调度器名称注入 |
| `pkg/util/leaderelection/leaderelection_test.go` | nil callback、nil lease、nil HolderIdentity、nil LeaseDurationSeconds 全场景 |
| `pkg/device/ascend/device_test.go` | 910C 奇数请求拒绝、模块对儿分配、`computeBestCombination910C` |