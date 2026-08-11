# Kubernetes v1.37 前瞻：Gang Scheduling 进入 Beta，DRA 与节点资源继续成熟

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

## 重要主线一：WAS / Gang Scheduling 进入 Beta

v1.36 为 Workload、PodGroup、Gang Scheduling、拓扑感知调度和工作负载感知抢占搭起了 Alpha 框架。v1.37 的关键变化是，Gang Scheduling、Workload API 和 Workload-aware Preemption 共同进入 Beta，并统一由 `GenericWorkload` feature gate 控制。

传统 kube-scheduler 逐个处理 Pod。对于分布式训练、MPI、大规模批处理等工作负载，部分 Pod 先运行、其余 Pod 长期 Pending，既无法完成任务，又会占住 GPU、CPU 和内存。WAS 把 PodGroup 作为调度实体，先判断至少 `minCount` 个成员能否形成可行放置，再进入绑定流程。

v1.37 进一步补齐了几块关键能力：

| KEP | 能力 | v1.37 阶段 | 价值 |
| --- | --- | --- | --- |
| [4671](https://kep.k8s.io/4671) | Workload API 与 Gang Scheduling | Beta | 以 PodGroup 为单位完成队列、调度周期和最小成员数判断 |
| [5710](https://kep.k8s.io/5710) | Workload-aware Preemption | Beta | 避免只抢占单个 Pod 所需资源、但整组仍然放不下 |
| [5729](https://kep.k8s.io/5729) | DRA ResourceClaim for Workloads | Beta | 让一组 Pod 共享或生成工作负载级设备声明 |
| [6012](https://kep.k8s.io/6012) | CompositePodGroup | Alpha | 表达分层、组合式工作负载，例如多个 prefill/decode 子组 |
| [6089](https://kep.k8s.io/6089) | Controller Integration APIs | Alpha | 为 Job、JobSet、LeaderWorkerSet、RayJob 等控制器提供统一集成积木 |
| [5547](https://kep.k8s.io/5547) | Job 集成 Workload API | Alpha 持续演进 | 让用户在 Job 中显式表达 gang、拓扑和 disruption 意图 |

Beta 并不意味着默认在生产中自动启用。`GenericWorkload` 在 v1.37 release 分支中仍默认关闭。建议先在专用 AI/批处理集群验证：队列等待时间、PodGroup 调度成功率、抢占后的任务完成时间、DRA ResourceClaim 生命周期，以及与 Kueue、Volcano 等现有批调度系统的职责边界。

## 重要主线二：DRA 从核心框架走向可运营设备管理

DRA 核心框架已在 v1.34 GA，v1.35 和 v1.36 持续补齐设备健康、容量、分区、污点和工作负载级声明。v1.37 的重点不是再证明 DRA 可用，而是把常用迁移路径和运维能力稳定下来。

### 进入 GA 的 DRA 能力

- [KEP-5004](https://kep.k8s.io/5004) Extended Resources via DRA：让 `nvidia.com/gpu: 2` 这类传统 extended resource 请求可由 DRA driver 满足，为 Device Plugin 到 DRA 的平滑迁移提供兼容入口。
- [KEP-5055](https://kep.k8s.io/5055) Device Taints and Tolerations：设备故障或维护时可以给单个设备打 taint，阻止新工作负载使用，并通过 toleration 表达显式接受。
- [KEP-4817](https://kep.k8s.io/4817) ResourceClaim Device Status：DRA driver 可以在 ResourceClaim status 中报告设备状态和标准化网络接口数据，方便网络设备、RDMA 与多网卡场景集成。

### 继续进入 Beta / Alpha 的能力

- Workload 级 ResourceClaim 进入 Beta，使 DRA 与 WAS 的生命周期真正接上；
- Device Attributes in Downward API 进入 Beta，让容器更容易读取分配到的设备信息；
- Derived Attributes、Device Compatibility Groups、Node Allocatable Resource Requests 等保持 Alpha，继续探索跨 driver 拓扑对齐、设备兼容性和 CPU/内存等原生资源的统一分配；
- v1.37 新增标准 `resource.kubernetes.io/numaNode` 属性，为不同 DRA driver 表达 NUMA 信息提供共同语言。

对 AI 平台来说，v1.37 更适合把“设备 taint、extended resource 兼容、ResourceClaim 状态”作为一组运维闭环来验证，而不是只测试能否分配 GPU。建议故障演练至少覆盖：设备变为不健康、driver 重启、ResourceSlice 重建、PodGroup 共享 claim、节点维护和 DRA 调度回退。

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

### WAS 与 DRA 的下一阶段

- CompositePodGroup 表达多层 PodGroup 和复杂 AI 推理/训练拓扑；
- Controller Integration APIs 为 JobSet、LeaderWorkerSet、KubeRay 等上层控制器提供统一调度意图结构；
- Scheduler Preemption for In-Place Pod Resize 允许为高优先级 Pod 的 Deferred resize 抢占低优先级 Pod；
- DRA Derived Attributes 用 CEL 对不同 driver 的属性做归一化，便于 GPU、NIC 等设备按共同 NUMA 拓扑对齐；
- DRA Device Compatibility Groups 避免不兼容的租户或设备分区同时使用同一硬件；
- DRA Node Allocatable Resource Requests 继续探索通过 DRA 统一管理 CPU、内存等节点原生资源。

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

本节待正式发布前结合 v1.37 release notes、Kubernetes 贡献数据和国内活动信息补充，避免在 RC 阶段沿用 v1.36 的旧名单或活动状态。

建议补充范围：

- DaoCloud 与国内贡献者在 Kubernetes v1.37 周期合入的重点 PR、KEP、文档和本地化贡献；
- KubeCon + CloudNativeCon China 2026、KCD 杭州及相关社区分享；
- Kueue、vLLM、DRA、WAS 和 AI Infra 方向的国内社区进展。

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
