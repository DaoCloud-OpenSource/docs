太平洋时间 2023 年 4 月 11 日，Kubernetes 1.27 正式发布。此版本距离上版本发布时隔 4 个月，是 2023 年的第一个版本。

新版本包含 x 个 enhancements「1.25 版本  个、1.26 版本  个」其中  个功能升级为稳定版， 个已有功能进行优化，另有多达  个全新功能以及  个废弃的功能。

Kubernetes 1.27 版本在多个方面实现重大突破，需要注意的是 详情可以参考本文第五小节「升级注意事项」。

本版本包含了多个 的功能，比如
相关的重要功能会在下一小节进行详细介绍。

### 1. 重要功能

Candidates:(waiting for major thems update from sig-release team)

1. openapi v3
2. multiple sevice CIDR
3. container resource based pod autoscaling: Graduate the container resource metrics feature on HPA to beta.
4. CRD validation expression language
5. CEL-based admission webhook match condtions
6. KMS v2: The API server now re-uses data encryption keys while the kms v2 plugin's key ID is stable. Update KMSv2 to beta. CacheSize field in EncryptionConfiguration is not supported for KMSv2 provider.
7. subresource support to kubectl
8. PDB and PodHealthyPolicy: The PodDisruptionBudget spec.unhealthyPodEvictionPolicy field has graduated to beta and is enabled by default.
9. Elastic Indexed Job
10. Aggregated Discovery
11. ValidatingAdmissionPolicy update
12. A new feature has been enabled to improve the performance of the iptables mode of kube-proxy in large clusters.
If you experience problems with Services not syncing to iptables correctly, you can disable the feature by passing --feature-gates=MinimizeIPTablesRestore=false to kube-proxy (and file a bug if this fixes it). (This might also be detected by seeing the value of kube-proxy's sync_proxy_rules_iptables_partial_restore_failures_total metric rising.)

#### [Alpha] VPA In-Place Update of Pod resources

In-place resize feature for Kubernetes Pods

- Changed the Pod API so that the resources defined for containers are mutable for cpu and memory resource types.
- Added resizePolicy for containers in a pod to allow users control over how their containers are resized.
- Added allocatedResources field to container status in pod status that describes the node resources allocated to a pod.
- Added resources field to container status that reports actual resources applied to running containers.
- Added resize field to pod status that describes the state of a requested pod resize.

#### Auto remove PVCs created by StatefulSet

StatefulSetAutoDeletePVC feature gate promoted to beta.

#### OTEL: APIServer Tracing

APIServerTracing 升级为 Beta，默认开启。Tracing in the API Server is still disabled by default, and requires a config file to enable

#### [Alpha] Kubelet Evented PLEG to Beta(but still disabled by default)

在节点 Pod 较多的情况下，通过容器运行时的 Event 驱动 Pod 状态更新，能够有效的提升效率。
Graduate CRI Events driven Pod LifeCycle Event Generator (Evented PLEG) to Beta

#### [Alpha] dynamic resource allocation

该功能是一种实现，而 storage 相关 iops 限制又有了自己的另一种实现。

#### volume group snapshot

### 2. 其他需要了解的功能

- [node] User Namespaces for statefulset pods: 用户命名空间功能仍然是 alpha，相比 v1.26 支持 StatefulSet。
- [node] GRPC probes 功能 GA。
- [apps] Cronjob 支持 Timezone 功能 GA。
- [auth] Added a new alpha API: ClusterTrustBundle (certificates.k8s.io/v1alpha1).
- [auth] The AdmissionWebhookMatchConditions featuregate is now in Alpha.
- [node & instrumentation] Adds feature gate NodeLogQuery which provides cluster administrators with a streaming view of logs using kubectl without them having to implement a client side reader or logging into the node.
- [node] Bump default API QPS limits for Kubelet.
- [apps] Enable the "StatefulSetStartOrdinal" feature gate in beta.
- [node] Graduate Kubelet Topology Manager to GA.
- [api-machinery] Promoted SelfSubjectReview to Beta
- [storage] SELinuxMountReadWriteOncePod graduated to Beta.
- [storage] Graduates the CSINodeExpandSecret feature to Beta.
- [scheduling] New "plugin_evaluation_total" is added to the scheduler.
- [node] Graduated seccomp profile defaulting to GA.
- [scheduling] PodSchedulingReadiness is graduated to beta.
- [apps] The DownwardAPIHugePages kubelet feature graduated to stable / GA.
- [scheduling] Graduate the ReadWriteOncePod feature gate to beta
- /metrics/slis is made available for control plane components allowing you to scrape health check metrics.
- [network] New feature gate, ServiceNodePortStaticSubrange, to enable the new strategy in the NodePort Service port allocators, so the node port range is subdivided and dynamic allocated NodePort port for Services are allocated preferentially from the upper range.
- [network] Added warnings about workload resources (Pods, ReplicaSets, Deployments, Jobs, CronJobs, or ReplicationControllers) whose names are not valid DNS labels.
- Kubernetes components that perform leader election now only support using Leases for this.
- Graduated the LegacyServiceAccountTokenTracking feature gate to Beta.

### 3. DaoCloud 参与功能

本次发布中， DaoCloud 重点贡献了 sig-node，sig-scheduling 和 kubeadm 相关内容，具体功能点如下：

- [client-go] 修复尝试获取 leader lease 的等待时间的问题。
- [node] 如果 Pod 的 spec.terminationGracePeriodSeconds 属性值是负数，则会被视为设定了1秒的 terminationGracePeriodSeconds。
- [node] 添加了一个可以限制节点进行并行镜像下载数量的新功能。
- [node] 改进目前 Memory QoS 功能，优化了其在 cgroup v2 场景的适配性。
- [scheduling] MinDomainsInPodTopologySpread 功能升级为 Beta。
- [scheduling] 当任何调度程序插件在 PostFilter 中返回 unschedulableAndUnresolvable 状态时，该 Pod 的调度周期立即终止。
- [instrumentation] 迁移控制器使用 contextual logging。
- [node] Kubelet：将“--container-runtime-endpoint”和“--image-service-endpoint”迁移到kubelet配置中。
- [kubeadm] Kubeadm: 添加特性门控 EtcdLearnerMode，它允许将新增的控制节点的 etcd 作为学习者 Learner 加入，然后再升级为投票成员。
- [node] Kubelet 默认允许 Pod 使用 net.ipv4.ip_local_reserved_ports sysctl，要求内核版本 3.16+。

在 v1.27 发布过程中，DaoCloud 参与上百个问题修复和功能研发，作为作者约有 90 个提交，详情请见[贡献列表](https://www.stackalytics.io/cncf?project_type=cncf-group&release=all&metric=commits&module=github.com/kubernetes/kubernetes&date=118)（该版本的两百多位贡献者中有来自 DaoCloud 的 15 位）。在 Kubernetes v1.27 的发布周期中，DaoCloud 的多名研发工程师取得了不少成就。其中，由张世明主要维护的项目 KWOK (Kubernetes Without Kubelet) 成为社区热点，并在大规模集群模拟方面有效地节约资源，提升效率。几位研发人员参与了 Kubernetes 官网的大量中文翻译工作，其中要海峰几乎包揽了近期官网博客的翻译，并成为 SIG-doc-zh 的维护者。此外，刘梦姣也是 SIG-doc 的维护者。在即将召开的 2023 年欧洲 KubeCon 上，殷纳将以调度方向的维护者身份作为讲师，分享调度方向的深度剖析；徐俊杰将分享 kubeadm 的深度解析。

### 4. 版本标志

本次发布的主题是 。 Kubernetes 是。
![logo](kubernetes-1.27.png)

### 5. 升级注意事项

本节主要介绍 v1.27 中 API 变化，以及功能的移除以及废弃，废弃的功能通常会在 1-2 个版本之后移除。更多详情请查看[Kubernetes 在 v1.27 中移除的特性和主要变更](https://kubernetes.io/zh-cn/blog/2023/03/17/upcoming-changes-in-kubernetes-v1-27/)

其中需要重点关注的，IPv6DualStack 外部云供应商特性门控已被删除。（该功能在1.23版本中成为GA，几个版本之前已删除所有其他组件的特性门控。）如果您仍然手动启用它，则必须立即停止。

#### k8s.gcr.io 重定向到 registry.k8s.io 相关说明

Kubernetes 项目为了托管其容器镜像，使用社区拥有的一个名为 registry.k8s.io. 的镜像仓库。 从 3 月 20 日起，所有来自过期 k8s.gcr.io 仓库的流量将被重定向到 registry.k8s.io。 已弃用的 k8s.gcr.io 仓库最终将被淘汰。

#### 其他需要注意的变化

- CSIStorageCapacity 的 storage.k8s.io/v1beta1 API 版本在 v1.24 中已被弃用，将在 v1.27 中被移除。
- 移除 NetworkPolicyEndPort、LocalStorageCapacityIsolation、StatefulSetMinReadySeconds、IdentifyPodOS、DaemonSetUpdateSurge、EphemeralContainers、CSIInlineVolume、CSIMigration、ControllerManagerLeaderMigration 特性门控，这些特性大部分都是在 v1.25 之前的版本正式 GA。
- kube-apiserver 移除了 --master-service-namespace 命令行参数
- kube-controller-manager 命令行参数 --enable-taint-manager 和 --pod-eviction-timeout 被移除。
- kubelet 移除了命令行参数 --container-runtime，该参数目前只有一个可选值 "remote" 并在之前版本中废弃。
- 弃用了 Alpha 状态的 seccomp 注解 seccomp.security.alpha.kubernetes.io/pod 和 container.seccomp.security.alpha.kubernetes.io。
- SecurityContextDeny 特性门控已经废弃，将在未来版本移除。
- Linux/arm will not ship in Kubernetes 1.27 as we are running into issues with building artifacts using golang 1.20.2

### 6. 历史文档

- [Kubernetes 正式发布 v1.26，稳定性显著提升](https://mp.weixin.qq.com/s/qwzmeIM4INz-_BK_gbwOxw)
- [Kubernetes 1.25 正式发布，多方面重大突破](https://mp.weixin.qq.com/s/aRmLBYpk0MhLJAwY85DyuA)
- [Kubernetes 1.24 走向成熟的 Kubernetes](https://mp.weixin.qq.com/s/vqH8ueaZeEeZbx_axNVSjg)
- [Kubernetes 1.23 正式发布，有哪些增强？](https://mp.weixin.qq.com/s/A5GBv5Yn6tQK_r6_FSyp9A)
- [Kubernetes 1.22 颠覆你的想象：可启用 Swap，推出 PSP 替换方案，还有……](https://mp.weixin.qq.com/s/9nH2UagDm6TkGhEyoYPgpQ)
- [Kubernetes 1.21 震撼发布 | PSP 将被废除，BareMetal 得到增强](https://mp.weixin.qq.com/s/amGjvytJatO-5a7Nz4BYPw)

### 7. 参考

1. <https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.27.md>
2. <https://kubernetes.io/zh-cn/blog/2023/03/17/upcoming-changes-in-kubernetes-v1-27/>
3. https://github.com/kubernetes-sigs/kwok/
4. 