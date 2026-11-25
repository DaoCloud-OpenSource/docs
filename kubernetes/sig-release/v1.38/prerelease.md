# Kubernetes v1.38 前瞻：弃用移除、升级必读与重点增强

作者：（待补充）（2026 年 11 月 25 日）

Kubernetes v1.38 预计在 2026 年 12 月 16 日（北美时间周三）发布。v1.38 已于 11 月 17 日进入代码冻结（Code Freeze），本文基于代码冻结后的开发状态整理主要变化。还有少量代码冻结例外在收尾，正式发布前内容仍可能调整，一切以最终的 CHANGELOG 和 Release Notes 为准。

按 v1.38 增强跟踪看板的状态（截至 11 月 22 日），本版本共跟踪 69 项增强：12 项进入 Stable，28 项处于 Beta，29 项处于 Alpha。

> 专题解读：[Kubernetes v1.38 AI Infra 解读](ai-infra.md)、[Kubernetes v1.38 Agent 视角解读](agent.md)。

## 目录

- Kubernetes API 的弃用与移除流程
- v1.38 的弃用与移除
- 持续进行中的重要变化
- 升级前必读：v1.38 的破坏性变更
- v1.38 重点增强
- v1.38 跟踪的增强一览
- 更多信息

## Kubernetes API 的弃用与移除流程

Kubernetes 有明确的 API 与特性弃用策略：

- 稳定版 API 只有在有新的稳定替代版本时才会被弃用。
- 被弃用后会保留一段周期并给出告警，不同稳定级别（GA/Beta/Alpha）对应不同的最短保留期。
- 移除后需迁移到替代方案。
- 特性 GA 后，对应的特性门控会先锁定为开启，过几个版本再删除。门控删除后如果还写在 `--feature-gates` 或配置文件里，组件会因为无法识别而启动失败。

v1.38 又有一批门控走到了“删除”这一步，升级前建议把各组件的启动参数和配置文件都检查一遍。

## v1.38 的弃用与移除

### 1) kubectl：移除 WebSocket 的 opt-out 环境变量

- 从 v1.30 起，`kubectl exec`、`attach`、`cp` 默认使用 WebSocket 而不是 SPDY；从 v1.31 起，`kubectl port-forward` 也是如此。
- v1.38 中 KEP-4006 进入 GA，`TranslateStreamCloseWebsocketRequests`、`PortForwardWebsockets`、`AuthorizePodWebsocketUpgradeCreatePermission`、`ExtendWebSocketsToKubelet` 四个门控锁定为开启，最早在 v1.41 删除。
- `KUBECTL_REMOTE_COMMAND_WEBSOCKETS` 和 `KUBECTL_PORT_FORWARD_WEBSOCKETS` 两个环境变量被移除。之后 kubectl 总是先尝试 WebSocket，必要时再回退到 SPDY。
- 建议：清理脚本和 CI 里的这两个变量，它们之后不再生效。`AuthorizePodWebsocketUpgradeCreatePermission` 锁定后，通过 WebSocket 发起 `pods/exec`、`pods/attach` 等请求需要对应子资源的 `create` 权限，不能再临时关闭这项检查。如果有自定义角色只授予了这些子资源的 `get`，需要提前补上。

参考：

- [KEP-4006：Transition from SPDY to WebSockets](https://kep.k8s.io/4006)

### 2) kuberc：`v1` 不再支持凭证插件 allowlist 中的 `name`

- 在 Beta 的 `kubectl.config.k8s.io/v1beta1` 中，凭证插件 allowlist 的条目可以用 `name` 作为 `command` 的别名，`name` 自 v1.36 起已弃用。
- v1.38 中 kuberc 进入 GA，新的 `kubectl.config.k8s.io/v1` 不再包含 `name`。
- 建议：把 kuberc 文件迁移到 `v1` 时，把 `name` 改成 `command`。

参考：

- [KEP-3104：Separate kubectl user preferences from cluster configs](https://kep.k8s.io/3104)

### 3) 一批特性门控与 API 被删除

- `DynamicResourceAllocation`：DRA 始终开启，继续设置这个门控会报错。
- `SeparateTaintEvictionController`：升级 kube-controller-manager 前需从 `--feature-gates` 中删除。如需关闭基于污点的驱逐，改用 `--controllers=-taint-eviction-controller`。
- Windows：`WinDSR`、`WinOverlay` 门控删除（KEP-5100）。Overlay 网络不再需要门控，DSR 继续通过 kube-proxy 的 `winkernel.enableDSR` 配置或 `--enable-dsr` 参数控制。`WindowsHostNetwork` 门控也一并删除。
- 其他已 GA 并锁定的门控删除：`PodSchedulingReadiness`、`MatchLabelKeysInPodAffinity`、`CustomResourceFieldSelectors`、`CRDValidationRatcheting`、`RecoverVolumeExpansionFailure`、`SupplementalGroupsPolicy`。
- API 删除：`metrics.k8s.io/v1alpha1`；`apidiscovery.k8s.io/v2beta1` 及 `AggregatedDiscoveryRemoveBetaType` 门控。
- 指标 `endpoint_slice_controller_changes` 弃用，改用 `endpoint_slice_controller_changes_total`，弃用期内两者同时输出。
- Kubemark 从 kubernetes/kubernetes 主仓库中移除。

参考：

- [Kubernetes v1.38 CHANGELOG](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.38.md)

## 持续进行中的重要变化

### 1) cgroup v1 支持将在未来版本移除

- 从 v1.35 起，kubelet 默认拒绝在 cgroup v1 节点上启动，除非在 kubelet 配置中显式设置 `failCgroupV1: false`。
- v1.38 仍保留这个选项，但它只是短期过渡手段。cgroup v1 支持计划在未来版本移除，具体版本尚未确定。
- 建议：尽快规划迁移到 cgroup v2，特别是仍运行 CentOS 7 等默认使用 cgroup v1 的操作系统的节点。

参考：

- [KEP-5573：Remove cgroup v1 support](https://kep.k8s.io/5573)
- [关于 cgroup v2](https://kubernetes.io/zh-cn/docs/concepts/architecture/cgroups/)

### 2) 准入 Webhook 不再收到认证/鉴权类请求

- `ExcludeAdmissionWebhookVirtualResources` 门控自 v1.37 起默认开启。
- 开启后，Validating 和 Mutating Webhook 不再收到 TokenReview、SelfSubjectReview、SubjectAccessReview、SelfSubjectAccessReview、LocalSubjectAccessReview、SelfSubjectRulesReview 这几类虚拟资源的请求。
- 创建或更新引用这些资源的 Webhook 配置会收到弃用告警，kube-apiserver 启动时也会在日志里列出受影响的配置。
- 如果确实依赖这些请求，目前还可以临时关闭该门控。该特性 GA 时门控会锁定。

参考：

- [KEP-5793：Manifest Based Admission Control Config](https://kep.k8s.io/5793)

## 升级前必读：v1.38 的破坏性变更

### 1) 容器运行时必须通过 CRI 上报 cgroup driver

v1.37 及更早版本的 kubelet 在运行时不支持时，会回退到 kubelet 配置里的 `cgroupDriver`。v1.38 计划默认关闭这条回退路径：

- kubelet 要求容器运行时实现 CRI `RuntimeConfig` RPC，containerd 2.0+ 和 CRI-O 1.28+ 已支持。
- `KubeletConfiguration.cgroupDriver` 字段弃用。新增的 `LegacyCgroupDriverFallback` 门控本身即为弃用状态，默认关闭。不开启这个门控时，配置 `cgroupDriver` 会被拒绝。
- kubelet 启动前容器运行时必须已经就绪。多次重试后仍拿不到 cgroup driver，kubelet 会在注册节点之前退出。
- kubeadm 不再把 `cgroupDriver` 默认设为 `systemd`。`ContainerRuntimeVersion` 预检会检查运行时是否支持该 RPC：kubelet 1.38 及以上报错，更早版本给出告警。
- 临时方案：设置 `LegacyCgroupDriverFallback=true`，并配置与运行时一致的 `cgroupDriver`。使用 kubeadm 时还要加上 `--ignore-preflight-errors=ContainerRuntimeVersion`，并在升级前更新 `kubelet-config` ConfigMap。
- `KubeletCgroupDriverFromCRI` 门控同时删除，自动探测保持开启。

> 注意：截至 11 月 25 日，对应实现 [kubernetes/kubernetes#136731](https://github.com/kubernetes/kubernetes/pull/136731) 仍处于开放状态，尚未合入；该变更已进入 v1.38 milestone 并获得 approve。具体以 v1.38 Release Notes 为准。

建议：还在使用 containerd 1.7 的集群，升级到 v1.38 前先把运行时升级到 containerd 2.x。

参考：

- [KEP-4033：Discover cgroup driver from CRI](https://kep.k8s.io/4033)

### 2) Pod 级资源：`LimitedSwap` 节点上的 swap 行为变化

- 影响范围：使用 cgroup v2 并开启 `LimitedSwap` 的 Linux 节点。
- v1.38 中，kubelet 依据 Pod 级内存设置计算 swap 限额。
- 如果 Pod 级内存 request 等于 limit，整个 Pod 及其中所有容器都不会使用 swap，即使某个容器的内存 request 小于 limit。
- 建议：升级前检查依赖 swap 的工作负载。

参考：

- [KEP-2837：Pod Level Resource Specifications](https://kep.k8s.io/2837)

### 3) 其他需要留意的变化

- user namespaces：如果 `/etc/subuid` 或 `/etc/subgid` 中范围的上限到达或超过 4294967295，kubelet 会拒绝启动，需要先缩小范围；升级前处于 `ContainerCreating` 的 Pod 需要删除，或先排空节点。
- env 文件：kubelet 拒绝在单引号闭合后紧跟 `#` 的条目（如 `KEY='value'#suffix`），以前会忽略后缀。受影响的容器在修正前可能无法启动。
- 调度器：Alpha 指标 `scheduler_queued_entities`、`scheduler_queue_incoming_entities_total` 的 `type` 标签改名为 `entity`，相关看板和告警要同步修改。树外调度插件调用 `NominatedPodsForNode` 时，第一个参数需要传入 `klog.Logger`。
- DRA：v1.37 中的 Alpha 字段 `pod.status.nodeAllocatableResourceClaimStatuses` 被 `pod.status.extraNodeAllocatableResources` 取代，用过该 Alpha 特性的 Pod 升级后需要重建。
- Go 开发者：CRD 的 API 类型迁移到新的 `k8s.io/apiextensions-api` 模块，protobuf 包名随之变化；cel-go 升级到 v0.33.0，模块路径改为 `cel.dev`。

## v1.38 重点增强

### 1) kuberc（GA）

kuberc 是独立于 kubeconfig 的 kubectl 偏好配置文件，用来放别名、默认参数和凭证插件策略，把集群凭据和个人使用习惯分开。v1.38 使用新的 `kubectl.config.k8s.io/v1`。

- [KEP-3104：Separate kubectl user preferences from cluster configs](https://kep.k8s.io/3104)

### 2) Pod 级资源与原地调整（GA）

可以给整个 Pod 设置 CPU、内存和 hugepages 的 requests/limits，容器在这个总预算内共享资源，带多个 sidecar 的 Pod 不必再逐个容器精算。配套的 Pod 级资源原地调整（In-Place Resize）也一并 GA。使用 `LimitedSwap` 的节点请留意上文的 swap 行为变化。

- [KEP-2837：Pod Level Resource Specifications](https://kep.k8s.io/2837)
- [KEP-5419：Pod Level Resources Support With In Place Pod Vertical Scaling](https://kep.k8s.io/5419)

### 3) DRA 可分区设备、设备绑定条件、可消耗容量（GA）

- 可分区设备：驱动可以同时发布整卡和分区（比如 GPU 整卡与 MIG 切片），调度器只会分配物理上放得下的组合。
- 设备绑定条件：调度器在 PreBind 阶段等待设备就绪后再绑定 Pod，失败或超时（默认 10 分钟）会回滚并重新调度。
- 可消耗容量：同一个设备可以按容量分给多个 ResourceClaim，调度器记录剩余容量。

同时 `DynamicResourceAllocation` 门控删除，DRA 始终开启。

- [KEP-4815：DRA: Partitionable Devices](https://kep.k8s.io/4815)
- [KEP-5007：DRA: Device Binding Conditions](https://kep.k8s.io/5007)
- [KEP-5075：DRA: Consumable Capacity](https://kep.k8s.io/5075)

### 4) WatchList（GA）与大规模集群优化

`WatchList` 和 `WatchListClient` 进入 GA：informer 可以通过流式 watch 拿到初始数据，不再依赖分页 LIST，降低大集群里 kube-apiserver 的内存峰值。控制面相关的还有：

- 服务端分片 List/Watch（KEP-5866，Beta）：客户端可以只订阅自己负责的分片，由 API Server 在源头过滤，方便 kube-state-metrics 这类控制器水平扩展。
- managedFields opt-out（KEP-5958，Alpha）：客户端可以选择不接收 `metadata.managedFields`，kube-controller-manager 和 kube-scheduler 的 informer 已经接入。
- 慢 watch 流暂停与恢复（KEP-6268）、etcd 对象压缩（KEP-6264）、kube-apiserver 加权负载均衡（KEP-6315）等，目前都处于 Alpha。

- [KEP-3157：Allow informers for getting a stream of data instead of chunking](https://kep.k8s.io/3157)

### 5) Workload-Aware Scheduling 继续推进（Beta）

CompositePodGroup API 进入 `scheduling.k8s.io/v1beta1`，拓扑感知工作负载调度（`TopologyAwareWorkloadScheduling`）和 WAS 控制器接入 API 进入 Beta，Job 控制器接入进入 Beta，Deployment 控制器接入以 Alpha 形式加入。

- [KEP-6012：CompositePodGroup API](https://kep.k8s.io/6012)
- [KEP-5732：Topology-aware workload scheduling](https://kep.k8s.io/5732)
- [KEP-5547：WAS: Integrate Workload APIs with Job controller](https://kep.k8s.io/5547)

### 6) StatefulSet `Recreate` 更新策略（Beta）

StatefulSet 滚动更新时，如果新 Pod 一直无法 Ready，更新会卡住；即使修正了配置，控制器也不会自动替换这些卡住的 Pod，只能手动删除。新的 `Recreate` 更新策略用来解决这类问题。

- [KEP-3541：Add Recreate Update Strategy to StatefulSet](https://kep.k8s.io/3541)

### 7) 节点与网络（Beta）

- 原地调整的调度器抢占（KEP-5836，默认开启）：Pod 的扩容请求因节点资源不足进入 `Deferred` 状态时，调度器可以抢占低优先级 Pod 腾出空间。
- nftables 模式的 localhost NodePort（KEP-6032，默认开启）：通过用户态代理支持 `127.0.0.1` 访问 NodePort，补上从 iptables 迁移到 nftables 时的一个兼容性差异。
- Windows CPU 与内存亲和（KEP-4885，默认开启）：管理员需要配置 CPU、Memory 和 Topology Manager 策略才会生效。
- 其他：容器停止信号（KEP-4960）、gRPC 探针 TLS 凭据（KEP-4939）、探针支持 h2c（KEP-5999）、kubelet 默认 Pod sysctls（KEP-5996）、CRI List 流式接口（KEP-5825）。

### 8) 值得关注的 Alpha

- EvictionRequest API（KEP-4563）：驱逐前先通知应用或控制器完成收尾（比如做完手头工作、清理、迁移数据），没有响应者或响应超时就回退到 API 驱逐，仍然遵守 PodDisruptionBudget。需要同时开启 `EvictionRequestAPI` 门控和对应的 Alpha API 组。
- 动态容器（KEP-5972）：通过新的 `pods/dynamic` 子资源，向运行中的 Pod 添加或移除主容器（`DynamicContainers` 门控）。配套的动态 node-local 临时卷（KEP-6395）可以通过同一个子资源动态添加或移除 `emptyDir`、`configMap`、`secret`、`projected`、`image` 卷，新加的卷需要由新加入的容器挂载。
- 网络隔离 Pod（KEP-6313）：Pod 可以不接入默认集群网络，只使用独立的网络命名空间，或者只使用通过 DRA 等方式提供的非默认网络（`NetworkIsolatedPods` 门控）。
- 从 OCI 仓库分发 seccomp profile（KEP-6061，`SecurityProfileOCI` 门控）。
- Pod 级 PID 限制（KEP-6063）：通过 `resources.limits.pids` 设置。
- 条件授权（KEP-5681）、DRA 抢占（KEP-5690）、Deployment 接入 WAS（KEP-6276）等。

## v1.38 跟踪的增强一览

以下按增强跟踪看板当前的阶段整理，最终以正式发布为准。

**Stable（12 项）**：[2837](https://kep.k8s.io/2837) Pod 级资源、[3104](https://kep.k8s.io/3104) kuberc、[3157](https://kep.k8s.io/3157) WatchList、[4006](https://kep.k8s.io/4006) SPDY 迁移到 WebSocket、[4815](https://kep.k8s.io/4815) DRA 可分区设备、[4858](https://kep.k8s.io/4858) IP/CIDR 校验改进、[5007](https://kep.k8s.io/5007) DRA 设备绑定条件、[5075](https://kep.k8s.io/5075) DRA 可消耗容量、[5419](https://kep.k8s.io/5419) Pod 级资源原地调整、[5905](https://kep.k8s.io/5905) kubernetes-mixin 迁入 kubernetes-sigs、[5922](https://kep.k8s.io/5922) 显式标注未来的一致性测试、[6356](https://kep.k8s.io/6356) 发布 ktesting。

**Beta（28 项）**：[2371](https://kep.k8s.io/2371) 基于 CRI 的容器与 Pod 统计、[2570](https://kep.k8s.io/2570) Memory QoS、[3541](https://kep.k8s.io/3541) StatefulSet Recreate 更新策略、[4671](https://kep.k8s.io/4671) Gang Scheduling、[4885](https://kep.k8s.io/4885) Windows CPU 与内存亲和、[4939](https://kep.k8s.io/4939) gRPC 探针 TLS 凭据、[4960](https://kep.k8s.io/4960) 容器停止信号、[5237](https://kep.k8s.io/5237) CCM 路由控制器改为 watch 驱动、[5491](https://kep.k8s.io/5491) DRA 列表类型属性、[5502](https://kep.k8s.io/5502) emptyDir 支持 stickyBit、[5517](https://kep.k8s.io/5517) DRA 节点可分配资源请求、[5547](https://kep.k8s.io/5547) WAS 接入 Job 控制器、[5710](https://kep.k8s.io/5710) 工作负载感知抢占、[5732](https://kep.k8s.io/5732) 拓扑感知工作负载调度、[5793](https://kep.k8s.io/5793) 基于清单文件的准入控制配置、[5825](https://kep.k8s.io/5825) CRI List 流式接口、[5836](https://kep.k8s.io/5836) 原地调整的调度器抢占、[5866](https://kep.k8s.io/5866) 服务端分片 List/Watch、[5936](https://kep.k8s.io/5936) 原子写入卷的用户字段、[5945](https://kep.k8s.io/5945) DRA 可选节点操作、[5996](https://kep.k8s.io/5996) kubelet 默认 Pod sysctls、[5999](https://kep.k8s.io/5999) 探针支持 h2c、[6012](https://kep.k8s.io/6012) CompositePodGroup API、[6030](https://kep.k8s.io/6030) 内存卷动态调整、[6032](https://kep.k8s.io/6032) nftables localhost NodePort、[6089](https://kep.k8s.io/6089) WAS 控制器接入 API、[6132](https://kep.k8s.io/6132) 调度器 PreQueueingHint、[6404](https://kep.k8s.io/6404) API Server 写吞吐优化。

**Alpha（29 项）**：[4443](https://kep.k8s.io/4443) Job PodFailurePolicy 自定义失败原因、[4563](https://kep.k8s.io/4563) EvictionRequest API、[5359](https://kep.k8s.io/5359) 工作负载控制 swap、[5474](https://kep.k8s.io/5474) 非特权容器可写 cgroup、[5554](https://kep.k8s.io/5554) 静态 CPU 管理策略下的原地调整、[5677](https://kep.k8s.io/5677) DRA 资源可用性可见、[5681](https://kep.k8s.io/5681) 条件授权、[5683](https://kep.k8s.io/5683) 节点生命周期条件、[5690](https://kep.k8s.io/5690) DRA 抢占、[5714](https://kep.k8s.io/5714) 可选的 cgroup namespace 隔离、[5758](https://kep.k8s.io/5758) 容器级 ulimits、[5941](https://kep.k8s.io/5941) DRA 共享可消耗容量、[5972](https://kep.k8s.io/5972) 动态容器、[6007](https://kep.k8s.io/6007) Topology Manager single-numa-node 密度选项、[6011](https://kep.k8s.io/6011) CSI ControllerGetNodeInfo、[6058](https://kep.k8s.io/6058) 卷重建时的 CSI global mount 回退、[6060](https://kep.k8s.io/6060) API Server 到 Webhook 的认证、[6061](https://kep.k8s.io/6061) 基于 OCI 制品分发安全配置、[6063](https://kep.k8s.io/6063) Pod 级 PID 限制、[6252](https://kep.k8s.io/6252) system-reserved/kube-reserved 支持 hugepages、[6264](https://kep.k8s.io/6264) etcd 对象压缩、[6268](https://kep.k8s.io/6268) 慢 watch 流暂停与恢复、[6276](https://kep.k8s.io/6276) WAS 接入 Deployment 控制器、[6283](https://kep.k8s.io/6283) 在平台中使用证书 API、[6313](https://kep.k8s.io/6313) 网络隔离 Pod、[6315](https://kep.k8s.io/6315) kube-apiserver 加权负载均衡、[6361](https://kep.k8s.io/6361) 控制器 Leader Election 恢复、[6395](https://kep.k8s.io/6395) 动态 node-local 临时卷、[6406](https://kep.k8s.io/6406) DRA 扩展资源容量请求。

## 更多信息

v1.38 计划在 12 月 3 日发布 `v1.38.0-rc.0`，12 月 16 日正式发布。等不及的小伙伴可以先用 rc 版本验证上面提到的升级变化。

- [Kubernetes v1.38 Sneak Peek（官方草稿，kubernetes/website#59188）](https://github.com/kubernetes/website/pull/59188)
- [Kubernetes 1.38 Release Highlights Discussion #3121](https://github.com/kubernetes/sig-release/discussions/3121)
- [v1.38 增强跟踪看板](https://github.com/orgs/kubernetes/projects/269/views/1)
- [Kubernetes v1.38 CHANGELOG](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.38.md)
- [Kubernetes v1.38 发布日程](https://github.com/kubernetes/sig-release/blob/master/releases/release-1.38/README.md)

## 如何参与 Kubernetes 社区

- 关注 Bluesky：`@kubernetes.io`
- 参与 [Discuss 社区讨论](https://discuss.kubernetes.io/)
- 加入 [Kubernetes Slack 社区](https://slack.k8s.io/)
- 在 [Stack Overflow](https://stackoverflow.com/questions/tagged/kubernetes) 参与问答
- 通过 [SIG 列表](https://kubernetes.dev/community/community-groups/sigs/) 找到感兴趣的 SIG，或关注 [Release Team](https://github.com/kubernetes/sig-release/tree/master/release-team) 的 shadow 招募
- 在 [Kubernetes 博客](https://kubernetes.io/blog/) 了解更多动态

## 来源

1. Kubernetes v1.38 Sneak Peek（kubernetes/website#59188）<https://github.com/kubernetes/website/pull/59188>
2. Kubernetes 1.38 Release Highlights Contact List #3122 <https://github.com/kubernetes/sig-release/issues/3122>
3. Kubernetes 1.38 Release Highlights Discussion #3121 <https://github.com/kubernetes/sig-release/discussions/3121>
4. Kubernetes v1.38 Release Schedule <https://github.com/kubernetes/sig-release/blob/master/releases/release-1.38/README.md>
5. Enhancements · v1.38 Release Tracking <https://github.com/orgs/kubernetes/projects/269/views/1>
6. Kubernetes v1.38 CHANGELOG <https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.38.md>
7. kubelet: disable cgroupDriver fallback by default（kubernetes/kubernetes#136731）<https://github.com/kubernetes/kubernetes/pull/136731>
8. Kubernetes v1.38 DRA 更新博客草稿（kubernetes/website#59213）<https://github.com/kubernetes/website/pull/59213>
9. Kubernetes Enhancement Proposals <https://kep.k8s.io/>
