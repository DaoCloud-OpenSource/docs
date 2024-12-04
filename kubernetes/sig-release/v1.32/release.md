# Kubernetes 1.32

太平洋时间 2024 年 12 月 11 日，主题为 {release-name} 的 Kubernetes v1.32 正式发布。

此版本距离上版本发布时隔 4 个月，是 2024 年的第三个版本。

与之前的版本类似，Kubernetes v1.32 版本引入了多项新的稳定、Beta 和 Alpha 版本的功能。一贯交付高质量版本的承诺凸显了 Kubernetes 社区的开发周期实力和社区的活跃支持。

此版本包含 44 项改进。在这些改进功能中，有 13 项已晋升为稳定版，另外有 12 项正在进入 Beta 阶段，以及最后的 19 项为 Alpha 阶段。

## 发布主题和 Logo

Kubernetes v1.32 的发布主题是 {release-name}。

![](./kubernetes-1.32.png)

Kubernetes v1.32's {release-story}

## GA 和稳定的功能

GA 全称 General Availability，即正式发布。Kubernetes 的进阶路线通常是 Alpha、Beta、Stable (即 GA)、Deprecation/Removal 这四个阶段。

- [KEP-3221 结构化鉴权配置, 可包含多个 webhook 的鉴权链。该链中的鉴权项可以具有明确定义的参数，这些参数可以按特定顺序验证请求，从而提供细粒度的控制](https://kep.k8s.io/3221)
- [KEP-4193 改进绑定的服务帐户令牌, 自动将 Pod 关联的节点的 name 和 uid （通过 spec.nodeName ）嵌入到生成的令牌中, 并允许用户获取专门与 Node 对象生命周期相关的令牌](https://kep.k8s.io/4193)
- [KEP-4358 自定义资源字段选择器, 允许为自定义资源类型设置预定义的字段选择配置，客户端可以使用以前预设的字段选择器来过滤资源](https://kep.k8s.io/4358)
- [KEP-4420 当生成的名称与现有资源名称冲突时，kube-apiserver 会自动重试使用 generateName 的创建请求，最多重试 7 次](https://kep.k8s.io/4420)
- [KEP-1860 为云提供商提供了一种禁用 kube-proxy 配置 LoadBalancer IP 行为的方法](https://kep.k8s.io/1860)
- [KEP-2681 为 Pod 添加 `status.hostIPs` 字段](https://kep.k8s.io/2681)
- [KEP-4292 在 kubectl debug 命令中的预定义配置文件之上添加了新的自定义配置文件功能, 使kubectl debug pod/node 或临时容器规范可配置](https://kep.k8s.io/4292)
- [KEP-4292 内存管理器是 kubelet 生态系统中的一个新组件，旨在向 QoS 类为 Guaranteed 的 Pod 提供多 NUMA 保证的内存分配](https://kep.k8s.io/1769)
- [KEP-1967 支持调整内存卷的大小以匹配 Pod 可分配内存](https://kep.k8s.io/1967)
- [KEP-3545 优化拓扑管理器中的多 NUMA 对齐，在计算包含多个 NUMA 节点的“最佳提示”时，会考虑 NUMA 节点之间的相对距离](https://kep.k8s.io/3545)
- [KEP-4026 为从 CronJob 派生的 Job 对象添加新注解 “batch.kubernetes.io/cronjob-scheduled-timestamp”](https://kep.k8s.io/4026)
- [KEP-4017 为 StatefulSet 和 Job 创建的 Pod 新增标签，用来标记自身索引](https://kep.k8s.io/4017)
- [KEP-1847 为 StatefulSet 提供了自动删除 PVC 的方法](https://kep.k8s.io/1847)

## 进入 Beta 阶段的功能

Beta 阶段的功能是指那些已经经过 Alpha 阶段的功能, 且在 Beta 阶段中添加了更多的测试和验证, 通常情况下是默认启用的。下文择取了部分特性详述。如果对其他特性感兴趣，请移步至具体的 KEP 页面了解进展和详情。

### KEP-1790 卷扩容失败恢复

用户可以将 PersistentVolumeClaim (PVC) 扩展至底层存储提供程序可能不支持的大小。在这种情况下, 通常扩展控制器会永远尝试扩展卷并不断失败。我们希望让用户更容易从扩展失败中恢复，以便用户可以使用可能成功的值重试卷扩展。为了实现恢复，此增强功能放宽了 PVC 对象的 API 验证，以允许减少请求的大小。

允许用户减小大小的问题是, 恶意用户可以使用此功能来滥用配额系统。他们可以通过快速膨胀然后收缩 PVC 来实现这一点。一般来说，CSI 和树内卷插件都被设计为永远不会对底层卷执行实际收缩，但它可以欺骗配额系统，让其相信用户使用的存储空间比他/她实际使用的存储空间少。为了解决这个问题, 虽然允许用户减少 PVC 的大小（前提是请求的大小 > pvc.Status.Capacity ），但配额计算将使用 max(pvc.Spec.Capacity, pvc.Status.AllocatedResources) 并减少 pvc.Status.AllocatedResources 仅在先前发布的扩展因某种终端故障而失败后由 resize-controller 完成。

### 更新总览

- [KEP-3157 Informer 可以获取数据流来启动缓存](https://kep.k8s.io/3157)
- [KEP-4601 鉴权属性将扩展为包括列表、监视和删除集合中的字段选择器和标签选择器。这将允许鉴权者在做出鉴权决定时使用这些选择器](https://kep.k8s.io/4601)
- [KEP-4633 仅允许对配置的端点进行匿名身份验证](https://kep.k8s.io/4633)
- [KEP-1790 允许用户通过减少请求的容量大小, 从卷扩容失败中恢复](https://kep.k8s.io/1790)
- [KEP-3476 提供了一组新的 API, 支持一致的组快照，该快照允许在同一时间点从多个卷拍摄快照，以实现写入顺序一致性](https://kep.k8s.io/3476)
- [KEP-1710 通过使用正确的 SELinux 标签挂载卷而不是递归更改卷上的每个文件来加速容器启动](https://kep.k8s.io/1710)
- [KEP-4247 调度器通过将 QueueingHint 引入 EventsToRegister，调度队列根据 QueueingHint 的结果对 Pod 重新排队, 有助于减少无用的调度重试，从而提高调度吞吐量](https://kep.k8s.io/4247)
- [KEP-4369 环境变量中允许使用特殊字符](https://kep.k8s.io/4369)
- [KEP-4381 借助结构化参数，kube-scheduler 和 Cluster Autoscaler 可以自行处理和模拟声明分配，而无需依赖第三方驱动程序](https://kep.k8s.io/4381)
- [KEP-4368 支持 Job 的 ManagedBy 字段](https://kep.k8s.io/4368)
- [KEP-3331 添加结构化身份验证配置](https://kep.k8s.io/3331)

## 进入 Alpha 阶段的功能

Alpha 阶段的功能是指那些刚刚被引入的功能，这些功能是默认关闭的，需要用户手动开启。下文择取了部分特性详述, 如果对其他特性感兴趣，请移步至具体的 KEP 页面了解进展和详情。


### KEP-3962 使用 CEL 表达式声明的更改准入策略

该功能利用了 CEL 的对象实例化和 JSON Patch 策略，并结合了 Server Side Apply 的合并算法。它简化了策略定义，减少了变更冲突，并提高了准入控制性能，同时为Kubernetes中更强大、可扩展的策略框架奠定了基础。

Kubernetes 现在支持基于通用表达式语言（CEL）的变更准入策略，提供了一种轻量、高效的替代变更准入 Webhook 的方法。通过这一增强功能，管理员可以使用 CEL 通过简单的声明性表达式来声明变更，例如设置标签、默认字段或注入边车。

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: MutatingAdmissionPolicy
metadata:
  name: "sidecar-policy.example.com"
spec:
  paramKind:
    kind: Sidecar
    apiVersion: mutations.example.com/v1
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE"]
      resources:   ["pods"]
  matchConditions:
    - name: does-not-already-have-sidecar
      expression: "!object.spec.initContainers.exists(ic, ic.name == \"mesh-proxy\")"
  failurePolicy: Fail
  reinvocationPolicy: IfNeeded
  mutations:
    - patchType: "JSONPatch"
      jsonPatch:
        expression: >
          [
            JSONPatch{
              op: "add", path: "/spec/initContainers/-",
              value: Object.spec.initContainers{
                name: "mesh-proxy",
                image: "mesh-proxy/v1.0.0",
                restartPolicy: "Always"
              }
            }
          ]
```

这种方法降低了操作复杂性，消除了对 Webhook 的需求，并直接与 kube-apiserver 集成，提供了更快、更可靠的进程内变更处理能力。

### KEP-283 Pod 级别设置资源请求和限制

此增强功能通过引入在Pod级别设置资源请求和限制的能力，简化了Kubernetes中的资源管理，创建了一个所有 Pod 中的容器都可以动态使用的共享池。这对于具有波动或突发资源需求的工作负载特别有价值，因为它最小化了过度配置并提高了整体资源效率。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-resources-demo
  namespace: pod-resources-example
spec:
  resources:
    limits:
      cpu: "1"
      memory: "200Mi"
    requests:
      cpu: "1"
      memory: "100Mi"
  containers:
  - name: pod-resources-demo-ctr-1
    image: nginx
    resources:
      limits:
        cpu: "0.5"
        memory: "100Mi"
      requests:
        cpu: "0.5"
        memory: "50Mi"
  - name: pod-resources-demo-ctr-2
    image: fedora
    command:
    - sleep
    - inf 
```

通过在Pod级别利用 Linux cgroup 设置，Kubernetes 确保这些资源限制得到执行，同时使紧密耦合的容器能够更有效地协作而不会遇到人为限制。

* 优先级：当同时指定 Pod 级和容器级资源时，Pod 级资源优先。
* QoS：Pod 级资源在影响 Pod 的 QoS 类时优先。
* OOM 分数：OOM 分数调整计算同时考虑 Pod 级和容器级资源。
* 兼容性：Pod 级资源旨在与现有功能兼容。

这标志着多容器 Pod 的显著改进，因为它减少了跨容器管理资源分配的操作复杂性。它还为紧密集成的应用程序（如 sidecar 架构）提供了性能提升，在这些应用程序中，容器共享工作负载或依赖彼此的可用性以实现最佳性能。


### 更多 DRA 增强特性 🔥

DRA 是 Kubernetes 资源管理系统的关键组件，这些增强旨在提高对需要专用硬件（如 GPU、FPGA 和网络适配器） 的工作负载进行资源分配的灵活性和效率。此次发布引入了多项改进，包括在 Pod 状态中添加资源健康状态。

相关 KEP 包括：

- [KEP-4680 将资源运行状况状态添加到设备插件和 DRA 的 Pod 状态](https://kep.k8s.io/4680)
- [KEP-4817 在 ResourceClaim.Status 中添加驱动程序拥有的字段以及可能的标准化网络接口数据](https://kep.k8s.io/4817)

### 更新总览

- [KEP-3962 添加了使用 CEL 表达式声明的更改准入策略，作为更改准入 Webhook 的替代方案](https://kep.k8s.io/3962)
- [KEP-4427 dnsConfig.searches 支持对 DNS 搜索字符串进行宽松验证, 允许包含 “.” 和 “_” 字符](https://kep.k8s.io/4427)
- [KEP-4832 提高调度吞吐量, 当 Pod 需要通过异步 API 调用发出抢占时](https://kep.k8s.io/4832)
- [KEP-3288 拆分容器的 stdout 和 stderr 日志流](https://kep.k8s.io/3288)
- [KEP-2837 允许在 Pod 级别设置资源请求和限制](https://kep.k8s.io/2837)
- [KEP-4680 将资源运行状况状态添加到设备插件和 DRA 的 Pod 状态](https://kep.k8s.io/4680)
- [KEP-4800 引入新的 CPU 管理器静态策略选项，该选项尽最大努力通过 L3（最后一级）缓存来对齐 CPU 资源](https://kep.k8s.io/4800)
- [KEP-4817 在 ResourceClaim.Status 中添加驱动程序拥有的字段以及可能的标准化网络接口数据](https://kep.k8s.io/4817)
- [KEP-4827 为所有核心组件实现 statusz 端点来公开有关组件基本信息、运行状况和关键指标的标准化实时数据](https://kep.k8s.io/4827)
- [KEP-4828 为所有核心组件实现 flagz 端点来公开有关组件的命令行标志的配置参数](https://kep.k8s.io/4828)
- [KEP-2862 为 Node 资源新增若干子资源类型，为 kubelet 提供更细粒度的 API 接口鉴权](https://kep.k8s.io/2862)
- [KEP-4818 PreStop Hook 的睡眠操作允许零值](https://kep.k8s.io/4818)
- [KEP-740  提供服务帐户令牌与外部签名进行集成方法, 易于更新令牌和提供安全性](https://kep.k8s.io/740)
- [KEP-4885 使用 CPUManager、MemoryManager 和拓扑管理器添加对 Windows 节点的 CPU 和内存亲和性支持](https://kep.k8s.io/4885)
- [KEP-3926 引入一个新的 DeleteOption，即使我们无法检索其数据，也可以删除对应的资源](https://kep.k8s.io/3926)
- [KEP-4222 添加 CBOR 数据格式作为 JSON 的有效替代方案](https://kep.k8s.io/4222)
- [KEP-4540 引入了新的 CPUManager 策略选项 strict-cpu-reservation，确保 reservedSystemCPUs 严格保留用于系统守护程序或中断处理，并且不会被 QoS 类为 Burstable 和 BestEffort 的 Pod 使用](https://kep.k8s.io/4540)
- [KEP-4802  初步支持 Windows 节点的体面关闭功能](https://kep.k8s.io/4802)

## 删除和废弃功能

### 撤回 DRA 的旧的实现

增强特性 KEP-3063 ("classic DRA") 在 Kubernetes 1.26 中引入了动态资源分配（DRA）。然而，在 Kubernetes v1.32 中，这种 DRA 的实现方法将发生重大变化。与原来实现相关的代码将被删除，只留下 KEP-4381 作为"新"的基础特性。改变现有方法的决定源于其与集群自动伸缩的不兼容性，因为资源可用性是不透明的，这使得 Cluster Autoscaler 和控制器的决策变得复杂。

新增的结构化参数模型替换了原有特性。这次移除将使 Kubernetes 能够更可预测地处理新的硬件需求和资源声明，避免了与 kube-apiserver 之间复杂的来回 API 调用。

### gitRepo 卷类型弃用说明

[gitRepo](https://kubernetes.io/docs/concepts/storage/volumes/#gitrepo) 卷类型虽然已经被标记为弃用, 但是在现有版本中仍然可以使用。考虑到 `gitRepo` 类型的卷, 存在严重的[安全漏洞](https://nvd.nist.gov/vuln/detail/CVE-2024-10220), 社区计划将在未来的版本中移除相关实现。因此, 在 Kubernetes v1.32 的发布中, 社区再次向用户发出警告, 建议用户尽快进行迁移。

### API 移除

`flowcontrol.apiserver.k8s.io/v1beta3`  版本的 FlowSchema 和 PriorityLevelConfiguration 已被移除。 为了对此做好准备，你可以编辑现有的清单文件并重写客户端软件，使用自 v1.29 起可用的 flowcontrol.apiserver.k8s.io/v1 API 版本。 所有现有的持久化对象都可以通过新 API 访问。flowcontrol.apiserver.k8s.io/v1beta3 中的重要变化包括： 当未指定时，PriorityLevelConfiguration 的 spec.limited.nominalConcurrencyShares 字段仅默认为 30，而显式设置的 0 值不会被更改为此默认值。

## DaoCloud 社区贡献

2024 年 11 月 12 至 15 日期间, [KubeCon + CloudNativeCon North America 2024](https://events.linuxfoundation.org/kubecon-cloudnativecon-north-america/) 如约举办, 以下是本年度我们获得的一些荣誉：

- CNCF 大使 - 蔡威 (再次当选)、要海峰
- CNCF 文档奖 - 要海峰
- Kubernetes Contributor Awards (SIG Storage 和 SIG Cluster Lifecyle) - 范宝发

成长故事分享：[从社区小白到 CNCF 大使](https://mp.weixin.qq.com/s/tdO2QhvE800TKy5RY7FCWw)

## 发行说明

上述内容就是 Kubernetes v1.32 的主要更新和内容啦，更多的发布说明可以查看 Kubernetes v1.32 版本的完整详细信息：`https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.32.md`。

我们下次版本发布时再见！

## 历史文档

- [Kubernetes 1.31 发布！十年 OCI 镜像借着 AI 的风终于加入 Volume 的大家庭 ~](https://mp.weixin.qq.com/s/bl5ozc90PhWMO3l-deiJbw)
- [最可爱的版本 UwU - Kubernetes v1.30 发布！](https://mp.weixin.qq.com/s?__biz=MzA5NTUxNzE4MQ==&mid=2659286459&idx=1&sn=bcb8d232b7b611caf89b7dbf17ce0299&chksm=8bcbfd29bcbc743f88806920a1f5200450deac6575db3d20371f76c54d33140d5f4ce39f19f7)
- [Kubernetes 1.29 全新特性： 抛弃 iptables 还在等什么...](https://mp.weixin.qq.com/s/ZZJBRWauVo-VwNFHkNQ_2w)
- [Kubernetes 1.28 震撼发布，Sidecar Containers 迎面而来](https://mp.weixin.qq.com/s/Dr_JpSD9tzfahslZO2bX5A)
- [近两年功能增加最多！Kubernetes 1.27 正式发布](https://mp.weixin.qq.com/s/maDEiCGzOPSDkH9dUxIxdA)
- [Kubernetes 正式发布 v1.26，稳定性显著提升](https://mp.weixin.qq.com/s/qwzmeIM4INz-_BK_gbwOxw)
- [Kubernetes 1.25 正式发布，多方面重大突破](https://mp.weixin.qq.com/s/aRmLBYpk0MhLJAwY85DyuA)
- [Kubernetes 1.24 走向成熟的 Kubernetes](https://mp.weixin.qq.com/s/vqH8ueaZeEeZbx_axNVSjg)
- [Kubernetes 1.23 正式发布，有哪些增强？](https://mp.weixin.qq.com/s/A5GBv5Yn6tQK_r6_FSyp9A)
- [Kubernetes 1.22 颠覆你的想象：可启用 Swap，推出 PSP 替换方案，还有……](https://mp.weixin.qq.com/s/9nH2UagDm6TkGhEyoYPgpQ)
- [Kubernetes 1.21 震撼发布 | PSP 将被废除，BareMetal 得到增强](https://mp.weixin.qq.com/s/amGjvytJatO-5a7Nz4BYPw)

## 参考

1. Kubernetes 增强特性 <https://kep.k8s.io/>
2. Kubernetes 1.32 发布团队 <https://github.com/kubernetes/sig-release/blob/master/releases/release-1.32>
3. Kubernetes 1.32 变更日志 <https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.32.md>
4. Kubernetes 1.32 主题讨论 <https://github.com/kubernetes/sig-release/discussions/2639>
