# Kubernetes v1.37 WAS 更新：Gang Scheduling 进入 Beta，调度开始理解“整组工作负载”

Kubernetes v1.37 已于 2026 年 8 月 26 日正式发布。对于 AI/ML、HPC 和大规模批处理工作负载，这个版本最值得关注的变化之一，是 Workload-Aware Scheduling（WAS，工作负载感知调度）核心能力进入 Beta。

kube-scheduler 默认以单个 Pod 为调度单位。分布式训练、MPI、大规模批处理等任务却往往需要一组 Pod 一起运行。实际集群中经常出现这样的情况：几个 Pod 已经占用了 GPU，其他成员却因为资源不足长期 Pending；整项任务既无法启动，也没有及时释放已经占用的资源。

WAS 的目标，是让 scheduler 不只看到一个个彼此独立的 Pod，而是理解它们属于同一项工作负载，需要共同排队、共同放置，必要时也要按整组进行抢占。

![Kubernetes v1.37 Workload-Aware Scheduling 更新](was-update.png)

v1.36 用 Workload、PodGroup、Gang Scheduling、拓扑感知调度和工作负载感知抢占搭起了 Alpha 框架。v1.37 继续向前推进：核心 API、Gang Scheduling 和工作负载感知抢占进入 Beta，同时开始用 `CompositePodGroup` 表达分层工作负载，并为 JobSet、LeaderWorkerSet、RayJob 等控制器提供可复用的接入积木。

## 先看结论：v1.37 的 WAS 更新了什么

| 阶段 | 重点能力 | 价值 |
| --- | --- | --- |
| Beta | Workload / PodGroup 核心 API、Gang Scheduling、工作负载感知抢占 | scheduler 可以把一组 Pod 作为统一的排队、放置和抢占单元 |
| Beta，可选 | Workload / PodGroup 级 DRA ResourceClaim | 一组 Pod 可以共享设备声明，把整组调度与 GPU、RDMA 等设备分配连接起来 |
| Alpha | `CompositePodGroup`、多层拓扑感知调度 | 表达 JobSet、LeaderWorkerSet、分离式推理等分层工作负载 |
| Alpha | WAS Controller APIs、`workloadbuilder`、Job `.spec.scheduling` | 让不同工作负载控制器复用一致的策略结构和对象生成逻辑 |

## Workload 与 PodGroup 核心 API 进入 Beta

[KEP-4671](https://kep.k8s.io/4671) 在 v1.37 把 Workload 和 PodGroup 核心 API 提升到 `scheduling.k8s.io/v1beta1`。

简单来说，两种对象的职责不同：

- Workload 描述静态模板和调度意图；
- PodGroup 表示一组 Pod 的运行时调度单元和状态；
- Pod 通过 `.spec.schedulingGroup.podGroupName` 指向所属的 PodGroup。

启用 Gang policy 后，scheduler 会先确认至少 `minCount` 个成员能够共同放下，再统一完成调度和绑定。资源不足时，整组继续等待，避免一部分 Pod 先占住 GPU、另一部分 Pod 却永远起不来的“半调度”状态。

v1.37 内部的调度模型也有明显变化：PodGroup 成为调度队列里的一等公民，成员 Pod 不再各自排队。整组共享等待、退避和调度周期，这既减少了成员之间的无效竞争，也为后续的组级公平性和更复杂策略打下基础。

另外，`minCount` 在 v1.37 变为可修改字段。弹性训练或批处理任务可以在不中断已经运行 Pod 的情况下，调整整组工作负载能够启动或继续扩展的最低成员数量。

这些 Beta 能力统一由 `GenericWorkload` 特性门控控制。v1.36 中的 `GangScheduling` 和 `WorkloadAwarePreemption` 两个旧 gate 已经合并到 `GenericWorkload`。需要注意，`GenericWorkload` 在 v1.37 仍默认关闭；进入 Beta 不代表升级集群后调度方式会自动改变。

## 抢占也要按整组工作负载计算

[KEP-5710](https://kep.k8s.io/5710) 在 v1.37 将工作负载感知抢占推进到 Beta。

传统 kube-scheduler 抢占低优先级 Pod，是为了给某一个高优先级 Pod 腾出资源。但在 WAS 场景中，调度器需要满足的是整个 PodGroup 的 `minCount`，还可能同时满足拓扑放置约束。只为其中一个成员腾出资源并没有意义，反而可能中断低优先级任务，却仍然无法让高优先级任务真正运行。

工作负载感知抢占会从整组视角计算需要释放多少资源、哪些低优先级对象应成为 victim，以及抢占之后整组工作负载是否真的能够取得进展。

v1.37 还补强了几个关键行为：

- scheduler 先模拟移除候选 victim，只执行一遍开销较大的放置算法，后续 reprieve 阶段复用结果，减少重复求解；
- 普通 Pod 抢占也会识别 PodGroup 并尊重 `disruptionMode`，避免只拆掉要求整体中断的工作负载中的一个 Pod；
- Beta API 将原来的 `PodGroup` / `Pod` disruption mode 改为更通用的 `All` / `Single`，让 PodGroup 和 CompositePodGroup 可以复用同一套语义；
- 启用 `PodGroupPreemptionPolicy` 后，可以在 PodGroup 层声明该工作负载是否允许主动抢占其他工作负载。

这里需要区分两个方向：`disruptionMode` 约束“自己成为 victim 时如何被中断”，`PodGroupPreemptionPolicy` 则控制“自己能否作为 preemptor 主动抢占别人”。

## 用 CompositePodGroup 表达分层工作负载

单层 PodGroup 可以表达“8 个 worker 一起启动”，但现代 AI 工作负载往往不是一层结构。例如：

- JobSet 中包含多个可以复制的训练 Job；
- LeaderWorkerSet 中包含 leader 与多组 worker；
- 分离式推理包含 prefill、decode 等不同角色；
- 一项任务需要跨多个机架部署，但每个子组内部又要求更严格的本地性。

[KEP-6012](https://kep.k8s.io/6012) 在 v1.37 以 Alpha 引入 `CompositePodGroup`，把 PodGroup 和 CompositePodGroup 组织成一棵树。非叶子节点由 CompositePodGroup 表示，用 `minGroupCount` 约束至少需要多少个子组；叶子 PodGroup 再用 `minCount` 约束实际 Pod 成员数量。

scheduler 从根节点向下检查整棵树。各层策略都满足时，层级中的 Pod 才会一起绑定；否则继续等待，避免出现“driver 已经运行、worker 却不够”或者“prefill 已经占用设备、decode 子组却无法启动”的半成品。

抢占时也可以根据 `disruptionMode` 选择不同语义：

- `Single`：只中断某一个子组；
- `All`：层级中任一成员需要被抢占时，按整棵子树处理。

`CompositePodGroup` 在 v1.37 仍是 Alpha。层级结构校验、调度算法性能、控制器集成和删除保护等能力还会继续演进，不应将当前 API 当作长期兼容承诺。

## 多层拓扑感知调度

与 CompositePodGroup 配套的 [KEP-5732](https://kep.k8s.io/5732) 拓扑感知工作负载调度也在 Alpha 阶段。

它允许调度器先生成满足拓扑约束的候选 placement，再在候选域中模拟整组 Pod 的放置。例如：

- 父组要求整个工作负载落在同一可用区；
- 子组要求落在该可用区内的同一机架；
- 更下层的设备请求还要满足 GPU、NIC、NUMA 或 fabric 本地性。

scheduler 按“可用区 → 机架 → 节点或设备”的层次逐步收窄候选域，比把所有约束分别摊到单个 Pod 上，更接近训练和推理系统的真实结构。

这项能力负责的是工作负载进入 scheduler 之后的放置决策，并不会自动发现网络拓扑，也不负责配额或队列准入。平台仍然需要通过节点标签、DRA driver 或其他拓扑来源提供可计算的信息。

## DRA ResourceClaim 与整组调度连接起来

[KEP-5729](https://kep.k8s.io/5729) 在 v1.37 进入 Beta。启用 `DRAWorkloadResourceClaims` 后，Workload 或 PodGroup 可以关联 `ResourceClaim` 和 `ResourceClaimTemplate`，让一组 Pod 共享设备声明，并让 claim 的创建、预留和回收跟随 PodGroup 生命周期。

这使 WAS 不只能够判断“一组 Pod 能否一起放下”，还可以把 GPU 分区、RDMA 接口或复杂拓扑设备纳入同一个调度问题。对于超过 `ResourceClaim.status.reservedFor` 旧有逐 Pod reservation 数量限制的大规模工作负载，也不再需要为每个 Pod 记录一条独立预留。

这项能力由独立 gate 控制，不能只开启 `GenericWorkload`。DRA 在 v1.37 的其他更新，可以参考独立文章：[Kubernetes v1.37 DRA 更新：从平滑迁移到精细化设备管理](dra.md)。

## 控制器接入：复用同一套调度积木

[KEP-6089](https://kep.k8s.io/6089) 在 v1.37 以 Alpha 提供可复用的 WAS Controller APIs，包括 basic/gang policy、拓扑约束、`Single`/`All` disruption mode 和工作负载级 ResourceClaim 等结构。

单层 PodGroup 使用 `WorkloadPodGroup*` 类型，分层工作负载使用 `WorkloadCompositePodGroup*` 类型。JobSet、LeaderWorkerSet、RayJob 等控制器可以把这些 building blocks 嵌入自己的 API，复用一致的字段结构和语义，而不需要各自重新设计一套“gang”“拓扑”和“中断策略”。

配套的 [`workloadbuilder`](https://github.com/kubernetes/kubernetes/tree/release-1.37/staging/src/k8s.io/component-helpers/scheduling/schedulingv1/workloadbuilder) Go 库负责：

- 合并默认值和用户配置；
- 按 allow-list 拒绝控制器尚未支持的 policy 或 disruption mode；
- 构造 Workload、PodGroup 或 CompositePodGroup 对象。

building blocks 和 `workloadbuilder` 本身没有独立特性门控，对象的生命周期仍由接入控制器负责。对分层控制器，建议由最上层的根控制器作为整棵工作负载树的唯一“编译器”，只生成一份 Workload；子控制器可以创建运行时 PodGroup，但不应重复生成 Workload。

KEP-6089 提供的是接入规范、API 积木和共享库，并不代表 JobSet、LeaderWorkerSet、RayJob 等 out-of-tree 控制器在 v1.37 已经自动完成集成。是否支持，需要分别查看对应项目版本。

## 原生 Job 首次使用新的调度 builder

原生 Job 控制器是首批 WAS Controller APIs 使用者。[KEP-5547](https://kep.k8s.io/5547) 在 v1.37 继续处于 Alpha，给 Job 增加实验性的 `.spec.scheduling`，可以显式选择 Gang Scheduling、拓扑约束和 disruption mode。

启用 `WorkloadWithJob` 后，即使 Job 没有填写 `.spec.scheduling`，controller 也会创建 Basic Workload 和 PodGroup，并给 Pod 写入 `.spec.schedulingGroup.podGroupName`。Basic policy 保留现有逐 Pod 调度行为，不会强制 `minCount`；显式选择 gang 但没有填写 `minCount` 时，Job controller 默认使用 `parallelism`。

下面的精简示例要求 4 个 Pod 以 gang 方式调度、落入同一个可用区，并在抢占时作为整体处理：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: distributed-training
spec:
  parallelism: 4
  completions: 4
  scheduling:
    schedulingPolicy:
      gang: {}
    schedulingConstraints:
      topology:
        - key: topology.kubernetes.io/zone
    disruptionMode:
      all: {}
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: example.com/training-worker:v1
```

当前 Alpha API 中，一个 group 最多配置一个 topology constraint，Job 最多配置 4 个共享 ResourceClaim。除了 `schedulingPolicy.gang.minCount` 可以调整，`.spec.scheduling` 是否存在、policy 类型、拓扑、disruption mode 和 ResourceClaim 列表在创建后都不可变。使用工作负载级共享 ResourceClaim 还需要额外启用 `DRAWorkloadResourceClaims`。

`WorkloadWithJob` 只控制 Job API 与 Job controller 的集成，不控制 KEP-6089 的 building blocks 或 `workloadbuilder`。Workload 核心 API 进入 Beta，也不代表 Job 集成和 Controller APIs 已经进入 Beta。

## 从 v1.36 升级：API 和特性门控都要迁移

如果曾在 Kubernetes v1.36 中试用 WAS，升级到 v1.37 前需要主动处理 Alpha API 和特性门控变化：

- `scheduling.k8s.io/v1alpha2` 已移除，升级前必须删除 API Server 中该版本的 Workload 和 PodGroup 对象；
- 核心 Workload 和 PodGroup API 应迁移到 `scheduling.k8s.io/v1beta1`；
- `scheduling.k8s.io/v1alpha3` 用于仍在试验阶段的扩展，包括 CompositePodGroup 等能力；
- `GangScheduling` 和 `WorkloadAwarePreemption` 两个旧 gate 已合并到 `GenericWorkload`，升级时应删除旧配置并显式启用新 gate；
- v1.37 的 `All` / `Single` disruption mode 与 v1.36 Alpha 结构不同，不能依赖自动转换；
- 如果需要回退到 v1.36，要提前处理 v1.37 创建的对象并恢复旧特性门控配置。

这也是 Alpha API 不承诺跨版本兼容的典型例子。对象清理、API 迁移、特性门控变更和回退步骤都应该在升级演练中验证。

## 如何启用和验证

WAS 在 v1.37 同时包含 Beta 核心和 Alpha 扩展，只启用一个 gate 不会自动获得全部能力：

| 特性门控 | 阶段 / 默认值 | 需要启用的组件 | 能力 |
| --- | --- | --- | --- |
| `GenericWorkload` | Beta / 默认关闭 | kube-apiserver、kube-controller-manager、kube-scheduler | Workload、PodGroup、Gang Scheduling 与工作负载感知抢占 |
| `PodGroupPreemptionPolicy` | Alpha / 默认关闭 | kube-apiserver、kube-scheduler | 在 PodGroup 层声明是否允许主动抢占其他工作负载 |
| `DRAWorkloadResourceClaims` | Beta / 默认关闭 | kube-apiserver、kube-controller-manager、kube-scheduler、kubelet | Workload / PodGroup 共享 ResourceClaim |
| `TopologyAwareWorkloadScheduling` | Alpha / 默认关闭 | kube-apiserver、kube-scheduler | 单层和多层拓扑感知放置 |
| `CompositePodGroup` | Alpha / 默认关闭 | kube-apiserver、kube-controller-manager、kube-scheduler | 分层工作负载与组级策略 |
| `WorkloadWithJob` | Alpha / 默认关闭 | kube-apiserver、kube-controller-manager | Job `.spec.scheduling` 集成 |

建议先在专用 AI 或批处理测试集群启用 `GenericWorkload`，验证最小的单层 PodGroup 和 Gang Scheduling，再逐步增加抢占、拓扑、DRA 和分层结构。重点观察：

- PodGroup 在队列中的等待和退避时间；
- `minCount` 满足率与整组调度成功率；
- 抢占后高、低优先级工作负载的完成时间；
- topology placement 的求解耗时和 scheduler 吞吐量；
- PodGroup、Workload 与 ResourceClaim 的创建、更新和回收；
- scheduler 或 controller 重启后的状态恢复与重复对象处理。

## WAS 不替代队列准入和配额系统

WAS 解决的是工作负载进入 kube-scheduler 之后，如何以整组视角进行排队、放置、绑定和抢占。它不负责工作负载级配额、跨租户公平性，也不替代外部队列准入系统。

如果集群已经使用 Kueue、Volcano 或自研调度系统，需要先明确职责边界：

- 谁负责工作负载准入与排队；
- 谁负责租户配额和公平共享；
- 谁决定 Gang Scheduling 的最小规模；
- 谁负责节点、机架和设备拓扑放置；
- 谁执行抢占，抢占的对象和策略是什么。

避免两套系统同时控制同一层决策，比单纯开启更多特性门控更重要。v1.37 的价值，是让原生 kube-scheduler 第一次拥有更完整的工作负载级调度语义，也为上层控制器和队列系统提供一套可以共同复用的底层模型。

## 参考资料

1. [Kubernetes v1.37 正式发布](https://kubernetes.io/blog/2026/08/26/kubernetes-v1-37-release/)
2. [KEP-4671：Gang Scheduling using Workload Object](https://kep.k8s.io/4671)
3. [KEP-5710：Workload-Aware Preemption](https://kep.k8s.io/5710)
4. [KEP-6012：CompositePodGroup API](https://kep.k8s.io/6012)
5. [KEP-5732：Topology-Aware Workload Scheduling](https://kep.k8s.io/5732)
6. [KEP-6089：Workload-Aware Scheduling Controller APIs](https://kep.k8s.io/6089)
7. [Kubernetes v1.36：Advancing Workload-Aware Scheduling](https://kubernetes.io/blog/2026/05/13/kubernetes-v1-36-advancing-workload-aware-scheduling/)
