# Kubernetes v1.36 即将发布：升级风险前置，准入与资源编排能力持续演进

2026 年 4 月 22 日（周三），Kubernetes v1.36 计划正式发布。

本文基于截至 2026-04-07 的上游公开信息与 `kubernetes/sig-release#2958` 讨论内容整理，正式发布当天请以 `CHANGELOG-1.36.md` 与 release notes 为准。

与前几个版本类似，v1.36 仍然延续了“稳定化 + 可扩展性 + 资源编排演进”的主线。本文按 v1.35 发布稿结构整理，先给出升级必读，再展开重点能力。

## 发布主题和 Logo

截至 2026-04-07，v1.36 的官方发布主题与 Logo 仍以 SIG Release 正式公告为准。本文先聚焦升级风险与关键特性，主题视觉素材可在正式发布后补充。

## 安装/升级注意事项

### API 与对象兼容性

- `Service.spec.externalIPs` 在 v1.36 开始弃用并给出告警，计划在 v1.43 移除。建议迁移到 `LoadBalancer`、`NodePort` 或 `Gateway API`（KEP-5707）。
- `gitRepo` 卷驱动在 v1.36 起永久禁用且不可重新启用。建议改为 `initContainer`、镜像构建阶段打包或外部 `git-sync`（KEP-5040）。

### 生态与运行时风险

- Ingress NGINX 已于 2026 年 3 月退役，不再提供后续修复和安全更新。现网虽可继续运行，但建议尽快完成迁移路线评估。
- 建议在升级窗口前完成 SELinux 相关盘点，识别现有工作负载对标签与策略的隐式依赖，避免在发布窗口集中暴露兼容问题。

## GA 和稳定的功能

GA（General Availability）代表功能进入稳定阶段，可作为生产可用能力评估。v1.36 的核心信号是：准入能力进一步内建化，身份签名治理能力增强，节点与资源侧能力持续成熟。

### Mutating Admission Policies 升级到 GA（KEP-3962）

过去很多团队依赖 mutating webhook 做策略注入、默认值补全和安全控制，但 webhook 体系本身有明显运维成本：需要额外部署与证书管理、故障会放大 API 请求路径风险、升级和排障链路也更长。对于多集群平台，这类“外置准入逻辑”通常是稳定性薄弱点之一。

v1.36 中，基于 CEL 的 Mutating Admission Policies 进入 GA，意味着“声明式、进程内”的变更准入能力进入稳定阶段。它与已 GA 的 Validating Admission Policy 形成闭环，让集群在“校验 + 变更”两个环节都能减少对外部 webhook 的硬依赖。对平台团队来说，最直接价值是把一部分高频、可声明化的准入逻辑收敛到 apiserver 内部能力，降低控制面外围组件复杂度。

落地上建议采用分层迁移：先从无副作用、纯字段变换的 webhook 规则迁入 CEL；再评估是否继续保留少量复杂 webhook（例如强依赖外部系统查询或复杂状态编排的场景）。这样可以在不牺牲策略能力的前提下，逐步换取更可控的稳定性与变更成本。

### ServiceAccount Token 外部签名升级到 GA（KEP-740）

传统模式下，kube-apiserver 直接持有 ServiceAccount token 签名密钥，密钥生命周期与控制面节点绑定较深。对有合规要求或集中密钥管理要求的组织，这会带来审计、轮换、权限隔离上的治理压力。

KEP-740 在 v1.36 的稳定化价值，在于把签名能力标准化地委托给外部系统（如 HSM、云 KMS），让 Kubernetes 与企业既有密钥治理体系对齐。它并不只是“换个签名位置”，而是把密钥保护边界、轮换流程和审计责任从单集群节点层面提升到统一安全基础设施层面。

实施时建议重点关注三件事：第一，签名链路延迟和可用性对认证路径的影响；第二，外部签名服务故障时的降级和恢复流程；第三，密钥轮换演练与审计证据闭环。做完这三项验证，外部签名才能真正转化为生产收益而不仅是架构升级。

### Volume Group Snapshot 面向 GA（KEP-3476）

单卷快照难以覆盖多卷应用的一致性恢复诉求：当数据库数据卷、日志卷、元数据卷之间存在写入顺序关系时，分别快照往往无法保证同一恢复点。对训练平台、状态型中间件和复杂事务应用，这个问题在故障恢复时尤为明显。

Volume Group Snapshot 的核心价值，是把“多个相关卷”作为一个逻辑组进行快照与恢复，目标是提供 crash-consistent 的恢复点。它依赖 CSI 侧的一组扩展 API，能力边界清晰，也更利于存储厂商和平台团队在统一接口下协作。

从平台实践看，这项能力最适合进入“备份恢复演练”而不仅是功能开关验证：建议把它纳入 RTO/RPO 目标校验，针对典型多卷工作负载做周期性恢复演练。只有把恢复链路跑通并量化结果，才能真正发挥该特性的业务价值。

### 细粒度 Kubelet API 鉴权（KEP-2862，Stable）

该能力允许按请求类型（如 `exec`、`logs`、`metrics`、`port-forward`）进行更细粒度授权，而不是把 kubelet 端点访问作为粗粒度权限整体放开。它的实际意义是让节点侧接口更接近最小权限模型，降低“拿到一种权限即可过度访问”的风险。

### DRA AdminAccess for ResourceClaims（KEP-5018，Stable）

该特性支持以特权模式创建 ResourceClaim，用于在设备已被占用时执行管理类任务（如健康检查、状态查看）。对共享加速器环境而言，这有助于把“运维可见性”与“业务占用路径”解耦，减少排障时对业务负载的干扰。

### User Namespaces（GA）落地实践

User Namespaces 进入 GA 后，容器内用户与宿主机用户隔离的工程可用性更强，适合在多租户场景中与 RuntimeClass、seccomp、SELinux/AppArmor 等机制配套落地，形成“默认最小权限 + 分层隔离”的节点安全基线。

### Node Log Query 进入稳定阶段（SIG Windows 补充）

根据 `#2958` 在 2026-03-31 的补充，Node Log Query 在 v1.36 进入 stable，意味着通过 kubelet `/logs` 查询节点服务日志的能力进一步固化。该能力覆盖 Linux 与 Windows 节点，并可处理系统日志提供器与文件日志路径。

从生产使用角度，仍需注意配置边界：能力稳定化不等于默认全面开放。是否开放系统日志处理仍依赖 kubelet 配置项 `enableSystemLogHandler`。建议将其作为“故障排查开关”纳入运维手册，而不是长期默认暴露。

### 更新总览

- [KEP-3962 Mutating Admission Policies（GA）](https://kep.k8s.io/3962)
- [KEP-740 ServiceAccount Token 外部签名（GA）](https://kep.k8s.io/740)
- [KEP-3476 Volume Group Snapshot（GA 方向）](https://kep.k8s.io/3476)
- [KEP-2862 细粒度 Kubelet API 鉴权（Stable）](https://kep.k8s.io/2862)
- [KEP-5018 DRA AdminAccess for ResourceClaims（Stable）](https://kep.k8s.io/5018)
- [KEP-5073 Declarative Validation（GA 方向）](https://kep.k8s.io/5073)

## 进入 Beta 阶段的功能

Beta 阶段功能通常已具备较高可用性，建议先在 staging 与灰度环境系统性验证，再分批引入生产。

### Constrained Impersonation（KEP-5284，Beta）

它允许发起模拟（impersonation）的一方主动将自身可用权限进一步收敛到子集，避免直接获得目标身份的完整权限视图。对多租户平台和审计敏感场景，这让模拟机制更适合纳入日常运维而非仅限特权操作。

### IP/CIDR Validation Improvements（KEP-4858，Beta）

该改动收紧了非规范和歧义 IP/CIDR 写法的接受范围，减少不同实现间解释不一致引发的安全与互通问题。升级前建议先做配置巡检，清理历史遗留的“可解析但不规范”地址写法，避免在发布窗口触发阻塞。

### statusz / flagz（KEP-4827、KEP-4828，Beta）

核心组件的 `/statusz` 与 `/flagz` 能力升级到 beta 且默认启用，使组件运行状态和关键配置暴露方式更一致。对平台可观测体系来说，这提升了控制面日常巡检和基线核对效率。

### Mixed Version Proxy（KEP-4020，Beta）

该能力在版本偏斜场景下把请求转发到可处理该资源的 API Server，并提供更完整的聚合发现视图。它对“滚动升级中偶发 404/发现不一致”的缓解价值较高，适合作为升级窗口稳定性增强项来评估。

### Stale Controller Detection and Mitigation（KEP-5647，Beta）

KEP-5647 主要解决的是“控制器基于陈旧 cache 做决策”的问题。Kubernetes controller 通常通过 informer cache 读取对象状态，而这个 cache 来自 apiserver 的 watch stream，本质上是最终一致的；在大规模、高 churn 或 apiserver/watch 延迟场景下，controller 的本地视图可能落后于真实状态，进而导致重复 reconcile、错误删除 Pod、错误扩缩容或无意义写入。

该 KEP 的核心机制，是让 controller 能感知 informer cache 当前推进到的 `resourceVersion`，并在关键写入后记录对应的 `resourceVersion`；下一轮 reconcile 前，如果本地 cache 尚未追上前一次写入，就跳过本轮处理并 requeue，等 cache 追上后再继续。它的价值不是让所有 controller 都变成强一致，而是在高风险控制器和关键决策点上提供“读己之写”的保护，把原来依赖经验判断的 stale read 风险，转化为可检测、可等待、可回退的控制器机制。

与此相关的 `server-side sharded list/watch` 仍属于 Alpha 能力，放在 Alpha 小节中单独说明。

### DRA 1.36 打包更新（稳定化 + 新能力并行）

DRA 在 v1.36 的信号不是单点特性，而是多项能力并行推进：包括 `prioritized list`、`extended resource`、`partitionable devices`、`device taints`、`binding conditions`，以及 workload/native resource/visibility 等方向。对 AI 与异构算力平台，更建议将其作为一组“资源编排能力跃迁”来评估。

### SIG Node 在 #2958 的新增重点：DRA 与节点能力并进

根据 `#2958` 在 2026-04-02 的补充，SIG Node 强调 v1.36 的 DRA 不是单一功能升级，而是“多条能力线并行推进”：包括可分区设备、资源健康状态、扩展资源路径等。对平台侧的直接意义是，资源调度策略可以从“是否可分配”进一步走向“按健康度、按粒度、按回退策略分配”。

同一批次补充里还强调了若干节点与运行时能力的成熟度提升，包括 User Namespaces、PSI 相关能力、OCI 卷源，以及 kubelet 侧针对大规模容器场景的 CRI list streaming 和 Memory QoS 行为调整。将这些点合并看待，更准确的解读是：v1.36 在“节点可扩展性 + 资源隔离精细化”方向上形成了联动改进，而不只是单点特性毕业。

### 更新总览

- [KEP-5284 Constrained Impersonation](https://kep.k8s.io/5284)
- [KEP-4858 IP/CIDR Validation Improvements](https://kep.k8s.io/4858)
- [KEP-4827 statusz](https://kep.k8s.io/4827)
- [KEP-4828 flagz](https://kep.k8s.io/4828)
- [KEP-4020 Mixed Version Proxy](https://kep.k8s.io/4020)
- [KEP-5647 Stale Controller Detection and Mitigation](https://kep.k8s.io/5647)

## 进入 Alpha 阶段的功能

Alpha 功能默认关闭，建议仅在边界可控场景试点，并明确可观测基线、回滚路径和启停条件。这里不需要把 25 个 Alpha 功能逐项展开，更适合按博客叙事分成几条主线：资源编排与弹性伸缩、控制面/节点可扩展性、可观测性与准入治理。WAS/Gang Scheduling 是 v1.36 Alpha 的重要组成，通常应与独立 WAS 小节合并阅读，这里不再重复展开。

### 资源编排与弹性伸缩：DRA Alpha 与 HPA Scale to Zero

DRA 在 v1.36 的 Alpha 更新主要扩展“设备信息如何暴露、资源如何表达、容量如何被看见，以及 workload 级资源声明如何参与调度”。其中，Device Attributes in Downward API 让容器能读取 DRA driver 暴露的设备元数据；List Types for Attributes 让 ResourceSlice 属性可以表达 NUMA、PCIe root 等多值拓扑关系；Native Resource Requests 探索将 CPU/Memory 等核心资源纳入 DRA 统一管理；Resource Availability Visibility 面向容量排障和规划提供更好的资源池可见性；ResourceClaim Support for Workloads 则让 PodGroup/Workload 级对象能够参与 DRA ResourceClaim 编排。

这部分最值得放在 DRA 主线里讲，而不是在 Alpha 小节里重复展开。它传递出的核心信号是：DRA 正在从“加速器设备分配框架”走向更通用的资源编排框架。生产集群应优先评估 Beta/Stable DRA 能力，Alpha 能力更适合在 AI/批处理专用测试环境中验证 driver、scheduler 与控制器之间的协作边界。

HPA Scale to Zero 则是另一个值得单独提的 Alpha 能力。HPA 在 Object/External metrics 场景支持从 0 到非 0 的伸缩能力，为事件驱动和低频工作负载提供更激进的成本优化空间。它不适用于 CPU/Memory 这类依赖运行中 Pod 的资源指标，而是更适合队列长度、外部事件积压量等可以在副本数为 0 时仍然被观测到的指标。试点时要重点关注冷启动时延、指标时效、误扩缩容保护和回滚路径。

### 控制面与节点可扩展性：Server-side Sharded List/Watch 与 CRI List Streaming

KEP-5866 主要解决的是“apiserver list/watch 流量无法真正水平分片”的问题。在大集群中，Pod 等核心资源事件量很高，很多控制器或观测组件希望通过多副本水平扩展来分摊压力；但传统 client-side sharding 下，每个副本仍然要从 apiserver 接收完整 watch stream，再在本地反序列化、过滤并丢弃不属于自己的对象。结果是副本数越多，整体网络、CPU 和内存浪费越大。

该 KEP 提出在 LIST/WATCH 请求中加入服务端分片能力，例如通过 `shardSelector` 指定 shard key 和 hash range，由 apiserver 在源头过滤对象和事件，使每个 watcher 只收到自己负责的 shard。它的效果是把“业务侧假分片”升级为“API Server 原生真分片”，降低 watch fan-out、客户端反序列化和无效事件处理成本，为 kube-state-metrics、未来 sharded controller 以及更大规模控制面提供基础扩展原语。

CRI List Streaming 则把类似问题放到节点侧处理：它为 kubelet 与容器运行时之间的 List 类调用引入服务端流式返回能力，避免一次性返回大量容器或镜像信息时造成 kubelet 内存峰值和响应延迟。节点上 Pod、容器和镜像数量越多，这类“单次大响应”的压力越明显。

如果本文要强调“大规模集群控制面/节点稳定性”，这两项比许多零散 Alpha 更值得介绍。试点时建议重点观察 apiserver list/watch 延迟、watch cache 行为、控制器重连、kubelet 内存占用、CRI 调用耗时，以及容器运行时实现对流式 List 的支持成熟度。

### 可观测性：Native Histogram Support for Kubernetes Metrics

v1.36 引入 Kubernetes 指标的 Native Histogram 支持，让控制面组件可以导出更高分辨率的延迟分布数据。相比传统 Prometheus histogram 依赖固定 buckets 的方式，Native Histogram 使用更稀疏、动态的表达方式，在不显著增加手工 bucket 维护成本的情况下，提升对长尾延迟和突发抖动的观察能力。

对平台团队来说，这项能力最直接服务于 apiserver 等核心组件的 SLI/SLO 建设。试点时建议先从观测链路兼容性开始：确认 Prometheus、远端存储、告警规则和 dashboard 对 native histogram 的支持程度，再逐步评估采集开销、保留周期与查询成本。

### 准入治理：Manifest Based Admission Control Config

该能力把准入控制配置进一步推向结构化、声明式的 manifest 管理方式，减少对分散命令行参数和独立配置文件的依赖。它和 Mutating Admission Policies 解决的问题不同：Mutating Admission Policies 关注“准入逻辑怎么声明”，而 Manifest Based Admission Control Config 更关注“准入插件配置如何被一致地管理、审计和交付”。

对多集群平台而言，这有助于把 admission plugin 配置纳入统一的配置发布和变更审计流程。由于仍处于 Alpha，建议先用于配置基线验证和测试集群演练，避免直接承载生产关键准入路径。

### 其他值得短提的 Alpha 功能

如果篇幅允许，还可以短提几项更贴近应用运行时和平台操作体验的 Alpha 能力：Per-container ulimits 让不同容器拥有更细粒度的 ulimit 配置；Container Stop Signals 让容器停止信号更可控，利于优雅退出；Pod Level Resource Managers 与静态 CPU Manager 相关的 in-place 资源更新方向，体现节点资源管理继续精细化；Deployment Pod Replacement Policy 与 StatefulSet Recreate 更新策略则属于 workload 控制器行为的早期探索。它们可以作为“其他 Alpha 候选方向”一笔带过，不建议在本文主线中展开。

### 更新总览

- [KEP-2021 HPA Scale to Zero](https://kep.k8s.io/2021)
- [KEP-5304 DRA: Device Attributes in Downward API](https://kep.k8s.io/5304)
- [KEP-5491 DRA: List Types for Attributes](https://kep.k8s.io/5491)
- [KEP-5517 DRA: Native Resource Requests](https://kep.k8s.io/5517)
- [KEP-5677 DRA: Resource Availability Visibility](https://kep.k8s.io/5677)
- [KEP-5729 DRA: ResourceClaim Support for Workloads](https://kep.k8s.io/5729)
- [KEP-5866 Server-side Sharded List/Watch](https://kep.k8s.io/5866)
- [KEP-5825 CRI List Streaming](https://kep.k8s.io/5825)
- [KEP-5808 Native Histogram Support for Kubernetes Metrics](https://kep.k8s.io/5808)
- [KEP-5793 Manifest Based Admission Control Config](https://kep.k8s.io/5793)
- [KEP-5832 Workload Aware Scheduling / PodGroup API 解耦](https://kep.k8s.io/5832)
- [KEP-5732 拓扑感知工作负载调度](https://kep.k8s.io/5732)
- [KEP-5710 工作负载感知抢占](https://kep.k8s.io/5710)
- [KEP-5547 WAS 集成到 Job 控制器](https://kep.k8s.io/5547)
- [KEP-4671 Gang Scheduling 基础能力](https://kep.k8s.io/4671)
- [KEP-5758 Per-container ulimits configuration](https://kep.k8s.io/5758)
- [KEP-4960 Container Stop Signals](https://kep.k8s.io/4960)
- [KEP-5526 Pod Level Resource Managers](https://kep.k8s.io/5526)
- [KEP-5554 静态 CPU Manager 场景下的 in-place 资源更新](https://kep.k8s.io/5554)
- [KEP-5882 Deployment Pod Replacement Policy](https://kep.k8s.io/5882)
- [KEP-3541 StatefulSet Recreate 更新策略](https://kep.k8s.io/3541)

> 说明：EvictionRequest API、HPA fallback external metrics 等当前仍以占位探索为主，暂不作为本版 release 主线展开；Deployment Pod Replacement Policy、StatefulSet Recreate 更新策略这类 workload 控制器早期能力可短提，但不建议抢占 WAS/DRA 主线篇幅。

## 删除和废弃功能

### Service `externalIPs` 开始弃用（KEP-5707）

v1.36 起，`Service.spec.externalIPs` 已进入弃用周期并提供告警信号。建议提前完成对象清单扫描与迁移计划，避免后续版本进入移除窗口时形成被动整改。

### `gitRepo` 卷驱动永久禁用（KEP-5040）

v1.36 起 `gitRepo` 卷驱动不可重新启用。建议统一切换至 `initContainer` 拉取、镜像构建阶段打包或外部 `git-sync` 方案，减少运行时拉取代码的安全与稳定性风险。

## 建议的升级动作

1. 全量扫描清单与集群对象，完成 `externalIPs`、`gitRepo` 使用点盘点和迁移计划。
2. 对入口层做维护状态审计，尽快推进 Ingress NGINX 迁移路线。
3. 以 staging 为主验证准入策略、API Machinery 与调度相关改动，再逐步推进生产。
4. 对多卷状态型业务执行组快照恢复演练，量化 RTO/RPO 并形成发布闸门。
5. 升级当天对照最终 `CHANGELOG-1.36.md` 与 release notes 做差异复核。

## DaoCloud 社区贡献

本节建议在正式发布前补充 DaoCloud 社区在 v1.36 周期内的贡献数据（如 SIG 角色、关键 PR/KEP 参与、演讲与维护工作），以保持与往期发布稿结构一致。

## 发行说明

上述内容为 v1.36 发布前精简稿，更多发布细节请以正式发布当日版本说明为准：

- Kubernetes v1.36 CHANGELOG：<https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.36.md>
- Kubernetes v1.36 release notes draft：<https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/release-notes/release-notes-draft.md>

## 历史文档

- [K8s 1.35 发布！安装/升级变化巨大，新特性 Gang Scheduling 重磅来袭！](https://github.com/DaoCloud-OpenSource/docs/blob/main/kubernetes/sig-release/v1.35/release.md)

## 参考

1. Kubernetes v1.36 Sneak Peek <https://kubernetes.io/blog/2026/03/30/kubernetes-v1-36-sneak-peek/>
2. Kubernetes v1.36 主题讨论 <https://github.com/kubernetes/sig-release/discussions/2958>
3. Kubernetes v1.36 Release Announcement PR <https://github.com/kubernetes/website/pull/55151>
4. Kubernetes v1.36 发布分支说明 <https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/README.md>
5. Kubernetes v1.36 变更日志 <https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.36.md>
6. Kubernetes v1.36 Release Notes Draft <https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/release-notes/release-notes-draft.md>
7. KEP-5707 <https://kep.k8s.io/5707>
8. KEP-5040 <https://kep.k8s.io/5040>
9. KEP-3962 <https://kep.k8s.io/3962>
10. KEP-740 <https://kep.k8s.io/740>
11. KEP-3476 <https://kep.k8s.io/3476>
