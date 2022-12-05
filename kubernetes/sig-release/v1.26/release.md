12 月 8 号 Kubernetes 正式发布了 v1.26，作为 2022 年最后的一个版本，其诸多功能都在稳定性上得到显著提升，并且也增加了很多新的功能，我们将从以下多个角度来介绍 1.26 的更新。
* `Kube APIServer`: 作为 Kubernetes 请求的入口，本次更新增加了 4 个 KEP 新功能，并且在响应压缩上做了一些优化，另外还有 2 个功能也从 Alpha 升级到了 Beta
* `节点`: 我们将 kubelet 关系最为密切的更新放在这里，主要包括 4 个新增的 KEP 功能，并且有 4 个功能在该版本 GA
* `存储`: 存储方面主要是增加了跨命名空间的从快照中分配到卷，并且有 2 个存储相关功能进入了 Beta 阶段，3 个功能正式 GA
* `网络` 网络主要是对 Kube Proxy 的更新，包括一个优化性能的 KEP, 并且有 2 个功能进入 Beta，4 个功能正式 GA
* `资源控制与调协` 主要是针对在 kube-controller-manager 中相关资源 controller 的更新，同样也有 2 个新增的 KEP 功能，有 2 个功能由 Alpha 升级到 Beta，还有 1 个功能正式 GA
* `调度` 方面主要做了一些优化与 Bug Fix，但新增了一个重要 KEP 功能 - *PodSchedulingReadiness*，控制 Pod 何时准备好调度, 有 1 个功能由 Alpha 升级到 Beta
* `可观测性` 在可观测性上主要是增加了新的暴露组件状态的机制 - Component Health SLIs，另外各组件中也增加了很多指标
* `kubectl 命令`，`kubeadm` 以及 `client-go` 也有一些优化和 Bug Fix
* 对于已经 GA 的功能，根据 Kubernetes 的版本迭代策略，我们在 1.26 中也移除了 11 个 Feature Gates，如果在命令中继续设置了这些 Feature Gates, 那么程序就会无法正常启动

在文章的最开始我们先介绍一些比较重要的 API 废弃和对升级有影响的更改
## API 废弃与更改
#### [PR#112306](https://github.com/kubernetes/kubernetes/pull/112306) `flowcontrol.apiserver.k8s.io` 增加 v1beta3 资源，并将 v1beta2 设为最优资源，在 1.27 中将设置 v1beta3 为最优资源

#### [PR#112643](https://github.com/kubernetes/kubernetes/pull/112643) DynamicKubeletConfig 在 1.23 中废弃，kubelet 中的逻辑已经在 1.24 中移除，该 PR 将 Feature Gate `DynamicKubeletConfig` 和 APIServer 内逻辑移除

#### [PR#110618](https://github.com/kubernetes/kubernetes/pull/110618) Kubelet 不再支持 v1alpha2 的 CRI，容器运行时必须实现 v1 版本容器运行时接口

#### [PR#113336](https://github.com/kubernetes/kubernetes/pull/113336) `CSIMigrationvSphere` 功能已经 GA 并且该功能无法被关闭
    官方提示：如果你需要使用 Windows，XFS 或者 raw block 的话，不要升级到 Kubernetes v1.26。直到 vSphere CSI Driver 在 v2.7.x 后的版本中增加相关的支持。

#### [PR#113710](https://github.com/kubernetes/kubernetes/pull/113710) kube-controller-manager 命令中 `--pod-eviction-timeout` flag 被废弃，并且和 `--enable-taint-manager` flag 一起在 1.27 被移除

#### [PR#112120](https://github.com/kubernetes/kubernetes/pull/112120) Kube 组件中删除一些无效的 klog 相关的 flags

##  Kube APIServer
#### [KEP-2799](https://github.com/kubernetes/enhancements/issues/2799) Reduction of Secret-based Service Account Tokens
**新增 Alpha Feature Gate —— `LegacyServiceAccountTokenTracking`** 来控制是否开启该功能

当开启 `LegacyServiceAccountTokenTracking` 时，基于 secret 的 sa token 会使用 label `kubernetes.io/legacy-token-last-used` 来记录最后的使用时间

#### [KEP-3488](https://github.com/kubernetes/enhancements/issues/3488) CEL for Admission Control
相关 PR: [PR#113314](https://github.com/kubernetes/kubernetes/pull/113314), [PR#113349](https://github.com/kubernetes/kubernetes/pull/113349), [PR#112994](https://github.com/kubernetes/kubernetes/pull/112994), [PR#112792](https://github.com/kubernetes/kubernetes/pull/112792), [PR#112926](https://github.com/kubernetes/kubernetes/pull/112926), [PR#112858](https://github.com/kubernetes/kubernetes/pull/112858)

基于 Kubernetes v1.25 的提供的 [KEP-2876 CRD验证表达式语言](https://github.com/kubernetes/enhancements/issues/2876)，该功能提供了一个新的准入控制器 —— ValidatingAdmissionPolicy，允许在不使用 Validation Webhook 时实现字段验证。
```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicy
metadata:
  name: "demo-policy.example.com"
Spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   ["apps"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["deployments"]
  validations:
    - expression: "object.spec.replicas <= 5"
```
这会要求资源的 `spec.replicas` 字段小于等于 5。

**新增 Alpha Feature Gate —— `ValidatingAdmissionPolicy`** 来控制是否开启该功能

#### [KEP-3352](https://github.com/kubernetes/enhancements/issues/3352) Aggregated Discovery [PR#113171](https://github.com/kubernetes/kubernetes/pull/113171)
用户当前获取 Discovery API 只能去遍历请求 Group 和 Version API，而该功能将这些调用减少到只需要请求 */api* 和 */apis* 两个接口即可

**新增 Alpha Feature Gate —— `AggregatedDiscoveryEndpoint`** 来控制是否开启该功能

#### [KEP-3325](https://github.com/kubernetes/enhancements/issues/3325) Auth API to get self user attributes [PR#111333](https://github.com/kubernetes/kubernetes/pull/111333)
在 `authentication/v1alpha1` 组中新增资源 `SelfSubjectReview` 来提供给用户查询自己在 Kubernetes 中映射的用户信息。

并且还增加了 `kubectl alpha auth whoami` 命令来方便查询
```bash
$ kubectl alpha auth whoami -o yaml
apiVersion: authentication.k8s.io/v1alpha1
kind: SelfSubjectReview
status:
  userInfo:
    username: jane.doe
    uid: b79dbf30-0c6a-11ed-861d-0242ac120002
    groups:
    - students
    - teachers
    - system:authenticated
    extra:
      skills:
      - reading
      - learning
      subjects:
      - math
      - sports
```

**新增 Alpha Feature Gate —— `APISelfSubjectAttributesReview`** 来控制是否开启该功能

#### [PR#112193](https://github.com/kubernetes/kubernetes/pull/112193) APIServer 增加 `--aggregator-reject-forwarding-redirect` flag, 用户可以设置为 false, 来继续转发 AA（Aggregated API）Server 的重定向响应，默认为 true

#### [PR#113015](https://github.com/kubernetes/kubernetes/pull/113015) 可以通过 --encryption-provider-config 文件指定自定义资源，这些自定义资源就可以在 etcd 中加密存储

### Response Compression
#### [PR#112299](https://github.com/kubernetes/kubernetes/pull/112299) 根据从数千个生产 k8s 集群收集的负载测试和生产数据，社区观察到 k8s APIServer 中的 gzip 压缩目前不是最优的。
ISSUE: https://github.com/kubernetes/kubernetes/issues/112296

这里有一些[报告](https://docs.google.com/document/d/1rMlYKOVyujboAEG2epxSYdx7eyevC7dypkD_kUlBxn4/edit)和[会议纪要](https://youtu.be/GKBqyV8y8j0)

#### [PR#112309](https://github.com/kubernetes/kubernetes/pull/112309) 在 kubeconfig 增加 `DisableCompression` 字段，设置为 true 时不再设置对响应进行压缩

#### [PR#112580](https://github.com/kubernetes/kubernetes/pull/112580) 在 kubectl 中增加 `DisableCompression` 字段，设置为 true 时不再设置对响应进行压缩

### BUG FIX
#### [PR#112772](https://github.com/kubernetes/kubernetes/pull/112772) 在检查 AA（Aggregated API）服务可用性时，同样不允许 AA（Aggregated API）Server 的重定向

#### [PR#113369](https://github.com/kubernetes/kubernetes/pull/113369) 从 delete 请求返回的对象中的 resourceVersion 与 Watch 时 Delete 事件中包含的 resourceVersion 一致

#### [PR#111866](https://github.com/kubernetes/kubernetes/pull/111866) 在更新自定义资源的状态时强制执行 x-kubernetes-list-type 验证

#### [PR#112526](https://github.com/kubernetes/kubernetes/pull/112526) 修复将来自 AA（Aggregated API）APIServer的 “304 Not Modified” 响应视为内部错误的问题

#### [PR#112979](https://github.com/kubernetes/kubernetes/pull/112979) 修复了使用 APIServer Tracing 时，如果指定的 Egress Selector 没有配置控制平面，APIServer 在启动时会出现 Panic 的问题

### 功能稳定性升级
## Alpha -> Beta
| Feature Gate | KEP |
| --- | --- |
| LegacyServiceAccountTokenNoAutoGeneration | [KEP-2799 Reduction of Secret-based Service Account Tokens](https://github.com/kubernetes/enhancements/issues/2799)
| APIServerIdentity | [KEP-1965 kube-apiserver identity](https://github.com/kubernetes/enhancements/issues/1965)

## 节点（Kubelet）
#### [KEP-3063](https://github.com/kubernetes/enhancements/issues/3063)dynamic resource allocation [PR#111023](https://github.com/kubernetes/kubernetes/pull/111023)
新增 *resource.k8s.io/v1alpha1* 组并且在该 Group 下增加动态资源分配相关的资源 - 'ResourceClaim', 'ResourceClass', 'ResourceClaimTemplate', 'PodScheduling'.

**新增 Alpha Feature Gate —— `DynamicResourceAllocation`** 来控制是否开启该功能

新的 API 比 Kubernetes 现有的设备插件(Device Plugins) 功能更灵活，因为它允许 Pod 请求特殊类型的资源，这些资源可以在节点级、集群级或按照用户设置的其他模式提供。
同样 Pod 结构也增加了相应对动态资源分配的支持
```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: with-resource
    image: busybox
    command: ["sh", "-c", "set && mount && ls -la /dev/"]
    resources:
      claims:
      - name: resource
  resourceClaims:
  - name: resource
    source:
      resourceClaimName: shared-claim
      # resourceClaimTemplateName: test-inline-claim-template
```

#### [KEP-3545](https://github.com/kubernetes/enhancements/issues/3545) Improved multi-numa alignment in Topology Manager [PR#112914](https://github.com/kubernetes/kubernetes/pull/112914)
该功能通过对 `TopologyManager` 的优化来更好的处理 NUMA(Non-Uniform Memory Access) 节点。

在 kubelet config 和 kubectl 命令中增加一个新的可配置项 `topologyManagerPolicyOptions` 字段和 `--topology-manager-policy-options` flag 来设置 Topology Manager Policy 的额外配置

并且**新增三个 Alpha Feature Gate** 来控制对 Topology Manager Policy 的配置
* `TopologyManagerPolicyOptions`
* `TopologyManagerPolicyAlphaOptions`
* `TopologyManagerPolicyBetaOptions`

`TopologyManagerPolicyOptions` 控制是否支持 TopologyManagerPolicyOptions 功能，而另外两个分别用来控制是否可以设置 TopologyManagerPolicy 的 Alpha 和 Beta 级别的 Options

当然使用 TopologyManager 功能还需要开启 `TopologyManager` Feature Gate，不过该 Feature Gate 已经处于 Beta 阶段，并不需要额外设置

#### [KEP-3386](https://github.com/kubernetes/enhancements/issues/3386) Kubelet Evented PLEG for Better Performance [PR#111384](https://github.com/kubernetes/kubernetes/pull/111384)
该功能让 kubelet 在跟踪节点中 Pod 状态时，通过尽可能依赖容器运行时接口(CRI) 的通知来减少定期轮训，这会减少 kubelet 对 CPU 的使用

**新增 Alpha Feature Gate —— `EventedPLEG`** 来控制是否开启该功能

#### [PR#86139](https://github.com/kubernetes/kubernetes/pull/86139) 在过去版本中，在容器 preStop 和 postStart 生命周期回调中使用 httpGet 时，即使将 schemes 设置为 HTTPS，也依然使用 http 来访问，并且设置请求中也不会应用用户设置的 Headers

```yaml
lifecycle:
  postStart:
    port: 443
    httpGet:
      scheme: HTTPS
    httpHeadlers:
    - name: HEADER
      value: VALUE
```
该 PR 修复了这些问题，并且当 https 访问异常时会回退回 http 请求，当发生回退时，会为该 Pod 创建一个 LifecycleHTTPFallback 事件，并且更新 `kubelet_lifecycle_handler_http_fallbacks_total` 指标.

另外**新增了一个 Alpha Feature Gate - `ConsistentHTTPGetHandlers`**，用户可以在 kubelet 中设置 `--feature-gates=ConsistentHTTPGetHandlers=false` 来关闭该回退行为

#### [KEP-3503](https://github.com/kubernetes/enhancements/issues/3503) Windows 中允许指定 Pod 是否加入到节点的网络命名空间 [PR#112961](https://github.com/kubernetes/kubernetes/pull/112961)

**新增 Alpha Feature Gate —— `WindowsHostNetworking`** 来控制是否开启该功能

#### [PR#112414](https://github.com/kubernetes/kubernetes/pull/112414) 允许将 /etc/resolv.conf 中多行选项合并为单行设置到 Pod 中
```
options ndots:1 attempts:3
options ndots:1 attempts:3 ndots:5
```
->
```
options ndots:5 attempts:3
```

### BUG FIX
#### [PR#113041](https://github.com/kubernetes/kubernetes/pull/113041) 修复了执行 kubectl exec 时，由于 lifecycle.preStop 导致 kubelet 由于容器名称重复而选择了错误容器的问题

#### [PR#108832](https://github.com/kubernetes/kubernetes/pull/108832) 当容器设置了 *limit.cpu*，但是 *requests.cpu* 为 "0" 时，cgroups `cpuShares` 取最小值 2，而不是使用 *limit.cpu*

#### [PR#112403](https://github.com/kubernetes/kubernetes/pull/112403) 对于 Kubernetes 上 raw block CSI 卷，对于已经挂载到节点上的卷的每个“映射”(即 raw block “挂载”)操作，kubelet 都错误地调用 CSI NodeStageVolume。此 PR 确保每个节点的每个卷只调用一次。

#### [PR#112184](https://github.com/kubernetes/kubernetes/pull/112184) kubelet 只设置 `--cloud-provider` 或者 `--node-ip` 时，会确保清除节点中无效的 annotation - `alpha.kubernetes.io/provided-node-ip`

#### [PR#113481](https://github.com/kubernetes/kubernetes/pull/113481) 修复 `kubectl logs --timestamps` 查看日志时，出现混乱的随机时间戳的问题

#### [PR#112607](https://github.com/kubernetes/kubernetes/pull/112607) 清理卷挂载时，只需要考虑插件目录而不是整个 kubelet 根目录

#### [PR#112518](https://github.com/kubernetes/kubernetes/pull/112518) 修复了启用 PodDisruptionConditions 特性门控时, Pod 在打了 NoExecute 污点的节点上继续运行的问题

#### [PR#112123](https://github.com/kubernetes/kubernetes/pull/112123) 将 cpuCFSQuotaPeriod 的最小值从 1us 设置为 1ms，设置 1ms 以下的值将会验证失败
相关 PR: [PR#112077](https://github.com/kubernetes/kubernetes/pull/112077), [PR#111520](https://github.com/kubernetes/kubernetes/pull/111520), [PR#63437](https://github.com/kubernetes/kubernetes/pull/63437)

### 功能稳定性升级
#### Beta -> GA
| Feature Gate | KEP |
| --- | --- |
| CPUManager | [KEP-3057 Graduate to CPUManager to GA](https://github.com/kubernetes/enhancements/issues/3570)
| DevicePlugins | [KEP-3573 Graduate DeviceManager to GA](https://github.com/kubernetes/enhancements/issues/3573)
| KubeletCredentialProviders | [KEP-2133 Kubelet Credential Provider](https://github.com/kubernetes/enhancements/issues/2133)
| WindowsHostProcessContainers | [KEP-1981 Support for Windows privileged containers](https://github.com/kubernetes/enhancements/issues/1981)

## 存储
#### [KEP-3294](https://github.com/kubernetes/enhancements/issues/3294) Provision volumes from cross-namespace snapshots
相关 PR: [PR#113186](https://github.com/kubernetes/kubernetes/pull/113186), [PR#kubernetes-csi/external-rpovisioner#805](https://github.com/kubernetes-csi/external-provisioner/pull/805)

在 Kubernetes 1.26 之前，通过 VolumeSnapshot，用户可以从快照中分配卷。但是它无法无法在 PersistentVolumeClaim 中绑定到其他名称空间的 VolumeSnapshots。

该功能支持用户通过设置 PVC 中新增的字段 `.spec.dataSourceRef.namespace` 来跨命名空间从快照中分配卷
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
spec:
  storageClassName: csi-hostpath-sc
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  dataSourceRef:
    apiGroup: snapshot.storage.k8s.io
    kind: VolumeSnapshot
    name: new-snapshot-demo
    namespace: prod
  volumeMode: Filesystem
```

**新增 Alpha Feature Gate —— `CrossNamespaceVolumeDataSource`** 来控制是否开启该功能

### 功能稳定性升级
#### Alpha -> Beta
| Feature Gate | KEP |
| --- | --- |
| RetroactiveDefaultStorageClass | [KEP-3333 Retroactive default StorageClass assignement](https://github.com/kubernetes/enhancements/issues/3333)
| NodeOutOfServiceVolumeDetach | [KEP-2268 Non-graceful node shutdown](https://github.com/kubernetes/enhancements/issues/2268)

##Beta -> GA
| Feature Gate | KEP |
| --- | --- |
| CSIMigrationAzureFile | [KEP-625 In-tree storage plugin to CSI Driver Migration](https://github.com/kubernetes/enhancements/issues/625)
| CSIMigrationvSphere | [KEP-1491 vSphere in-tree to CSI driver migration](https://github.com/kubernetes/enhancements/issues/1491)
| DelegateFSGroupToCSIDriver | [KEP-2317 Allow Kubernetes to supply pod's fsgroup to CSI driver on mount](https://github.com/kubernetes/enhancements/issues/2317)

## 网络
#### [PR#110268](https://github.com/kubernetes/kubernetes/pull/110268) 优化 kube-proxy 性能，它只发送在调用 iptables-restore 中更改的规则，而不是整个规则集
**新增 Alpha Feature Gate —— `MinimizeIPTablesRestore`** 来控制是否开启该功能

#### [PR#108250](https://github.com/kubernetes/kubernetes/pull/108250) kube-proxy 新增 flag `--iptables-localhost-nodeports` 来允许用户禁止通过 localhost 访问 NodePort 的 Services
#### [PR#111806](https://github.com/kubernetes/kubernetes/pull/111806) 如果用户要求使用 ipvs，但是系统没有正确配置时，不再回退到 iptables 模式，而是返回错误
#### [PR#113363](https://github.com/kubernetes/kubernetes/pull/113363) 如果 kube-proxy 检测到分配给 Node 的 `pod.Spec.PodCIDRs` 已经改变，那么它将重新启动
#### [PR#112133](https://github.com/kubernetes/kubernetes/pull/112133) 移除已经废弃的 "userspace" 代理模式

### 功能稳定性升级
#### Alpha -> Beta
| Feature Gate | KEP |
| --- | --- |
| ProxyTerminatingEndpoints | [KEP-1669 Proxy Terminating Endpoints](https://github.com/kubernetes/enhancements/issues/1669)
| ExpandedDNSConfig | [KEP-2595 Expanded DNS configuration](https://github.com/kubernetes/enhancements/issues/2595)

#### Beta -> GA
| Feature Gate | KEP |
| --- | --- |
| MixedProtocolLBService | [KEP-1435 Support of mixed protocols in Services with type=LoadBalancer](https://github.com/kubernetes/enhancements/issues/1435)
| ServiceIPStaticSubrange | [KEP-3070 Reserve Service IP Ranges For Dynamic and Static IP Allocation](https://github.com/kubernetes/enhancements/issues/3070)
| ServiceInternalTrafficPolicy | [KEP-2086 Service Internal Traffic Policy](https://github.com/kubernetes/enhancements/issues/2086)
| EndpointSliceTerminatingCondition | [KEP-1672 Tracking Terminating Endpoints](https://github.com/kubernetes/enhancements/issues/1672)

## 资源控制与调协（kube-controller）
#### [KEP-3017](https://github.com/kubernetes/enhancements/issues/3017) PodHealthyPolicy for PodDisruptionBudget [PR#113375](https://github.com/kubernetes/kubernetes/pull/113375)
在 `PodDisruptionBudget` 资源中新增 `.spec.unhealthyPodEvictionPolicy` 字段来控制不健康的 Pod 何时应该被考虑驱逐。

`unhealthyPodEvictionPolicy` 字段当前可以设置的值有两个 —— `IfHealthyBudget` 和 `AlwaysAllow
```yaml
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: zookeeper
  unhealthyPodEvictionPolicy: IfHealthyBudget
```

**新增 Alpha Feature Gate —— `PDBUnhealthyPodEvictionPolicy`** 来控制是否开启该功能

#### [KEP-3335](https://github.com/kubernetes/enhancements/issues/3335) StatefulSet Start Ordinal [PR#112744](https://github.com/kubernetes/kubernetes/pull/112744)
StatefulSet 当前是从 0 开始对 Pod 进行编号

该功能在 StatefulSet 中增加一个 `.spec.ordinals.start` 字段来控制 Pod 的起始编号
```yaml
apiVersion: apps/v1
kind: StatefulSet
spec:
  ordinals:
    start: 1
```
**新增 Alpha Feature Gate —— `StatefulSetStartOrdinal`** 来控制是否开启该功能

### BUG FIX
#### [PR#112011](https://github.com/kubernetes/kubernetes/pull/112011) 如果多个 HPA 涉及到相同的 Pod，那么会停止工作，并设置 HPA 的 ScalingAction Condition 为 False, Reason 为 AmbiguousSelector, 该 PR 还包括多个 HPA 指向同一 Deployment 的情况。

### 功能稳定性升级
#### Alpha -> Beta
| Feature Gate | KEP |
| --- | --- |
| JobPodFailurePolicy | [KEP-3329 Retriable and non-retriable Pod failures for Jobs](https://github.com/kubernetes/enhancements/issues/3329) |
| PodDisruptionConditions | [KEP-3329 Retriable and non-retriable Pod failures for Jobs](https://github.com/kubernetes/enhancements/issues/3329) |

#### Beta -> GA
| Feature Gate | KEP |
| --- | --- |
| JobTrackingWithFinalizers | [KEP-2307 Job tracking without lingering Pods](https://github.com/kubernetes/enhancements/issues/2307) |

## 资源调度（kube scheduler）
#### [KEP-3521](https://github.com/kubernetes/enhancements/issues/3521) Pod Scheduling Readiness
相关 PR: [PR#113275](https://github.com/kubernetes/kubernetes/pull/113275), [PR#113274](https://github.com/kubernetes/kubernetes/pull/113274), [PR#113442](https://github.com/kubernetes/kubernetes/pull/113442)

并不是 Pending 状态 的 Pod 都已准备好被调度，有些 Pod 会因为缺少必要资源的状态而无法成功调度，而这也会对调度器中带来额外的工作。

该功能在 Pod 中增加 `.spec.schedulingGates` 字段来控制 Pod 是否已经准备好真正调度
```
spec:
  schedulingGates:
  - name: <value>
```

只有在 `spec.schedulingGates` 被清空时，才会开始调度：
```bash
$ kubectl get pod example-po
 NAME         READY   STATUS            RESTARTS   AGE
 example-po   0/1     SchedulingGated   0          30s
```

**新增 Alpha Feature Gate —— `PodSchedulingReadiness`** 来控制是否开启该功能

#### [PR#111726](https://github.com/kubernetes/kubernetes/pull/111726) 在 Scheduler 的调试 Dummper 中输出 Penning 状态的 Pod 信息

### 优化与 BUG FIX
#### [PR#111809](https://github.com/kubernetes/kubernetes/pull/111809) 在 Patch 更新 Pod 状态时，除了 net.ConnectionRefused 外，遇到 ServiceUnavailable 和 InternalError 错误时也会进行重试
在 APIServer 暂时无法处理请求时，会出现 ServiceUnavailable 错误
> E0805 20:54:21.624945  123623 scheduler.go:356] Error updating pod foo: the server is currently unable to handle the request (patch pods foo)

InternalError 通常会由于 webhook 的临时故障而出现
> E0811 23:32:30.886582  213747 scheduler.go:357] Error updating pod foo: Internal error occurred: failed calling webhook "xyz": Post "xyz": context deadline exceeded

#### [PR#112357](https://github.com/kubernetes/kubernetes/pull/112357) 在 PodTopologySpread 插件添加与 TaintToleration 插件一致的污点过滤逻辑

#### [PR#112507](https://github.com/kubernetes/kubernetes/pull/112507) 修复了 podTopologySpread 插件中的计算错误，以避免意外的调度结果。

#### [PR#111999](https://github.com/kubernetes/kubernetes/pull/111999) Pod 由于预期的错误而导致调度失败时，将 reason 更新为 `SchedulerError` 而不是 `Unschedulable`

#### [PR#112257](https://github.com/kubernetes/kubernetes/pull/112257) 增加 v1beta3 版本 Scheduler Config 的废弃日志

### 功能稳定性升级
#### Alpha -> Beta
| Feature Gate | KEP |
| --- | --- |
| NodeInclusionPolicy | [KEP-3094 Take taints/tolerations into consideration when calculating PodTopologySpread skew](https://github.com/kubernetes/enhancements/issues/3094)

## 可观测性
#### [KEP-3466](https://github.com/kubernetes/enhancements/issues/3466) Kubernetes Component Health SLIs
之前并没有标准的格式来暴露 Kubernetes 组件的健康信息，该功能在每个组件中增加了一个新的路径 */metrics/slis*，以 Prometheus 格式来公开服务级别指标(Service Level Indicator).

每个组件都需要暴露两个 metrics:
* **gauge**: 组件当前的健康检查状态
```
# HELP kubernetes_healthcheck [ALPHA] This metric records the result of a single healthcheck.
# TYPE kubernetes_healthcheck gauge
kubernetes_healthcheck{name="etcd",type="healthz"} 1
kubernetes_healthcheck{name="etcd",type="readyz"} 1
```
* **counter**: 对每个健康检查检测到的状态的累积计数
```
# HELP kubernetes_healthchecks_total [ALPHA] This metric records the results of all healthcheck.
# TYPE kubernetes_healthchecks_total counter
kubernetes_healthchecks_total{name="etcd",status="success",type="healthz"} 15
kubernetes_healthchecks_total{name="etcd",status="success",type="readyz"} 15
```

* Kube APIServer: [PR#112741](https://github.com/kubernetes/kubernetes/pull/112741)
* Kube Controller Manager: [PR#112978](https://github.com/kubernetes/kubernetes/pull/112978)
* Kube Scheduler: [PR#113026](https://github.com/kubernetes/kubernetes/pull/113026)
* Kubelet: [PR#113030](https://github.com/kubernetes/kubernetes/pull/113030)
* Kube Proxy: [PR#113057](https://github.com/kubernetes/kubernetes/pull/113057)
* Cloud Controller Manager: [PR#113340](https://github.com/kubernetes/kubernetes/pull/113340)

**新增 Alpha Feature Gate —— `ComponentSLIs`** 来控制是否开启该功能

#### 各组件中增加了一些 metrics 指标，并修复了一些指标计算的问题


## Kubectl 命令
### 子命令增强和修复
#### [PR#109525](https://github.com/kubernetes/kubernetes/pull/109525) `kubectl wait` 命令支持在 *-o jsonpath=* 中设置不存在的字段，这在某些字段被异步设置时会很有用
#### [PR#111096](https://github.com/kubernetes/kubernetes/pull/111096) `kubectl api-resources` 在 *-o wide* 输出时增加 categories 列，并且增加 *--categories* 参数来支持根据 categories 过滤
#### [PR#113819](https://github.com/kubernetes/kubernetes/pull/113819) `kubectl alpha event` 移动为顶级命令 `kubectl events`
#### [PR#111093](https://github.com/kubernetes/kubernetes/pull/111093) 修复了 `kubectl rollout history --revision=<version> -o json|yaml <resource>` 来输出 json/yaml 时,返回最新版本而不是指定的 revision.

#### [PR#111571](https://github.com/kubernetes/kubernetes/pull/111571) 优化 `kubectl label --dry-run` 命令的提示信息，避免用户误解为 label 已被设置
before
```bash
$ kubectl label pod foo bar=baz --dry-run=server
pod/foo labeled
```
after
```bash
$ kubectl label pod foo bar=baz --dry-run=server
pod/foo labeled (server dry run)
```

#### [PR#112556](https://github.com/kubernetes/kubernetes/pull/112556) 优化 `kubectl patch` 使用 StrategicMerigePatch 更新自定义资源时的错误信息
#### [PR#112700](https://github.com/kubernetes/kubernetes/pull/112700) 修复 `kubectl covert` 选择了错误的 api 版本的问题
#### [PR#109505](https://github.com/kubernetes/kubernetes/pull/109505) `kubectl annotate` 设置与原值相同值的 annotation 时，不再抛出错误
#### [PR#110907](https://github.com/kubernetes/kubernetes/pull/110907) 执行 kubectl apply 时，如果指定了 `--namespace`，但是没有指定 `--prune-allowlist`，会删除非命名空间的资源，该 pr 只是增加打印一个警告，在 1.28 中 kubectl apply 在指定 namespace 时，不再删除无命名空间的资源 pv & namespace
#### [PR#113116](https://github.com/kubernetes/kubernetes/pull/113116) `kubectl apply` 增加 `--prune-allowlist` flag, 配合 `--prune` flag 使用，替代已废弃的 `--prune-whitelist` flag

### Other
#### [PR#113146](https://github.com/kubernetes/kubernetes/pull/113146) `kubectl explain` 命令可以通过环境变量 `KUBECTL_EXPLAIN_OPENAPIV3` 来使用 OpenAPIv3
#### [PR#112553](https://github.com/kubernetes/kubernetes/pull/112553) Kubectl 转义输出中的终端特殊字符。修复 CVE-2021-25743。
#### [PR#112150](https://github.com/kubernetes/kubernetes/pull/112150) 优化 kubectl 对 APIServer 返回的无效请求的显示
#### [PR#112243](https://github.com/kubernetes/kubernetes/pull/112243), [PR#112261](https://github.com/kubernetes/kubernetes/pull/112261) 废弃 `kubectl run` 命令的多个 flags，即使设置了他们也会被忽略

### Shell 补全
#### [PR#113636](https://github.com/kubernetes/kubernetes/pull/113636) kubectl shell 补全支持在 bash 中显示命令描述
```bash
bash-5.1$ kubectl a[tab][tab]
alpha          (Commands for features in alpha)
annotate       (Update the annotations on a resource)
api-resources  (Print the supported API resources on the server)
api-versions   (Print the supported API versions on the server, in the form of "group/version")
apply          (Apply a configuration to a resource by file name or stdin)
argo           (The command argo is a plugin installed by the user)
attach         (Attach to a running container)
auth           (Inspect authorization)
autoscale      (Auto-scale a deployment, replica set, stateful set, or replication controller)

bash-5.1$ kubectl --c[tab][tab]
--cache-dir              (Default cache directory)
--certificate-authority  (Path to a cert file for the certificate authority)
--client-certificate     (Path to a client certificate file for TLS)
--client-key             (Path to a client key file for TLS)
--cluster                (The name of the kubeconfig cluster to use)
--context                (The name of the kubeconfig context to use)
```

#### [PR#105867](https://github.com/kubernetes/kubernetes/pull/105867) 提供对 kubectl 插件的 shell 补全，插件可以通过 `kube_complete-<pluginName>` 来提供插件命令的 shell 补全

## Kubeadm
### 命令修复和增强
#### [PR#113005](https://github.com/kubernetes/kubernetes/pull/113005) `kubeadm join phase control-plane-preapare certs` 支持使用 `--dry-run` 运行
#### [PR#112945](https://github.com/kubernetes/kubernetes/pull/112945) 支持子阶段的 dry-run 模式，例如 `kubeadm reset phase cleanup-node --dry-run`
#### [PR#111512](https://github.com/kubernetes/kubernetes/pull/111512) `kubeadm init` 命令中增加一个新 phase —— `show-join-command`, 用户可以通过 `kubeadm init --skip-phase=show-join-command` 来跳过打印 join 信息. 该阶段无法单独执行
#### [PR#112172](https://github.com/kubernetes/kubernetes/pull/112172) `kubeadm reset` 命令增加 `--cleanup-tmp-dir` flag, 它会清理 /etc/kubernetes/tmp 中的内容，默认为 false
#### [PR#112732](https://github.com/kubernetes/kubernetes/pull/112732) kubeadm 增加对配置中镜像仓库格式的校验
#### [PR#111783](https://github.com/kubernetes/kubernetes/pull/111783) kubeadm 读取的 kubeconfig 的 CertificateAuthorityData 为空时，会尝试从外部 CertificateAuthority 文件加载 CA 证书
#### [PR#112508](https://github.com/kubernetes/kubernetes/pull/112508) 在 preflight check 时允许 RSA 和 ECDSA 格式的 key
#### [PR#110972](https://github.com/kubernetes/kubernetes/pull/110972) `kubeadm reset` 在执行时会尽可能的去清理掉陈旧的数据，旧数据会在每个 reset 阶段执行时被清除，默认的 etcd 数据目录会在 remove-etcd-member 阶段执行时被删除
#### [PR#112751](https://github.com/kubernetes/kubernetes/pull/112751) 修复验证 ClusterConfiguration 网络相关字段(dnsDomain, serviceSubnet, podSubnet) 时的bug

### Other
#### [PR#111277](https://github.com/kubernetes/kubernetes/pull/111277) 优化 kubeadm 在运行子命令时的错误提示信息
#### [PR#112008](https://github.com/kubernetes/kubernetes/pull/112008) 由于 1.25 中不再将 *node-role.kubernetes.io/master* 污点设置在控制面节点中，kubeadm 不再为 CoreDNS Deployment 设置 `node-role.kubernetes.io/master` 容忍.
#### [PR#112000](https://github.com/kubernetes/kubernetes/pull/112000) kubeadm init|join|upgrade 命令删除 `--container-runtime` flag，因为自从 dockershim 被移除后，该 flag 仅有一个可以设置的值 `--container-runtime=remote`

## Client-Go
#### [PR#112200](https://github.com/kubernetes/kubernetes/pull/112200) client-go 的 SharedInformerFactory 增加 Shutdown 方法，来等待 Factory 内所有运行的 informer 都结束

## 移除已经 GA 的功能 Feature Gate
* ServiceLoadBalancerClass
* ServiceLBNodePortControl
* CSRDuration
* DefaultPodTopologySpread
* NonPreemptingPriority
* PodAffinityNamespaceSelector
* PreferNominatedNode
* PodOverhead
* UnversionedKubeletConfigMap
* IndexedJob
* SuspendJob

## 功能降级
#### Beta -> Alpha
`LocalStorageCapacityIsolationFSQuotaMonitoring` 在 v1.25 中升级到 Beta，但是由于 ConfigMap 更新后不会正常同步到 Pod 文件系统的问题而被回退到 Alpha
