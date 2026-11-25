# Kubernetes v1.38 AI Infra 解读：DRA 三项能力 GA，WAS 补齐积木，推理资源原地伸缩

作者：（待补充）（2026 年 11 月 25 日）

Kubernetes v1.38 预计在 2026 年 12 月 16 日发布。这篇文章写给做 AI Infra 的同学：管 GPU/加速器集群的、做训练和推理平台的、维护调度器和设备驱动的。和通用版前瞻相比，这里只看和 AI 工作负载直接相关的变化，按场景组织。

先说结论，v1.38 对 AI Infra 最重要的是三件事：

1. **设备**：DRA 的可分区设备、设备绑定条件、可消耗容量三项进入 GA，`DynamicResourceAllocation` 门控删除，DRA 始终开启。
2. **调度**：Workload-Aware Scheduling（WAS）的 CompositePodGroup、拓扑感知调度、控制器接入 API 进入 Beta，Job 控制器接入 Beta，Deployment 接入 Alpha；DRA 抢占以 Alpha 加入。
3. **资源弹性**：Pod 级资源和 Pod 级原地调整进入 GA，原地扩容触发的调度器抢占进入 Beta 并默认开启。推理服务扩缩容可以更少地重建 Pod。

另外有两个 Alpha 方向值得提前关注：驱逐前让应用先收尾的 EvictionRequest API，以及面向 Agent 沙箱、warm pool 的动态容器。

本文基于代码冻结（11 月 17 日）后的状态整理，部分特性是以代码冻结例外的形式在最近几天合入的，正式发布前仍可能调整，一切以 v1.38 的 CHANGELOG 和 Release Notes 为准。

> 通用版前瞻（弃用、移除和破坏性变更）见：[Kubernetes v1.38 前瞻：弃用移除、升级必读与重点增强](prerelease.md)；Agent 场景的展开见：[Kubernetes v1.38 Agent 视角解读](agent.md)。

## 目录

- 一张表看重点
- 一、DRA：加速器管理的默认路径
- 二、WAS：从调度单个 Pod 到调度一组 Pod
- 三、推理服务的资源弹性：Pod 级资源与原地调整
- 四、长任务的体面退出：驱逐、节点维护与失败原因
- 五、Agent 与沙箱：动态容器和网络隔离
- 六、大规模 GPU 集群的控制面
- 升级到 v1.38 前的 AI 集群检查清单

## 一张表看重点

| 你要解决的问题 | v1.38 的变化 | KEP | v1.38 阶段 |
| --- | --- | --- | --- |
| 一张 GPU 既能整卡用，也能切分（如 MIG） | DRA 可分区设备 | [4815](https://kep.k8s.io/4815) | GA |
| 多个任务按显存、带宽等容量共用一个设备 | DRA 可消耗容量，容量范围支持小数 | [5075](https://kep.k8s.io/5075) | GA |
| 设备要先准备好（比如通过 fabric 挂接）才能绑定 Pod | DRA 设备绑定条件 | [5007](https://kep.k8s.io/5007) | GA |
| 一个设备同时属于多个拓扑或互联分组 | DRA 列表类型属性 | [5491](https://kep.k8s.io/5491) | Beta |
| 驱动自身要占用主机内存等资源 | DRA 节点可分配资源请求 | [5517](https://kep.k8s.io/5517) | Beta（默认关闭） |
| 高优先级任务抢占低优先级任务的 GPU | DRA 抢占 | [5690](https://kep.k8s.io/5690) | Alpha |
| 用 extended resource 写法申请“几份 GPU” | DRA 扩展资源容量请求 | [6406](https://kep.k8s.io/6406) | Alpha |
| 分布式训练一组 Pod 要么一起起、要么都不占资源 | Gang Scheduling、CompositePodGroup | [4671](https://kep.k8s.io/4671)、[6012](https://kep.k8s.io/6012) | Beta |
| 整组 Pod 放进同一个拓扑域 | 拓扑感知工作负载调度 | [5732](https://kep.k8s.io/5732) | Beta |
| 抢占时按组考虑，而不是逐个 Pod | 工作负载感知抢占 | [5710](https://kep.k8s.io/5710) | Beta |
| 训练 Job、推理 Deployment 接入整组调度 | 控制器接入 API、Job 接入；Deployment 接入 | [6089](https://kep.k8s.io/6089)、[5547](https://kep.k8s.io/5547)、[6276](https://kep.k8s.io/6276) | Beta；Alpha |
| 推理 Pod 里的主容器和多个 sidecar 共享资源预算 | Pod 级资源 | [2837](https://kep.k8s.io/2837) | GA |
| 推理服务扩缩容不重建 Pod | Pod 级原地调整；Deferred 扩容触发调度器抢占 | [5419](https://kep.k8s.io/5419)、[5836](https://kep.k8s.io/5836) | GA；Beta |
| 调整 `/dev/shm` 这类内存卷的大小 | 内存卷动态调整 | [6030](https://kep.k8s.io/6030) | Beta |
| 训练任务被驱逐前先保存 checkpoint | EvictionRequest API | [4563](https://kep.k8s.io/4563) | Alpha |
| GPU 节点维护时，排空进度对外可见 | 节点生命周期条件 | [5683](https://kep.k8s.io/5683) | Alpha |
| 区分训练任务失败的原因 | Job PodFailurePolicy 自定义失败原因 | [4443](https://kep.k8s.io/4443) | Alpha |
| Agent 沙箱、warm pool | 动态容器、动态 node-local 临时卷 | [5972](https://kep.k8s.io/5972)、[6395](https://kep.k8s.io/6395) | Alpha |
| Pod 只用 RDMA 等专用网络，不接入集群网络 | 网络隔离 Pod | [6313](https://kep.k8s.io/6313) | Alpha |

## 一、DRA：加速器管理的默认路径

DRA 核心框架在 v1.34 GA，v1.37 又把 extended resource 兼容路径（KEP-5004）、设备污点（KEP-5055）等推到了 GA。v1.38 再把三项关键能力推到 GA，同时删除了 `DynamicResourceAllocation` 门控：DRA 不再是一个可以关掉的特性。

### 1. 三项 GA：可分区、绑定条件、可消耗容量

| 能力 | 门控 | 解决的问题 |
| --- | --- | --- |
| 可分区设备（KEP-4815） | `DRAPartitionableDevices`，锁定开启 | 驱动在 ResourceSlice 里同时发布整卡和各种分区，每个设备声明自己消耗多少共享计数器。整卡被分配后，它的分区不会再被分配；分区之间只要计数器消耗不超过物理设备，就可以任意组合。GPU 整卡和 MIG 切片因此可以放在同一个资源池里。 |
| 设备绑定条件（KEP-5007） | `DRADeviceBindingConditions`，锁定开启 | 调度器在 PreBind 阶段等待 ResourceClaim status 里的条件变为 true，再绑定 Pod。出现绑定失败条件，或在 `bindingTimeout`（默认 10 分钟）内条件仍未满足，调度器会清除分配并重新调度。典型场景是可组合基础设施里通过 PCIe/CXL 交换机按需挂接的 GPU，以及使用前需要重新烧写的 FPGA。 |
| 可消耗容量（KEP-5075） | `DRAConsumableCapacity`，默认开启 | 设备设置 `allowMultipleAllocations: true` 后，多个独立分配可以从同一个设备的容量里扣减，调度器记录剩余量。比如一个 48 GiB 的加速器分给 16 GiB 和 24 GiB 两个请求后，还剩 8 GiB。同时 `DRAFractionalCapacityRange` 也已 GA，容量请求策略的最小值、最大值和步长可以使用最多三位小数。 |

需要注意的是，调度器负责的是容量记账，显存、带宽等资源的实际隔离和限制仍由驱动和硬件实现。

### 2. Beta：属性、驱动开销、节点操作与调度效率

- **列表类型属性（KEP-5491，`DRAListTypeAttributes`，默认开启）**：设备属性可以是一个列表，一个设备可以同时属于多个分组。`matchAttribute` 要求选中的设备至少共享一个值，`distinctAttribute` 要求任意两个设备都不共享值，单值属性视为只含一个元素的列表。**注意**：某个属性从单值改成列表后，CEL 选择器里的 `==` 要改成 `.includes()`。
- **节点可分配资源请求（KEP-5517，`DRANodeAllocatableResources`）**：驱动可以声明设备使用时额外占用的主机资源，比如主机内存开销，v1.38 新增对 ephemeral storage 的支持（此前已支持 CPU、内存和 hugepages）。这个 Beta 在 v1.38 默认关闭；`--min-compatibility-version` 设为 1.38 或更高的组件默认开启，但 kubelet 没有这个参数，需要在节点上显式开启。调度器只会把这类 Pod 放到 kubelet 声明了 `DRAExtraNodeAllocatableResources` 节点特性的节点上。Pod status 字段从 `.status.nodeAllocatableResourceClaimStatuses` 改为 `.status.extraNodeAllocatableResources`，v1.37 用过 Alpha 的 Pod 升级后需要重建。
- **可选节点操作（KEP-5945，`DRAOptionalNodeOperations`，默认开启）**：ResourceSlice 的 `spec.skipNodeOperations` 可以让 kubelet 跳过节点侧的 prepare 和清理：`["*"]` 两者都跳过，`["NodeUnprepareResources"]` 只跳过清理。只有 claim 里所有已分配设备都声明跳过时，kubelet 才会跳过。调度器只会把这类 Pod 放到声明了 `DRAOptionalNodeOperations` 节点特性的节点上。
- **PreQueueingHint（KEP-6132，`SchedulerPreQueueingHints`，默认开启）**：ResourceClaim 变化时，DRA 插件只重新入队引用该 claim 的 Pod，不再把所有不可调度的 Pod 都重新入队。

### 3. Alpha：共享容量、扩展资源份额、可用性视图与抢占

- **DRA 抢占（KEP-5690，`DRAPreemption`，kube-scheduler）**：调度器可以抢占使用节点本地独占设备的低优先级 Pod（设备来自 Pod 自己的 ResourceClaim，比如由 ResourceClaimTemplate 生成），并把释放出来的设备留给抢占者，直到它被调度。“在线推理抢占离线任务的 GPU”这类场景终于有了原生的 Alpha 支持。该特性以代码冻结例外的形式于 11 月 20 日合入。
- **扩展资源容量请求（KEP-6406，`DRAExtendedResourceCapacity`，依赖 `DRAConsumableCapacity`）**：在 DeviceClass 上通过 `spec.extendedResourceName` 和 `spec.extendedResourceRequest.capacity.requests` 定义每一份的容量，比如每份 10 个 `shares`。Pod 里写 `example.com/gpu-share: 2`，就会得到一个包含两份分配、每份 10 shares 的 ResourceClaim。对还在用 extended resource 写法的存量工作负载来说，这是一条走向共享 GPU 的平滑路径。设备需要发布 `shares` 容量。
- **共享可消耗容量（KEP-5941，`DRASharedConsumableCapacity`）**：不同设备从同一个预算里扣减，比如多个虚拟网卡共享一条 100 Gb/s 的物理链路。调度器同样只负责记账。
- **资源可用性视图（KEP-5677，`DRAResourcePoolStatus`）**：通过 ResourcePoolStatusRequest 获取某一时刻的设备可用情况。v1.38 修正了统计口径，比如同时被分配和打了污点的设备只计一次；资源池未发布完整时，控制器会设置 `WaitingForPools` 条件，最多等待 15 秒后写入最终状态。

### 4. 升级注意

- 从启动参数和配置文件中删除 `DynamicResourceAllocation`。`DRAPartitionableDevices`、`DRADeviceBindingConditions` 以及设备污点相关门控已锁定，设为 `false` 会报错。
- 设备污点规则（DeviceTaintRule）：要匹配所有设备需显式设置 `spec.deviceSelector.all: true`。空 selector 仍匹配所有设备，但会产生 API Server 告警，`kubectl --warnings-as-errors` 会因此返回非零；不写 selector 则不匹配任何设备。此外，`NoExecute` 驱逐现在会认可 effect 为空的容忍。
- API Server 会拒绝一些以前被接受的对象：包含多个斜杠的 qualified name、单设备 node selector 多于一个 term、负数或非法的容量请求、超出 int64 的版本号属性。已有对象只要不改这些字段，仍然可以更新。
- 驱动开发者：`k8s.io/dynamic-resource-allocation` 的 ResourceSlice 发布 helper 会拒绝重复驱动域名的属性名和容量名（比如应写 `memory`，而不是 `<driver name>/memory`），可以用 `kubeletplugin.ValidateQualifiedNames(false)` 暂时关闭这个校验。
- admin access 请求不再占用设备容量和共享计数器，也不再阻止同一 Pod 的其他 claim 使用该设备。
- v1.38 修复了 v1.37.0 引入的回归：PreBind 失败后，已分配给 ResourceClaim 的设备可能被当作空闲，造成重复分配。

## 二、WAS：从调度单个 Pod 到调度一组 Pod

分布式训练和多机推理有几个共同诉求：一组 Pod 要么一起起来，要么都别占资源；整组放进合适的拓扑域；抢占时按组考虑。Gang Scheduling 在 v1.35 以 Alpha 引入，v1.37 中 Workload 与 PodGroup 核心 API 进入 Beta。v1.38 的重点是把积木补齐，并接入内置控制器。

| KEP | 内容 | v1.38 |
| --- | --- | --- |
| [4671](https://kep.k8s.io/4671) Gang Scheduling | Workload 与 PodGroup 核心 API（`GenericWorkload`） | Beta（有较大变更） |
| [6012](https://kep.k8s.io/6012) CompositePodGroup | 用组合的 PodGroup 描述多层结构，API 进入 `scheduling.k8s.io/v1beta1` | Beta |
| [5732](https://kep.k8s.io/5732) 拓扑感知工作负载调度 | `TopologyAwareWorkloadScheduling`，按拓扑约束放置整组 Pod | Beta |
| [5710](https://kep.k8s.io/5710) 工作负载感知抢占 | 按组评估抢占；`PodGroupPreemptionPolicy` 门控删除，字段改由 `GenericWorkload` 或 `CompositePodGroup` 控制 | Beta |
| [6089](https://kep.k8s.io/6089) 控制器接入 API | 供内置和外部控制器复用的积木类型 | Beta |
| [5547](https://kep.k8s.io/5547) Job 接入 | Job 的 `spec.scheduling`，以及按名称查找 Workload 和 PodGroup | Beta |
| [6276](https://kep.k8s.io/6276) Deployment 接入 | 通过显式字段把 Deployment 关联到 Workload | Alpha |

几个值得展开的点：

- **CompositePodGroup 进入 v1beta1**。一个工作负载里有多种角色、每种角色又是一组 Pod 时，可以用 CompositePodGroup 把这些 PodGroup 组合起来整体考虑，比如 PD 分离推理里的 prefill 组和 decode 组。
- **Job 接入的行为变化**。Job 控制器改为按名称查找 Job 对应的 Workload 和 PodGroup：
  - 设置了 `spec.scheduling` 的 Job 会得到自己的 Workload 和 PodGroup，不管它的 owner 是谁，CronJob 创建的 Job 也一样。
  - 有 controller owner、但没有设置 `spec.scheduling` 的 Job 不会自动获得。
  - 带有 `scheduling.k8s.io/workload-name` 或 `scheduling.k8s.io/group-template-name` 注解的 Job，会选用已有 Workload 中的模板，这时不能再设置 `spec.scheduling`。上层控制器可以借此先创建 Workload，再让子 Job 通过注解引用其中的模板。
- **控制器接入 API 进入 Beta**。目标是让 Job、Deployment 以及外部控制器共用同一套 Workload 和 PodGroup 积木，而不是各自实现一套 gang 语义。`v1alpha3` 的 Go 类型保留为别名；拓扑约束改用 `WorkloadTopologyConstraint`，对应的 applyconfigurations 移到 `client-go/applyconfigurations/scheduling/v1`。

**如何试用**：Beta API 默认不开启，需要在 kube-apiserver 上通过 `--runtime-config` 启用 `scheduling.k8s.io/v1beta1`，并在 kube-apiserver、kube-scheduler、kube-controller-manager 上开启对应门控（如 `GenericWorkload`）。如果你用的是自定义调度 profile，开启 `GenericWorkload` 后，kube-scheduler 启动时会为每个没有在 `PodGroupPostFilter` 启用的 PostFilter 插件记录一条错误日志，这些插件不会对 PodGroup 中的 Pod 生效。默认 profile 不受影响。

**升级注意**：v1.37 用过 Alpha API 的外部控制器，要按上面的类型变化更新代码。已经在用 Kueue、Volcano 等方案的团队不必急着替换，可以先在测试集群里验证 Workload 和 PodGroup 的语义，再看上层框架何时对接。

## 三、推理服务的资源弹性：Pod 级资源与原地调整

### 1. Pod 级资源（GA）

推理 Pod 里除了模型服务，往往还有指标采集、日志、代理或模型下载之类的 sidecar。以前要给每个容器单独估算资源，现在可以直接给整个 Pod 设一个总预算（CPU、内存、hugepages），容器在预算内共享：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: llm-server
spec:
  resources:            # Pod 级资源，v1.38 GA
    requests:
      cpu: "8"
      memory: 64Gi
    limits:
      cpu: "16"
      memory: 64Gi
  containers:
  - name: server
    image: vllm/vllm-openai:latest
  - name: metrics-exporter
    image: example.com/metrics-exporter:latest
```

GPU 仍然通过 DRA 的 ResourceClaim（或 extended resource）申请，Pod 级资源只管 CPU、内存和 hugepages。

### 2. Pod 级原地调整（GA）与调度器抢占（Beta）

- **Pod 级原地调整（KEP-5419）**：通过 `pods/resize` 子资源修改 Pod 级的 CPU 和内存，不需要重建 Pod，例如：

  ```bash
  kubectl patch pod llm-server --subresource resize \
    --patch '{"spec":{"resources":{"limits":{"cpu":"24"}}}}'
  ```

  Pod 级没有单独的 `resizePolicy`，是否重启由各容器自己的 `resizePolicy` 决定。没有设置自身 limit 的容器继承 Pod 级 limit，所以如果它的内存策略是 `RestartRequired`，调整 Pod 级内存可能导致它重启。原地调整只支持 cgroup v2，也不会改变 Pod 的 QoS 类别。
- **原地扩容的调度器抢占（KEP-5836，`InPlacePodVerticalScalingSchedulerPreemption`，Beta，默认开启）**：扩容因节点资源不足进入 `Deferred` 状态时，调度器会抢占低优先级 Pod 腾出空间。调度器判断扩容可以放下时，会给 Pod 加上 `SchedulerEvaluationCompleted` 条件（reason 为 `DeferredPodResize`）。**用自定义调度器的团队注意**：评估 Deferred 扩容时，没有实现 `IsRelevantForPodResize` 接口的自定义插件会被跳过；覆盖了默认 Bind 插件的自定义 profile，需要在 Bind 阶段启用 `DeferredPodScheduling`。

### 3. 和推理、训练相关的几个小特性

- **内存卷动态调整（KEP-6030，`InPlacePodVerticalScalingMemoryBackedVolumes`，Beta）**：原地调整时可以修改 `medium: Memory` 的 emptyDir 的 `sizeLimit`，也可以新增或删除这个字段。PyTorch DataLoader、NCCL 常用的 `/dev/shm` 通常就是这类卷。
- **静态 CPU 管理策略下的原地调整（KEP-5554，`InPlacePodVerticalScalingExclusiveCPUs`，Alpha）**：由静态 CPU 管理策略管理、独占 CPU 的 Guaranteed Pod，此前无法通过 `resize` 原地修改 CPU，v1.38 开始支持。
- **Topology Manager 新选项（KEP-6007，Alpha）**：提升 `single-numa-node` 策略下的装箱密度。GPU 节点常用这个策略让 GPU、网卡和 CPU 对齐到同一个 NUMA 节点。

**升级注意**：使用 cgroup v2 和 `LimitedSwap` 的节点上，kubelet 会依据 Pod 级内存设置计算 swap。Pod 级内存 request 等于 limit 时，整个 Pod 都不会使用 swap；开启 `LimitedSwap` 时，对 Pod 级内存 request 的原地调整会被标记为 `Infeasible`。

## 四、长任务的体面退出：驱逐、节点维护与失败原因

训练任务动辄跑几天，被驱逐、节点维护、硬件故障都是常态。v1.38 在这个方向上有几项 Alpha：

- **EvictionRequest API（KEP-4563，`EvictionRequestAPI`）**：新增声明式的 EvictionRequest API，用来协作式地终止 Pod。比如节点排空时，控制器先创建 EvictionRequest，由 Pod 配置的响应方（训练框架或 Operator）先保存 checkpoint、完成收尾；没有响应方，或响应方没有完成，再回退到普通的 API 驱逐，仍然遵守 PodDisruptionBudget。它不是 PDB 的替代品。使用时需要同时开启门控和对应的 Alpha API 组。
- **节点生命周期条件（KEP-5683，`NodeLifecycleConditions`）**：在 Node 上提供一组公认的生命周期条件，让核心组件和生态里的生命周期 Operator 读到同一个信号。v1.38 是第二个 Alpha，`kubectl drain` 成为第一个写入方，会发布 `DrainInProgress` 和 `Drained`，但不改变 drain 本身的行为。GPU 节点的维护、重启和故障处置流程可以基于这些条件协同。
- **Job 失败原因（KEP-4443）**：Job 的 `podFailurePolicy` 规则可以设置可选的失败原因，Job 失败时这个原因会写进 Job 的 condition。平台可以据此区分硬件故障、OOM 和代码错误，再分别决定告警和重试策略。

另外两个相关变化：

- **单设备故障隔离**：DRA 设备污点在 v1.37 已经 GA，可以只隔离坏掉或维护中的那张卡，不必隔离整台节点；v1.38 中相关门控锁定为开启。
- **StatefulSet `Recreate` 更新策略（KEP-3541，Beta）**：用 StatefulSet 部署推理服务时，如果新版本的 Pod 一直无法 Ready，滚动更新会卡住，修正配置后也需要手动删 Pod。`Recreate` 策略用来解决这个问题。

## 五、Agent 与沙箱：动态容器和网络隔离

Agent 场景里，一次工具调用往往需要一个短生命周期的执行环境，而冷启动一个 Pod 太慢。v1.38 有一组面向这类场景的 Alpha（更完整的展开见 [Agent 视角解读](agent.md)）：

- **动态容器（KEP-5972，`DynamicContainers`）**：通过新的 `pods/dynamic` 子资源，在一次请求里向运行中的 Pod 添加或移除主容器，也可以同时调整资源。Pod 因此可以成为一个可复用的执行容器：先调度、初始化好，再按需放入要执行的容器。KEP 给出的场景包括 Agent 的 warm pool、Ray 或 Slurm 这类框架把一个 Pod 当作资源预算再细分调度、以及原地升级 sidecar，长期目标是从请求到代码开始执行在 100ms 以内。还有一个只读的 `pods/allocated` 子资源，用来读取 kubelet 实际分配后的 Pod spec。
  - 限制：只能增删主容器；Pod 必须处于 Running 且已完成初始化；不能改变 QoS 类别；不允许特权容器和 hostPort；不支持 Windows 和开启 swap 的 Pod。
  - 权限：默认的 `admin`、`edit`、`view` 角色都不包含 `pods/dynamic` 的 update 权限，需要显式授权。
- **动态 node-local 临时卷（KEP-6395，`DynamicNodeLocalEphemeralVolumes`）**：通过同一个子资源动态添加或移除 `emptyDir`、`configMap`、`secret`、`projected`、`image` 卷。新卷需要由通过动态容器加入的新容器挂载；第一个 Alpha 还不支持修改已有容器的 `volumeMounts`。
- **网络隔离 Pod（KEP-6313，`NetworkIsolatedPods`）**：设置 `spec.podNetwork: Isolated` 后，Pod 不接入默认的集群网络，只有 loopback 接口，也没有 Pod IP，可以再通过 DRA 等方式提供非默认网络。适合不需要访问集群网络的沙箱，也适合只走 RDMA 等专用网络的 Pod。
- **从 OCI 仓库分发 seccomp profile（KEP-6061，`SecurityProfileOCI`）**：沙箱的安全配置可以像镜像一样版本化分发。

## 六、大规模 GPU 集群的控制面

GPU 集群的节点数和 Pod 数一上来，控制面的压力也会上来。v1.38 中相关的变化：

- **WatchList（KEP-3157）GA**：informer 通过流式 watch 获取初始数据，不再依赖分页 LIST，降低 kube-apiserver 的内存峰值。
- **服务端分片 List/Watch（KEP-5866，Beta）**：客户端可以只订阅自己负责的分片，由 API Server 在源头过滤，控制器可以真正水平扩展。
- **managedFields opt-out（KEP-5958，Alpha）**：客户端可以选择不接收 `metadata.managedFields`，kube-controller-manager 和 kube-scheduler 的 informer 已经接入，可以减少传输和内存开销。
- **其他 Alpha**：慢 watch 流暂停与恢复（KEP-6268）、etcd 对象压缩（KEP-6264）、kube-apiserver 加权负载均衡（KEP-6315）、控制器 Leader Election 恢复（KEP-6361）。

## 升级到 v1.38 前的 AI 集群检查清单

1. **容器运行时**：v1.38 计划要求运行时实现 CRI `RuntimeConfig` RPC，也就是 containerd 2.0+ 或 CRI-O 1.28+（对应 PR [kubernetes/kubernetes#136731](https://github.com/kubernetes/kubernetes/pull/136731) 截至 11 月 25 日仍处于开放状态，尚未合入，以 Release Notes 为准）。GPU 节点上的运行时版本往往和驱动、容器工具链一起固定，建议提前在测试节点池验证。
2. **特性门控**：删除 `DynamicResourceAllocation`；不要把已锁定的 DRA 门控设为 `false`；kube-controller-manager 上删除 `SeparateTaintEvictionController`。
3. **DRA 驱动**：检查属性名和容量名是否重复了驱动域名，列表属性对应的 CEL 选择器改用 `.includes()`，DeviceTaintRule 的 selector 写法，以及 `extraNodeAllocatableResources` 字段改名。
4. **WAS**：用过 Alpha API 的控制器要更新拓扑约束类型；`PodGroupPreemptionPolicy` 门控已删除；自定义调度 profile 检查 PostFilter 插件是否也在 `PodGroupPostFilter` 启用，以及 Bind 阶段的 `DeferredPodScheduling` 配置。
5. **自定义调度插件**：评估 Deferred 扩容时，未实现 `IsRelevantForPodResize` 的插件会被跳过；树外插件调用 `NominatedPodsForNode` 需要传入 `klog.Logger`。
6. **swap**：开启 `LimitedSwap` 的节点，检查 Pod 级内存 request 和 limit 的设置。
7. **cgroup v2**：Pod 级资源的原地调整只支持 cgroup v2，cgroup v1 支持也将在未来版本移除，还没迁移的节点池尽早规划。

## 历史文章

- [Kubernetes v1.37 DRA 更新：从平滑迁移到精细化设备管理](../v1.37/dra.md)
- [Kubernetes v1.37 Workload-Aware Scheduling](../v1.37/was.md)
- [Kubernetes v1.37 今晚发布：DRA 持续成熟，Workload-Aware Scheduling 进入 Beta](../v1.37/release.md)

## 参考

1. Kubernetes v1.38 Sneak Peek（kubernetes/website#59188）<https://github.com/kubernetes/website/pull/59188>
2. Kubernetes v1.38 DRA 更新博客草稿（kubernetes/website#59213）<https://github.com/kubernetes/website/pull/59213>
3. Enhancements · v1.38 Release Tracking <https://github.com/orgs/kubernetes/projects/269/views/1>
4. Kubernetes 1.38 Release Highlights Discussion #3121 <https://github.com/kubernetes/sig-release/discussions/3121>
5. Kubernetes v1.38 CHANGELOG <https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.38.md>
6. DRA Preemption Alpha（kubernetes/kubernetes#143456）<https://github.com/kubernetes/kubernetes/pull/143456>
7. Graduate PodLevelResources to GA（kubernetes/kubernetes#140536）<https://github.com/kubernetes/kubernetes/pull/140536>
8. Graduate InPlacePodLevelResourcesVerticalScaling to GA（kubernetes/kubernetes#143999）<https://github.com/kubernetes/kubernetes/pull/143999>
9. Scheduler preemption for IPPR: beta update（kubernetes/kubernetes#143158）<https://github.com/kubernetes/kubernetes/pull/143158>
10. Dynamic resize of memory-backed volumes（kubernetes/kubernetes#141807）<https://github.com/kubernetes/kubernetes/pull/141807>
11. [WAS] KEP-5547 Job controller（kubernetes/kubernetes#143534）<https://github.com/kubernetes/kubernetes/pull/143534>
12. [WAS] KEP-6089 Promote Building Blocks APIs（kubernetes/kubernetes#142962）<https://github.com/kubernetes/kubernetes/pull/142962>
13. Dynamic Containers（kubernetes/kubernetes#143714、#143715）<https://github.com/kubernetes/kubernetes/pull/143715>
14. Dynamic node-local ephemeral volumes（kubernetes/kubernetes#143709）<https://github.com/kubernetes/kubernetes/pull/143709>
15. kubelet: disable cgroupDriver fallback by default（kubernetes/kubernetes#136731）<https://github.com/kubernetes/kubernetes/pull/136731>
16. Kubernetes Enhancement Proposals <https://kep.k8s.io/>
