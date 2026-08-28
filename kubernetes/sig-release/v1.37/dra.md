# Kubernetes v1.37 DRA 更新：从平滑迁移到精细化设备管理

Kubernetes v1.37 已于 2026 年 8 月 26 日正式发布。DRA（Dynamic Resource Allocation，动态资源分配）核心框架从 v1.34 进入 GA 之后，v1.35、v1.36 又陆续补上设备健康、容量共享、动态分区、污点与工作负载级声明等能力。

到了 v1.37，DRA 要解决的已经不只是“能不能用 `ResourceClaim` 申请一块 GPU”，而是更贴近生产环境的问题：现有工作负载如何平滑迁移、故障设备如何隔离、不同厂商和不同类型的设备如何表达拓扑关系，以及一组 Pod 如何共同申请和使用设备。

![Kubernetes v1.37 DRA 更新](dra-update.png)

本文单独梳理 Kubernetes v1.37 中值得关注的 DRA 更新，并在文末预告我和 NVIDIA 的 Kang Zhang 在 KubeCon + CloudNativeCon China 2026 上的 DRA 分享。

## 先看结论：v1.37 的 DRA 更新了什么

v1.37 的主线可以概括为三层：GA 能力降低迁移和运维门槛，Beta 能力补齐设备使用链路，Alpha 能力开始处理更复杂的设备模型与大规模调度问题。

| 阶段 | 重点能力 | 价值 |
| --- | --- | --- |
| Stable / GA | Extended Resources via DRA、Device Taints and Tolerations、ResourceClaim Device Status、标准 `numaNode` 属性 | 兼容旧工作负载，细化故障隔离，并统一设备状态和 NUMA 拓扑表达 |
| Beta | Workload 级 ResourceClaim、Device Attributes Downward API；容量共享、动态分区、绑定条件和资源健康继续成熟 | 连接 DRA 与 Workload-Aware Scheduling，补齐设备信息注入、共享、准备和健康观测链路 |
| Alpha | 多值属性、节点资源核算、资源可用量、可选节点操作、派生属性、设备兼容组和调度队列优化 | 面向异构设备、超节点、复杂拓扑和大规模集群继续探索 |

## Extended Resource 兼容路径进入 GA

[KEP-5004](https://kep.k8s.io/5004) Extended Resources via DRA 在 v1.37 进入 GA。DRA driver 可以通过 `DeviceClass` 承接 `nvidia.com/gpu: 2`、`example.com/accelerator: 1` 这类传统 extended resource 请求。现有工作负载不需要先改写为 `ResourceClaim`，平台也不需要为了同一种设备长期同时维护 Device Plugin 和 DRA 两套分配路径。

这项能力从 v1.35 Alpha、v1.36 Beta 一路走到 v1.37 Stable。对还在使用 Device Plugin 的平台来说，它提供了一条风险更低的迁移路径：

1. 先部署和验证 DRA driver，用 `DeviceClass` 关联原有 extended resource 名称；
2. 保持应用 YAML 不变，按节点池或设备类型逐步把后端分配切换到 DRA；
3. 等 driver、监控和故障处置稳定后，再让需要高级能力的新工作负载直接使用 `ResourceClaim`。

应用团队不必和平台改造同时切换 API，平台团队也可以分阶段验证，这可能是 v1.37 DRA 最直接的生产价值。

## 设备故障隔离、状态与拓扑表达进入稳定阶段

v1.37 中几项已经成熟的能力，把 DRA 从“设备分配接口”继续推进成“设备管理接口”。

| KEP | v1.37 阶段 | 解决的问题 |
| --- | --- | --- |
| [5004](https://kep.k8s.io/5004) Extended Resources via DRA | GA | 旧 extended resource 工作负载无需修改即可由 DRA driver 分配设备 |
| [5055](https://kep.k8s.io/5055) Device Taints and Tolerations | GA | 给故障或维护中的单个设备设置 taint，阻止后续分配，而不是隔离整台节点 |
| [4817](https://kep.k8s.io/4817) ResourceClaim Device Status | GA | 由 driver 在 claim status 中报告设备状态和标准化网络接口数据，支撑 RDMA、多网卡和网络设备集成 |
| [6072](https://kep.k8s.io/6072) Standard `numaNode` Device Attribute | Stable | 统一使用 `resource.kubernetes.io/numaNode` 表达 NUMA 位置，让不同 driver 的设备可以比较拓扑关系 |

其中，`numaNode` 直接以 Stable 落地，因为它标准化的是设备属性名称，没有独立特性门控，也不改变内置分配行为。它看起来只是一个命名约定，却为 GPU、NIC、TPU 等来自不同 driver 的设备进行同 NUMA 节点放置提供了共同语言。

设备故障相关的几个概念也需要区分：

- Device Taints and Tolerations 控制故障或维护中的设备是否还能被新工作负载选中；
- [KEP-4817](https://kep.k8s.io/4817) 提供 `ResourceClaim.status.devices` 中由 driver 报告的设备状态和标准化网络接口数据；
- [KEP-4680](https://kep.k8s.io/4680) Resource Health Status 继续处于 Beta，关注 Pod 和容器看到的设备健康状态。

三者分别覆盖“还能不能分配”“设备自身报告了什么”和“工作负载看到了什么”，共同形成发现、隔离和排障链路。需要特别说明的是，KEP-4680 在 v1.37 并未进入 GA。

## Workload 级 ResourceClaim 连接 DRA 与 WAS

[KEP-5729](https://kep.k8s.io/5729) 在 v1.37 进入 Beta。启用默认关闭的 `DRAWorkloadResourceClaims` 后，Workload 或 PodGroup 可以关联 `ResourceClaim` 和 `ResourceClaimTemplate`，让设备声明跟随整组工作负载的生命周期，而不再局限于逐 Pod 创建和预留。

这对分布式训练、MPI 和大规模批处理尤其重要。一组训练进程可能需要共享 GPU 分区、RDMA 接口，或者在统一的拓扑约束下共同使用一组设备；只在单个 Pod 层描述资源，很难完整表达这种关系。Workload 级声明把 DRA 与 Workload-Aware Scheduling（WAS）连接起来，使调度器能够从整组工作负载的角度考虑设备和 Pod。

v1.37 还收紧了特性门控关闭时的行为：如果 Pod 引用的 `ResourceClaimTemplate` 与 PodGroup 中的声明匹配，但 `DRAWorkloadResourceClaims` 没有启用，系统不会退化成“为每个 Pod 各创建一份 claim”。这可以避免原本面向整组共享的设备声明被批量复制，进而耗尽 DRA 资源。

Beta 代表 API 和实现更加成熟，但并不等于升级后自动启用。`DRAWorkloadResourceClaims` 在 v1.37 仍默认关闭，需要在 kube-apiserver、kube-controller-manager、kube-scheduler 和 kubelet 上显式启用。

## Beta 能力补齐设备使用链路

除了工作负载级 `ResourceClaim`，v1.37 的 DRA 还有一组 Beta 能力覆盖设备信息注入、共享容量、动态分区、健康状态和绑定时序。其中 KEP-5304 在 v1.37 从 Alpha 进入 Beta，其余几项延续并完善此前的 Beta 实现。

| KEP | v1.37 阶段 | 解决的问题 |
| --- | --- | --- |
| [5304](https://kep.k8s.io/5304) Device Attributes Downward API | Beta | 将 driver 在 claim preparation 阶段生成的设备 metadata 通过 CDI JSON 文件注入容器，应用无需额外 controller 即可读取 PCI 地址、MAC 等设备信息；该能力没有独立特性门控 |
| [5075](https://kep.k8s.io/5075) Consumable Capacity | Beta | 允许多个独立 `ResourceClaim` 从同一设备的可消费容量中分配份额，例如共享网络带宽或虚拟 GPU 内存 |
| [4815](https://kep.k8s.io/4815) Partitionable Devices | Beta | 描述 GPU、TPU 等设备的可切分结构和多主机拓扑，让工作负载请求具体分区，而不是一组彼此无关的设备 |
| [5007](https://kep.k8s.io/5007) Device Binding Conditions | Beta | 在外部设备真正准备完成前延迟 Pod 与节点的绑定，并在准备失败或超时时重新调度 |
| [4680](https://kep.k8s.io/4680) Resource Health Status | Beta | 把 Device Plugin 和 DRA 设备健康暴露到 Pod 或容器状态，帮助定位设备故障导致的崩溃和异常 |

把这些能力放在一条链路里看，会更容易理解它们的价值：scheduler 先判断设备能否切分和共享，再等待外部设备完成准备；kubelet 和 driver 把设备信息交给容器；运行期间，平台通过 claim status、Pod status 和 device taint 观察并隔离故障。

## Alpha 探索转向复杂设备模型

v1.37 的 DRA Alpha 工作大多围绕属性、容量、兼容性和调度性能展开：

- [KEP-5491](https://kep.k8s.io/5491) List Types for Attributes 进入第二轮 Alpha，让设备属性可以保存多个值，例如一颗 CPU 同时邻接多个 PCIe root；
- [KEP-5517](https://kep.k8s.io/5517) Node Allocatable Resource Requests 进入第二轮 Alpha，尝试让 DRA 管理的 CPU、内存等节点资源参与 scheduler 与 kubelet 的常规资源核算，避免在 `ResourceClaim` 和 Pod resources 中重复申请；
- [KEP-5677](https://kep.k8s.io/5677) Resource Availability Visibility 继续处于 Alpha，目标是在 `kubectl describe resourceslice` 和 `kubectl describe node` 中展示设备池的实际剩余容量，而不只是总容量；
- [KEP-5945](https://kep.k8s.io/5945) Optional Node Preparation 允许对不需要节点本地初始化的分配跳过 kubelet prepare/unprepare 调用，减少不必要的 driver 依赖；
- [KEP-6080](https://kep.k8s.io/6080) Derived Attributes 允许使用 CEL 归一化不同 driver 的属性，例如统一拓扑标识、从复杂拓扑字符串中提取信息或生成性能分层；
- [KEP-5963](https://kep.k8s.io/5963) Device Compatibility Groups 为共享同一容量计数器的可分区设备补充兼容约束，例如在调度阶段拒绝同一 GPU 上互斥的 MIG 与 vGPU 模式，而不是等到节点 prepare 时才失败；
- [KEP-6132](https://kep.k8s.io/6132) Scheduler PreQueueingHints 为 scheduler 事件处理增加新的 Alpha extension point，使 `ResourceClaim` 变化只重新排队真正受影响的 Pod。它是 scheduler 性能能力，不是新的 DRA API；该功能原计划直接进入 Beta，因仍有 Bug 未解决而调整为 Alpha。

这些 Alpha 能力仍默认关闭，API 和行为也可能继续变化。它们更适合用于验证未来方向，不适合作为当前生产环境的兼容性承诺。

## AI 平台在 v1.37 更值得验证什么

如果要在 AI 平台上试用 DRA，v1.37 更值得验证的是完整的资源生命周期，而不只是看第一次 GPU 能不能成功分配：

- 迁移：旧 extended resource 工作负载能否在 YAML 不变的情况下切换到 DRA driver；
- 观测：`ResourceClaim`、Pod status、driver 指标和事件能否快速定位设备异常；
- 隔离：单设备故障或维护时，能否阻止新分配，而不下线整台节点；
- 拓扑：GPU、NIC、NUMA、PCIe 或 fabric 关系能否被不同 driver 一致表达并参与调度；
- 整组调度：PodGroup 共享 claim 时，设备声明、调度、绑定和回收能否跟随工作负载生命周期；
- 回退：driver 重启、`ResourceSlice` 重建、设备准备失败和节点维护时，调度能否安全重试或回退。

建议从 GA 能力和成熟 driver 开始，在专用测试集群逐步开启默认关闭的 Beta、Alpha 特性。故障演练至少覆盖设备不健康、driver 重启、`ResourceSlice` 重建、PodGroup 共享 claim、节点维护和 DRA 调度回退。

## 9 月 9 日上海 KubeCon：从架构到千卡规模实践

如果这篇文章回答的是“Kubernetes v1.37 的 DRA 更新了什么”，我和 NVIDIA 的 Kang Zhang 在 KubeCon + CloudNativeCon China 2026 上的分享，则会进一步回答“这些能力如何组合成一套面向 AI/HPC 的设备管理架构，以及在大规模环境里会遇到什么问题”。

![KubeCon + CloudNativeCon China 2026 DRA 分享](kubecon-china.png)

分享信息如下：

- 题目：**Kubernetes DRA Architecture: Scheduling, Status, and Topology at Scale**
- 时间：2026 年 9 月 9 日 16:15–16:45
- 地点：上海，Pearl Hall
- 讲者：Paco Xu（DaoCloud）与 Kang Zhang（NVIDIA）
- 语言与难度：中文，中级

我们会从 DRA 的整体架构出发，把资源入口、设备建模、scheduler 分配、binding、kubelet preparation、status、health 和访问边界串联起来；重点讨论超节点系统中的拓扑感知调度，以及 NUMA、PCIe、NVLink、fabric 等关系如何影响 AI/HPC 工作负载的性能。

分享中还会介绍一个 NVIDIA Dynamo/GB200 实践案例：IMEX daemon/channel 的分配机制如何从分布式状态冲突，演进为集中式更新，再演进为基于拓扑对齐状态对象的分布式更新，并在千卡规模下把分配延迟从分钟级降低到秒级。

欢迎 9 月 9 日来现场交流。完整议程和分享介绍可以查看[大会日程页面](https://www.lfopensource.cn/kubecon-cloudnativecon-openinfra-summit-pytorch-conference-china/program/schedule/?id=1223308)。

## 参考资料

1. [Kubernetes v1.37 正式发布](https://kubernetes.io/blog/2026/08/26/kubernetes-v1-37-release/)
2. [Kubernetes DRA 功能文档](https://kubernetes.io/docs/concepts/resource-management/dynamic-resource-allocation/dra-features/)
3. [Kubernetes v1.37 CHANGELOG](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.37.md)
4. [Kubernetes Enhancement Proposals](https://kep.k8s.io/)
