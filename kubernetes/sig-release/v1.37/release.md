# Kubernetes v1.37 前瞻：DRA 继续成熟，Workload-Aware Scheduling 进入 Beta

Kubernetes v1.37 计划于 2026 年 8 月 26 日（周三）发布。截至 2026 年 8 月 7 日，`release-1.37` 分支和 v1.37.0-rc.0 已经创建，正式 release notes 与发布博客仍在收尾。

本文结合 Kubernetes v1.36 及更早版本发布文章的结构，基于 v1.37 Sneak Peek、SIG Release Highlights 讨论、`release-1.37` 分支 feature gates 和 release notes draft 整理。由于当前仍是 RC 阶段，特性数量、阶段、发布主题和已知问题都可能在正式发布前变化；最终请以 v1.37.0 CHANGELOG 与正式 release notes 为准。

如果用一句话概括 v1.37：Kubernetes 正在把过去几个版本铺开的能力变成更完整的生产路径——Workload Aware Scheduling（WAS）进入 Beta，DRA 多项设备能力进入 GA，节点侧的 Memory QoS、Rootless Kubelet 和 Pod 级资源管理继续成熟，API Server 与 etcd 则进一步优化大规模集群的启动、Watch 和恢复效率。

## 发布状态、主题和 Logo

v1.37.0-rc.0 已于 2026 年 8 月 5 日发布，正式版本计划于 8 月 26 日发布。

截至本文整理时间，v1.37 的正式主题、Logo 和发布统计尚未公布。按照 Kubernetes 社区发布流程，这些内容会随正式发布博客公开，本文暂不提前补图或猜测主题。

正式发布前需要回填：

- v1.37 主题、Logo、设计者和主题故事；
- Stable、Beta、Alpha、Deprecated 的最终数量；
- v1.37.0 已知问题与最后一轮 release notes；
- DaoCloud 和国内社区在本周期的贡献与活动更新。

## 先看升级风险

与特性列表相比，下面几项更值得集群管理员先处理。

### SELinux 卷挂载行为变化

`SELinuxMount` 在 v1.37 进入 GA 并默认启用。对于 `.spec.seLinuxMount: true` 的 CSI Driver，kubelet 会优先使用 `-o context=<label>` 挂载卷，而不是递归修改卷内文件标签。这能显著减少大卷挂载时的递归 relabel 开销，但也改变了共享卷的兼容边界。

同一节点上，如果多个 Pod 使用不同 SELinux 标签共享同一个卷，过去递归 relabel 下可能可以共存，v1.37 中则可能因为一个挂载只能使用一个 SELinux context 而启动失败。需要保留旧行为的工作负载，可在 Pod 中显式设置 `seLinuxChangePolicy: Recursive`。

未启用 SELinux 的集群不受影响。启用了 SELinux 的集群应在 v1.36 上先启用可选的 `selinux-warning-controller`，再检查 `selinux_warning_controller_selinux_volume_conflict`、`volume_manager_selinux_volume_context_mismatch_warnings_total` 指标和相关事件，然后安排升级。

### WAS Alpha API 和 feature gate 迁移

Workload Aware Scheduling 在 v1.37 进入 Beta，但 Alpha 使用者需要执行迁移动作：

- `scheduling.k8s.io/v1alpha2` 被移除；升级前必须删除对应对象；
- 核心 Beta API 进入 `v1beta1`，仍处于试验的扩展能力使用新的 `v1alpha3`；
- `GangScheduling` 和 `WorkloadAwarePreemption` feature gate 合并到 `GenericWorkload`；升级配置中应移除旧 gate；
- 如果需要回退到 v1.36，则要重新配置旧 gate，并注意 `v1alpha3` 无法向 `v1alpha2` 做兼容转换。

这也是 Alpha API 不承诺跨版本兼容的典型案例。已经试用 v1.36 WAS 的集群，应把对象清理和 feature gate 迁移纳入升级闸门。

### kubelet 行为和监控变化

- `eventRecordQPS: 0` 现在严格表示“不限流”。如果现有环境依赖此前的实际限流行为，应显式设置非零值，例如 `50`。
- kubelet 启动时会记录生效配置。由于日志可能暴露配置细节，应把 `nodes/logs` ClusterRole 仅授予可信管理员和运维组件。
- kubelet 内嵌 cAdvisor 切换到更轻量的库模块，一批长期弃用的 cAdvisor flags 会导致 kubelet 启动失败；`userDefinedMetrics`、`container_application_*` 以及少量旧 `/metrics/cadvisor` 指标也不再提供。升级前应扫描 kubelet 参数、Prometheus rules 和 dashboard。
- Static Pod 不能再引用 Secret、ConfigMap 等 API 资源，且 `PreventStaticPodAPIReferences` gate 已移除，无法恢复旧行为。

### kubeadm 配置 API

已经从 v1.31 开始弃用的 kubeadm `v1beta3` 配置 API 在 v1.37 被移除。仍保存 `v1beta3` 配置的集群，应在升级前使用兼容版本的 `kubeadm config migrate` 转换到 `v1beta4`。

v1.37 中出现的 kubeadm `v1` 只是实验占位，当前不能作为正式配置 API 使用。

### kube-proxy：IPVS 进入明确退出周期

v1.37 会对 kube-proxy 的 IPVS 模式输出弃用告警。社区当前计划在 v1.40 默认禁用 IPVS，并在 v1.43 完全移除。对于较新的 Linux 内核，建议开始验证 nftables；不满足 nftables 条件的环境仍可使用 iptables。

同时，未显式设置 kube-proxy mode 的配置会收到告警。v1.37 中 kubeadm 仍会把空值明确写为 `iptables`，但这是为未来把默认后端切换到 nftables 做准备。平台团队应避免继续依赖隐式默认值。

### cgroup v1 仍可临时绕过，但不再是长期方案

v1.37 对 cgroup v1 没有新增移除动作，但自 v1.35 起 `failCgroupV1` 已默认设为 `true`。仍使用 cgroup v1 的节点必须显式设置 `failCgroupV1: false` 才能启动 kubelet。

In-Place Pod Resize、Memory QoS 等新能力依赖 cgroup v2，cgroup v1 代码也不再作为主要测试路径。继续使用 override 只适合作为短期过渡，应尽快完成操作系统、容器运行时和节点池迁移。

## 专题一：DRA 从“能分配设备”走向平滑迁移和精细管理

DRA 核心框架已在 v1.34 GA，v1.35 和 v1.36 又陆续补齐设备健康、容量、分区、污点和工作负载级声明。到了 v1.37，主线不再只是“用 ResourceClaim 申请一块 GPU”，而是同时解决三个更接近生产的问题：如何让旧工作负载无感迁移到 DRA、如何把设备故障与状态纳入运维闭环，以及如何描述跨厂商、可切分、具备复杂拓扑关系的设备。

### Extended Resource 兼容路径进入 GA

[KEP-5004](https://kep.k8s.io/5004) Extended Resources via DRA 在 v1.37 进入 GA。DRA driver 可以通过 DeviceClass 承接 `nvidia.com/gpu: 2`、`example.com/accelerator: 1` 这类传统 extended resource 请求；工作负载不需要先改写成 ResourceClaim，也不需要为同一种设备同时部署 Device Plugin 和 DRA 两套分配路径。

这项能力从 v1.35 Alpha、v1.36 Beta 走到 v1.37 Stable，真正价值并不是新增一种请求语法，而是提供迁移层：应用清单保持不变，平台可以逐个节点池或逐类设备把后端分配逻辑迁移到 DRA。等 driver、监控和故障处置链路稳定后，再让需要高级能力的新工作负载直接使用 ResourceClaim。

### 设备状态、故障隔离与拓扑表达继续稳定

v1.37 中几项已经成熟的能力把 DRA 从“分配接口”推进成“设备管理接口”：

| KEP | v1.37 阶段 | 解决的问题 |
| --- | --- | --- |
| [5004](https://kep.k8s.io/5004) Extended Resources via DRA | GA | 旧 extended resource 工作负载无需修改即可由 DRA driver 分配设备 |
| [5055](https://kep.k8s.io/5055) Device Taints and Tolerations | GA | 对单个故障或维护中的设备设置 taint，避免继续分配，而不是隔离整台节点 |
| [4817](https://kep.k8s.io/4817) ResourceClaim Device Status | GA | 由 driver 在 claim status 中报告设备状态和标准化网络接口数据，支撑 RDMA、多网卡与网络设备集成 |
| [6072](https://kep.k8s.io/6072) Standard `numaNode` Device Attribute | Stable | 统一使用 `resource.kubernetes.io/numaNode` 表达 NUMA 位置，让不同 driver 的设备能够比较拓扑关系 |

其中 `numaNode` 直接以 Stable 落地，因为它标准化的是设备属性名称，没有独立 feature gate，也不改变内置分配行为。它看似只是命名约定，却为 GPU、NIC、TPU 等来自不同 driver 的设备做同 NUMA 节点放置提供了共同语言。

### Workload 级 ResourceClaim 连接 DRA 与 WAS

[KEP-5729](https://kep.k8s.io/5729) 在 v1.37 进入 Beta。开启默认关闭的 `DRAWorkloadResourceClaims` 后，Workload 或 PodGroup 可以直接引用 ResourceClaim，让一份 claim 按工作负载生命周期创建并由整组 Pod 共享，不再受旧的逐 Pod reservation 数量限制。这对于共享 GPU 分区、RDMA 接口或一组训练进程共同使用的设备尤其重要。

v1.37 还收紧了关闭 gate 时的行为：如果 Pod 引用的 ResourceClaimTemplate 与 PodGroup 中的声明匹配，但 `DRAWorkloadResourceClaims` 没有启用，系统不会退化成“为每个 Pod 各建一份 claim”。这避免了原本面向整组共享的设备声明被批量复制，进而耗尽 DRA 资源。

### Alpha 探索转向复杂设备模型

v1.37 的 DRA Alpha 工作大多围绕“属性、容量和兼容性”展开：

- [KEP-5491](https://kep.k8s.io/5491) List Types for Attributes 进入第二轮 Alpha，让设备属性可以保存多个值，例如一颗 CPU 同时邻接多个 PCIe root；
- [KEP-5517](https://kep.k8s.io/5517) Node Allocatable Resource Requests 进入第二轮 Alpha，尝试让 DRA 管理的 CPU、内存等节点资源参与 scheduler 与 kubelet 的常规资源核算，避免在 ResourceClaim 和 Pod resources 中重复申请；
- [KEP-5677](https://kep.k8s.io/5677) Resource Availability Visibility 继续 Alpha，目标是在 `kubectl describe resourceslice` 和 `kubectl describe node` 中展示设备池的实际剩余容量，而不只是总容量；
- [KEP-5945](https://kep.k8s.io/5945) Optional Node Preparation 允许对无需节点本地初始化的分配跳过 kubelet prepare/unprepare 调用，减少不必要的 driver 依赖；
- [KEP-6080](https://kep.k8s.io/6080) Derived Attributes 允许用 CEL 归一化不同 driver 的属性，既能把 `numa` 与 `numaNode` 之类的命名差异映射起来，也能从复杂拓扑字符串中提取标识或生成性能分层；
- [KEP-5963](https://kep.k8s.io/5963) Device Compatibility Groups 为共享同一容量计数器的可分区设备补充兼容约束，让 scheduler 在调度阶段拒绝互斥的设备模式，而不是等到节点 prepare 时才失败。

这些 Alpha 能力仍默认关闭，也可能继续调整 API。对 AI 平台来说，v1.37 更适合按“兼容迁移—状态观测—故障隔离—复杂拓扑”四条路径验证 DRA，而不是只测试首次分配 GPU 是否成功。故障演练至少应覆盖：设备变为不健康、driver 重启、ResourceSlice 重建、PodGroup 共享 claim、节点维护和 DRA 调度回退。

## 专题二：Workload-Aware Scheduling 从 Pod 调度走向工作负载调度

传统 kube-scheduler 逐个处理 Pod。对于分布式训练、MPI 和大规模批处理，常见问题是部分 Pod 已经占住 GPU、CPU 和内存，其余成员却长期 Pending，整项任务既无法开始，也无法释放资源。v1.36 为 Workload、PodGroup、Gang Scheduling、拓扑感知调度和工作负载感知抢占搭起 Alpha 框架；v1.37 则把核心 API、Gang Scheduling 和 Workload-aware Preemption 一起推进到 Beta。

### Workload 与 PodGroup 核心 API 进入 Beta

[KEP-4671](https://kep.k8s.io/4671) 将 Workload 与 PodGroup 核心 API 提升到 `scheduling.k8s.io/v1beta1`。Workload 描述相对静态的模板和调度意图，PodGroup 表示一组 Pod 的运行时状态；对于 Gang policy，scheduler 要先确认至少 `minCount` 个成员能够形成可行放置，再统一进入调度和绑定流程。

核心能力使用 `v1beta1`，CompositePodGroup 等仍处于试验阶段的扩展则使用 `scheduling.k8s.io/v1alpha3`；旧的 `v1alpha2` 已被移除。已经在 v1.36 试用 WAS 的集群不能只修改 feature gate，还必须先完成对象和 API 版本迁移。

v1.37 不只是更改 API 版本，还调整了内部调度模型：PodGroup 成为调度队列中的一等对象，成员 Pod 不再各自独立排队。这样整组 Pod 共享队列等待、退避与调度周期，也为后续更复杂的组级排队策略打下基础。与此同时，过去不可变的 `minCount` 现在允许更新，控制器可以在不打断已运行 Pod 的前提下缩小或扩大弹性 gang 的最小规模。

核心能力统一由 Beta gate `GenericWorkload` 控制，原来的 `GangScheduling` 和 `WorkloadAwarePreemption` gate 已被合并。需要注意，`GenericWorkload` 在 v1.37 仍默认关闭；Beta 表示 API 和实现更加成熟，不等于升级后自动改变所有集群的调度方式。

### 抢占开始理解“整组是否放得下、整组是否能拆”

[KEP-5710](https://kep.k8s.io/5710) Workload-aware Preemption 同样进入 Beta。普通 Pod 抢占只回答“为一个高优先级 Pod 腾出哪些资源”，而工作负载感知抢占必须回答“清理哪些低优先级工作负载后，整个高优先级 PodGroup 才能满足 `minCount` 和放置约束”。

v1.37 对这条路径做了三项关键完善：

- scheduler 先模拟移除候选 victims，并只运行一次高成本的工作负载放置算法；后续 reprieve 阶段复用该结果判断哪些 victim 可以留下，降低多次重复求解的开销；
- 默认 Pod 抢占也会识别 PodGroup，并尊重其 `disruptionMode`，避免一个要求整组中断的工作负载只被拆掉单个 Pod；
- Beta API 将原来的 `PodGroup` / `Pod` disruption mode 重命名为更通用的 `All` / `Single`，为 PodGroup 与 CompositePodGroup 共享语义做准备。启用 `PodGroupPreemptionPolicy` 后，还可以在 PodGroup 层明确声明是否允许它主动抢占其他工作负载。

### CompositePodGroup 表达分层 AI 工作负载

单层 PodGroup 适合“8 个 worker 一起启动”，却难以表达 JobSet、LeaderWorkerSet 或分离式推理中的多层关系。[KEP-6012](https://kep.k8s.io/6012) 在 v1.37 以 Alpha 引入 `CompositePodGroup`：它把 PodGroup 和其他 CompositePodGroup 组织成树，每个父组可以用 `minGroupCount` 约束至少需要多少个子组，每个叶子 PodGroup 再用 `minCount` 约束成员 Pod 数量。

scheduler 会从根节点递归检查整棵树。只有父子各层的策略都满足时，整棵层级中的 Pod 才会原子绑定；否则全部保持未调度，避免出现“driver 已运行但 worker 不足”或“prefill 已占设备但 decode 子组无法启动”的半成品部署。抢占时也可以选择 `Single`，允许独立中断某个子组；或者选择 `All`，一旦层级中任一成员需要被抢占，就按整个子树处理。

与它配套的 [KEP-5732](https://kep.k8s.io/5732) 多层拓扑感知调度也处于 Alpha。父组可以先要求整个工作负载落在同一个可用区，子组再要求各自落在该可用区内的同一机架。scheduler 按“可用区 → 机架”的顺序自上而下收窄候选拓扑域，比把所有约束平铺在单个 Pod 上更贴近大规模训练和推理系统的实际结构。

### 控制器接入不再各自重复造轮子

[KEP-6089](https://kep.k8s.io/6089) Controller Integration APIs 为上层控制器提供统一的调度积木，包括 gang/basic policy、拓扑约束、`Single`/`All` disruption mode 和工作负载级 ResourceClaim。控制器仍可以按自己的领域模型命名字段，再通过 [`workloadbuilder`](https://github.com/kubernetes/component-helpers/tree/master/scheduling/schedulingv1/workloadbuilder) 库校验并生成 Workload、PodGroup 或 CompositePodGroup 对象。

原生 Job controller 是第一批采用者。[KEP-5547](https://kep.k8s.io/5547) 为 Job 增加实验性的 `.spec.scheduling`，用户可以显式选择 gang scheduling、拓扑、disruption mode 和整组共享的 ResourceClaim；不填写时仍保持现有逐 Pod 调度行为。除 gang 的 `minCount` 可调整外，该配置创建后保持不可变。该集成仍为 Alpha，需要开启 `WorkloadWithJob`，不能因为 Workload API 进入 Beta 就把 Job 集成也视为 Beta。

### 如何启用和验证

WAS 在 v1.37 同时包含 Beta 核心和 Alpha 扩展，不能只开启一个 gate 就默认获得全部能力：

| feature gate | 阶段 / 默认值 | 需要启用的组件 | 能力 |
| --- | --- | --- | --- |
| `GenericWorkload` | Beta / 默认关闭 | kube-apiserver、kube-controller-manager、kube-scheduler | Workload、PodGroup、Gang Scheduling 与工作负载感知抢占 |
| `DRAWorkloadResourceClaims` | Beta / 默认关闭 | kube-apiserver、kube-controller-manager、kube-scheduler、kubelet | Workload / PodGroup 共享 ResourceClaim |
| `TopologyAwareWorkloadScheduling` | Alpha / 默认关闭 | kube-apiserver、kube-scheduler | 单层和多层拓扑感知放置 |
| `CompositePodGroup` | Alpha / 默认关闭 | kube-apiserver、kube-controller-manager、kube-scheduler | 分层工作负载与组级策略 |
| `WorkloadWithJob` | Alpha / 默认关闭 | kube-apiserver、kube-controller-manager | Job `.spec.scheduling` 集成 |

建议先在专用 AI 或批处理集群验证队列等待时间、PodGroup 调度成功率、抢占后的任务完成时间、拓扑求解开销和 ResourceClaim 生命周期。已经使用 Kueue、Volcano 或自研调度器的平台，还需要先划清准入排队、配额、Gang Scheduling 与节点放置分别由谁负责，避免两个系统同时管理同一层决策。

## GA 和稳定的功能

### Metrics API 终于进入 v1（KEP-5207）

`metrics.k8s.io` 从 v1.8 起长期处于 `v1beta1`，在 v1.37 进入 Stable 并提供 `v1` API。HPA 与 `kubectl top` 都依赖这套标准的 Pod/Node CPU、内存指标接口。

这次升级不改变 API 结构，`v1` 与 `v1beta1` 会在迁移期并存，因此现有客户端不需要在升级当天强制切换。它更像是对多年生产使用事实的正式确认，但 API client 和 metrics-server 仍应逐步切换到 `v1`。

### Pod Certificates 与 ClusterTrustBundles 进入 GA（KEP-4317、KEP-3257）

Pod Certificates 允许工作负载通过 projected volume 请求和轮换 X.509 证书，签发器则通过 `PodCertificateRequest` 完成签名。ClusterTrustBundle 提供集群级信任锚分发机制，让 Pod 按 signer 选择并挂载 CA bundle。

两者组合后，平台无需在每个 namespace 复制 CA ConfigMap，也不必把长期静态证书烘进镜像。它们为 Pod mTLS、SPIFFE 风格身份和企业内部 PKI 集成提供了更标准的 Kubernetes API 路径。

进入 GA 不代表集群会自动拥有签发系统。平台仍需部署 signer/controller，设计 signerName、RBAC、证书生命周期、吊销和审计策略。

### HPA 可配置容差进入 GA（KEP-4951）

HPA 可以分别配置 scale-up 和 scale-down tolerance，不再只能依赖 controller-manager 的全局容差。对于波动较大的在线业务，可以提高缩容容差来减少抖动；对于突发流量敏感业务，则可以降低扩容容差来更快响应。

建议结合指标采样周期、stabilization window 和业务冷启动时间一起调优，避免只调 tolerance 造成频繁扩缩容。

### Storage Version Migrator 内置化进入 GA（KEP-4192）

Storage Version Migration 成为内置控制面能力，`storagemigration.k8s.io/v1` 默认可用。它可以把 etcd 中仍以旧 storage version 保存的对象重写到当前版本，对 API 版本演进和静态数据加密密钥轮换都很重要，也减少了部署外部 migrator 的运维成本。

v1.37 还为迁移状态增加进度信息。大型集群应关注迁移对 API Server 与 etcd 的写入压力，并分批验证 CRD、聚合 API 和密钥轮换流程。

### Resilient Watch Cache Initialization 进入 GA（KEP-4568）

Watch cache 初始化不再在 API Server 启动或恢复时向 etcd 形成明显的惊群压力，请求也能在 cache 预热期间更平稳地处理。它与 v1.37 的 Concurrent Watch Object Decode、Etcd RangeStream 一起构成控制面启动和恢复优化主线。

### Node Declared Features 进入 GA（KEP-5328）

节点可以在 Node status 中声明实际支持的能力，调度器据此过滤不具备所需功能的节点。这解决了 feature gate 已在控制面开启，但混合版本节点、运行时或操作系统实际能力不同的问题，为更快、更安全地推广节点特性提供基础。

### KYAML 输出进入 Stable（KEP-5295）

支持 `--output` 的 kubectl 命令可以使用 `-o kyaml`。KYAML 通过更明确的字符串和数字表示减少 YAML 1.1 中常见的隐式类型陷阱，例如把 `NO` 解析成布尔值的“Norway problem”。

它是新的输出格式，不会替换现有 `-o yaml`。对配置生成、代码评审和 GitOps 流程来说，更适合先比较 diff 和下游解析器兼容性，再决定是否作为默认导出格式。

## 进入 Beta 阶段的功能

### Kubelet in User Namespace / Rootless Mode（KEP-2033）

该能力允许 kubelet 和同一节点命名空间内的 CRI、OCI、CNI 等组件在 Linux user namespace 中以宿主机非 root 用户运行，同时在 namespace 内保留所需的 root 语义。这样可以缩小节点组件漏洞影响宿主机的范围。

v1.37 中相关 gate 进入 Beta 并默认启用，但这只表示支持能力可用，不会自动把现有节点改造成 rootless。节点 user namespace、运行时、网络、挂载、设备访问和 systemd 服务仍需要按发行版逐项配置与验证。

### Memory QoS（KEP-2570）

Memory QoS 在经历多轮 Alpha 设计和回滚安全改进后进入 Beta，基于 cgroup v2 的 `memory.min`、`memory.low` 和 `memory.high` 提供分层内存保护与节流。

v1.37 默认启用该能力的支持，但关键保护和节流行为仍由 kubelet 配置控制。建议使用 Linux 5.9+ 内核和支持 cgroup v2 的运行时，在内存压力测试中关注延迟、reclaim、OOM、`memory.events` 以及关闭 feature gate 后旧 cgroup 值能否正确清理。

### HPA Scale to Zero（KEP-2021）

HPA 从 Alpha 进入 Beta，可以在 Object 或 External metrics 场景将工作负载从 0 扩到非 0，再缩回 0。它适合队列积压、事件数量等即使没有运行中 Pod 也能观测的指标，不适用于需要从 Pod 采集的 CPU/内存指标。

生产评估应重点关注冷启动时间、指标断流、最大扩容速度和“0 副本时监控链路是否仍然存在”。

### Pod 级资源管理继续成熟（KEP-2837、KEP-5526）

`PodLevelResources` 在 v1.37 release 分支中仍保持 Beta；v1.37 新增了默认值和 QoS 计算修复。基于它的 Pod Level Resource Managers 从 Alpha 进入 Beta，让 Topology、CPU 和 Memory Manager 可以围绕 `pod.spec.resources` 做 NUMA 对齐，并在 Pod 内划分容器独占资源和共享池。

需要注意，`PodLevelResourceManagers` 在 v1.37 仍默认关闭，并依赖 `PodLevelResources`。HPC、AI/ML 和 NFV 场景可以试点，普通工作负载不需要为升级主动启用。

### CRI Stats（KEP-2371）

kubelet 从 CRI 获取 Pod 和容器统计信息的能力进入 Beta，继续减少对内嵌 cAdvisor 采集路径的依赖。运行时必须正确实现对应 CRI stats 接口，平台也需要比较切换前后的指标完整性、标签、采样延迟和资源开销。

### 存储侧 Beta 更新

- [KEP-4049](https://kep.k8s.io/4049) Storage Capacity Scoring：调度器在动态制备卷时可按可用存储容量为节点打分；
- [KEP-5030](https://kep.k8s.io/5030) CSI Volume Attach Limits 与 Cluster Autoscaler 集成：扩容决策能考虑节点可附加卷数量，减少扩容后仍无法调度；
- [KEP-5541](https://kep.k8s.io/5541) PVC Unused Since Time：PVC status 增加 `Unused` condition，帮助发现长期未使用的卷，但不能直接替代业务确认和回收策略。

### API Server、控制器与可观测性 Beta 更新

- Concurrent Watch Object Decode（KEP-6178）默认启用，通过有界 worker pool 并行解码和转换 etcd watch event，同时保持事件顺序；使用 CRD conversion webhook 的集群要关注初始化期间并发调用上升；
- Etcd RangeStream（KEP-5966）使用单个流式 RPC 初始化 watch cache，减少分页 Range 请求和内存峰值，要求 etcd 3.7+；
- Stale Controller Mitigation（KEP-5647）继续提供 controller cache 陈旧度指标与 read-your-own-writes 保护；
- Manifest-Based Admission Control Config（KEP-5793）允许 API Server 从本地 manifest 加载 webhook 和 admission policy，使关键准入控制在首个请求前生效，且不能通过 Kubernetes API 删除；HA 集群必须保证所有 API Server 文件一致；
- Handling Undecryptable Resources（KEP-3926）让管理员通过 API 识别和清理因 KMS key 丢失等原因无法解密的对象，减少直接操作 etcd 的需要；
- Prometheus Native Histogram（KEP-5808）进入 Beta；迁移期应确认 Prometheus、远端存储、查询、告警和 dashboard 兼容，并保留 classic histogram 以避免现有规则失效。

## 进入 Alpha 阶段的功能

Alpha 功能默认关闭，API 和行为可能在后续版本不兼容变化。v1.37 的新 Alpha 数量较多，本文只选择与私有云、AI 平台和节点运维关系更直接的能力。

### Pod-Level Checkpoint / Restore（KEP-5823）

过去的 checkpoint 能力以单容器为中心，难以恢复同一 Pod 内共享的网络、IPC 和生命周期状态。Pod-Level Checkpoint / Restore 试图把整个 Pod 作为 checkpoint 单元，为长时间训练、批处理迁移、快速预热、故障分析和取证提供基础。

这项能力依赖 CRIU、容器运行时和节点内核，安全面也很大。当前更适合在隔离测试环境验证 checkpoint 文件保护、跨节点兼容性、外部存储和恢复后的网络身份，不应直接作为生产容灾承诺。

### Volume Health Monitor（KEP-1432）

该 KEP 在重新设计后回到 Alpha，引入 CSI controller / node 侧健康 RPC，把卷或存储后端的 `Inaccessible`、`DataLoss`、`Degraded` 等状态写入 Kubernetes 可读状态。它解决的是“PVC 仍然 Bound，但底层磁盘已损坏或不可达”的观测盲区。

v1.37 只建立标准信号，不定义自动修复动作。平台仍需结合 CSI driver、告警、Pod disruption 和业务数据恢复策略决定如何响应。

### StatefulSet Recreate 更新策略（KEP-3541）

StatefulSet 新增类似 Deployment 的 `Recreate` 策略：删除全部旧 Pod、等待终止完成，再按 `podManagementPolicy` 创建新版本。它能绕过部分 RollingUpdate 卡住的问题，但会主动造成服务中断。

只有能够接受停机、且数据由 PVC 或外部系统可靠保存的工作负载才适合试验。数据库、共识系统和有顺序约束的集群还需要额外验证 fencing、quorum 与恢复顺序。

### 调度与设备管理的 Alpha 延伸

CompositePodGroup、Controller Integration APIs、Job 集成，以及 DRA Derived Attributes、Device Compatibility Groups、Node Allocatable Resource Requests 等能力，已经在前面的 DRA 与 Workload-Aware Scheduling 两个专题中展开，这里不再重复。除此以外，Scheduler Preemption for In-Place Pod Resize 允许为高优先级 Pod 的 Deferred resize 抢占低优先级 Pod，补齐原地扩容在资源不足时的抢占路径。

### 节点、网络与安全探索

- Dynamic Resize of Memory-backed Volumes：通过 Pod `/resize` 子资源原地调整 `medium: Memory` 的 emptyDir `sizeLimit`；
- Default Pod Sysctls：由 kubelet 为节点或节点池上的 Pod 设置默认 sysctl，Pod 显式配置仍可覆盖；
- gRPC Probe TLS 与 HTTP/2 cleartext probe：扩展 kubelet 原生探针协议能力；
- Volume Bind Mount Options：为容器的 volumeMount 增加 `noexec`、`nodev`、`nosuid` 等限制；
- nftables Localhost NodePort Userspace Proxy：为 nftables 模式补充 `localhost:<NodePort>` 兼容路径；
- API Server Authentication to Webhooks：为 admission webhook 请求提供短期、可本地验证的 ServiceAccount token，减少静态 kubeconfig 凭证依赖。

这些能力涉及节点内核、运行时、网络和安全边界，建议只在专用节点池逐项开启，不要一次性批量启用。

## 删除和废弃功能

### `kubectl run --filename/-f`

`kubectl run` 的 `--filename` / `-f` 参数开始弃用。该参数此前已经被静默忽略，因为 `kubectl run` 只会根据命令行中的 NAME、`--image` 等参数生成 Pod。使用文件创建对象应切换到 `kubectl create -f` 或 `kubectl apply -f`。

### Static Pod API 引用

Static Pod 不再允许通过 `secretRef`、`configMapRef` 等字段引用 API Server 资源，相关 feature gate 已删除。Static Pod 配置应改用节点本地文件、静态挂载或由节点配置管理系统分发。

### kube-proxy IPVS

IPVS 在 v1.37 进入明确的弃用告警阶段。升级本身不会立即关闭 IPVS，但现在就应开始记录现有 mode、内核和规则行为，并建立 nftables 或 iptables 的迁移测试环境。

### 其他需要关注的移除

- kubeadm `v1beta3` 配置 API 被移除；
- WAS 的 `scheduling.k8s.io/v1alpha2` 被 `v1beta1` / `v1alpha3` 路径替代；
- 一批已经锁定为 GA 的 feature gates 被清理；不要长期把已经锁定或删除的 gate 写死在组件参数中；
- `gitRepo` volume plugin 在 v1.36 已永久禁用，v1.37 完成对应稳定化清理；仍需使用 `initContainer`、构建时打包或 `git-sync` 替代。

Release Highlights 讨论还列出了若干已到期的 Alpha/Beta API 版本候选。由于当前 RC release notes 尚未完整列出这些条目，正式发布前需要再用最终 API deprecation guide 和 CHANGELOG 复核，不建议仅依据前瞻稿执行删除。

## 建议的升级动作

1. 在 v1.36 集群提前审计 SELinux volume conflict，并为需要旧行为的 Pod 设置 `seLinuxChangePolicy: Recursive`。
2. 扫描 kubelet flags、`eventRecordQPS`、cAdvisor 指标和 `nodes/logs` RBAC，确保升级后 kubelet 能启动且监控不出现大面积缺口。
3. 把 kubeadm `v1beta3` 配置迁移到 `v1beta4`，不要尝试使用尚未可用的实验 `v1` 配置。
4. 如果试用过 WAS，删除 `scheduling.k8s.io/v1alpha2` 对象并迁移 feature gates；没有试用过则保持 `GenericWorkload` 关闭，先在测试集群评估。
5. 明确记录每个集群的 kube-proxy mode；IPVS 集群建立 nftables/iptables 对照测试，空 mode 配置改为显式值。
6. cgroup v1 节点制定迁移期限。Memory QoS、Rootless Kubelet 等节点能力应在 cgroup v2 专用节点池灰度。
7. DRA 用户重点演练设备 taint、driver 重启、ResourceClaim status 和 PodGroup 共享 claim，不只验证首次调度成功。
8. 大规模集群在预生产环境对比 API Server 启动时间、watch cache 初始化、etcd 内存、CRD conversion webhook 并发和 controller staleness 指标。
9. 正式发布日再次核对 v1.37.0 CHANGELOG、release notes、known issues、容器运行时兼容矩阵和发行版支持状态。

## DaoCloud 社区贡献与活动

- DaoCloud 与国内贡献者在 Kubernetes v1.37 周期合入的重点 PR、KEP、文档和本地化贡献；
- KubeCon + CloudNativeCon China 2026 将于 9月7-8日在上海举行，本次活动还包括了 PyTorchCon 和 OpenInfraCon，DaoCloud 届时会有多个分享如下
  - Beyond Model Sharding: Atomic Scheduling and Disaggregated LLM Serving with LeaderWorkerSet 颜开 + 陈子聪（华为）
  - Cybertwin-based Cloud Native Network (CCNN): Network Architecture Innovation and Practice  蓝维洲 + 梁丹丹（鹏城）
  - ⚡ Fast Restarts, Not Just Fast Starts: Accelerating Pod Recovery 范宝发
  - ⚡ Before vLLM starts: Preflight Checks for LWS for LLM Inference on K8S 潘远航
  - End-to-End Observability for LLM Inference: From Token to GPU 谭建，陈泯全
  - Why Your TTFT Lies: Diagnosing PD-Disaggregated LLM Inference with Minimal Cross-Layer Metrics  Kebe & 李辉
  - Kubernetes DRA Architecture: Scheduling, Status, and Topology at Scale 徐俊杰+张康（NVIDIA）
  - Project Lightning Talk: KubeEdge Everywhere: Latest Project Update with industrial cases  张红兵
- KCD 杭州正在议题征集中，截止日期为 2026 年 8 月 todo 日，DaoCloud 开源工程师蔡威是此次活动的组织者之一。
- Kueue、vLLM、DRA、WAS 和 AI Infra 方向的国内社区进展。
- KubeCon 北美主题预告？
  - 11/9 09:38–09:43 — Ubiquitous Edge Computing: KubeEdge Industrial Cases Sharing
Hongbing Zhang，KubeEdge 工业落地案例，5 分钟 Project Lightning Talk。
  - 11/10 11:30–12:00 — Steering the Ship: Ask the Kubernetes Steering Committee
Paco Xu，与 Kat Cosgrove、Maciej Szulik；Kubernetes Steering Committee 问答。
  - 11/12 13:45–14:15 — Explore TAG Workloads Foundation: Core Runtime, Batch Scheduling, and Moar
Paco Xu，与 NVIDIA、Broadcom 等共同介绍 TAG Workloads Foundation。

TODO：增加一个预告图？

## 发行说明

截至本文整理时间，建议持续跟踪以下官方页面：

- Kubernetes v1.37 CHANGELOG：<https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.37.md>
- Kubernetes v1.37 release notes draft：<https://github.com/kubernetes/sig-release/blob/master/releases/release-1.37/release-notes/release-notes-draft.md>
- Kubernetes v1.37 发布日程：<https://www.kubernetes.dev/resources/release/>

正式发布后，需要把文中的“计划”“预计”“进入 RC”等表述改为最终时态，并补充官方发布博客、主题 Logo、最终统计与 known issues。

## 历史文档

- Kubernetes v1.36 正式发布：DRA 加速成熟，WAS 迈向原生工作负载调度
- K8s 1.35 发布！安装/升级变化巨大，新特性 Gang Scheduling 重磅来袭！
- 迎风破浪的三只熊——Kubernetes v1.34 发布，看点全解析
- 重磅！K8s 正式支持 Sidecar 容器，v1.33 版本这些改动将影响你的集群
- Kubernetes 1.32 还在写 Webhook? 你已经 OUT 了！
- Kubernetes 1.31 发布！十年 OCI 镜像借着 AI 的风终于加入 Volume 的大家庭
- 最可爱的版本 UwU - Kubernetes v1.30 发布！
- Kubernetes 1.29 全新特性：抛弃 iptables 还在等什么...
- Kubernetes 1.28 震撼发布，Sidecar Containers 迎面而来
- 近两年功能增加最多！Kubernetes 1.27 正式发布
- Kubernetes 正式发布 v1.26，稳定性显著提升
- Kubernetes 1.25 正式发布，多方面重大突破
- Kubernetes 1.24 走向成熟的 Kubernetes

## 参考

1. Kubernetes v1.37 Sneak Peek <https://kubernetes.io/blog/2026/07/31/kubernetes-v1-37-sneak-peek/>
2. Kubernetes 1.37 Release Highlights Discussion #3051 <https://github.com/kubernetes/sig-release/discussions/3051>
3. Kubernetes v1.37 Release Information <https://www.kubernetes.dev/resources/release/>
4. Kubernetes v1.37 CHANGELOG <https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.37.md>
5. Kubernetes v1.37 Release Notes Draft <https://github.com/kubernetes/sig-release/blob/master/releases/release-1.37/release-notes/release-notes-draft.md>
6. Kubernetes 1.37: Deep dive into new alpha features（作为 Alpha 选题清单参考，技术阶段以官方 KEP 和 release 分支为准）<https://palark.com/blog/kubernetes-1-37-release-features/>
7. Kubernetes Enhancement Proposals <https://kep.k8s.io/>
8. SELinux Volume Label Changes goes GA（含 v1.36 到 v1.37 升级路径）<https://kubernetes.io/blog/2026/04/22/breaking-changes-in-selinux-volume-labeling/>
