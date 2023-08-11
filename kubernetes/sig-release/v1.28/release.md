# Kubernetes 1.28 发布日志

太平洋时间 2023 年 8 月 15 日，Kubernetes 1.28 正式发布。此版本距离上版本发布时隔 4 个月，是 2023 年的第二个版本。

新版本中 release 团队跟踪了 46 个 enhancements。其中 12 个功能升级为稳定版，14 个已有功能进行优化升级为 Beta，另有 20 个 Alpha 级别的功能，大多数为全新功能。1.28 版本包含了很多重要功能以及用户体验优化，接下来将详细介绍部分重要功能。

## 1. 重要功能

### [架构] 版本偏差策略扩展至 3 个版本

从 1.28 控制平面/1.25 节点开始，Kubernetes 版本偏差策略将支持的控制平面/节点偏差扩展到 3 个版本。这使得节点的年度次要版本升级成为可能，同时保持受支持的次要版本。更多信息，请访问 https://kubernetes.io/releases/version-skew-policy/

### [API Machinery] KEP-2876: CRD 基于通用表达式语言 (CEL) 的验证规则 Beta

CRD 基于通用表达式语言 (CEL) 的验证规则，在 v1.25 中就已经升级为 Beta。通过将 CEL 表达式直接集成在 CRD 中，可以是开发者在不使用 Webhook 的情况下解决大部分对 CR 实例进行验证的用例。在未来的版本，我们将继续扩展 CEL 表达式的功能，以支持默认值和 CRD 转换。

此特性使用 x-kubernetes-validations 扩展包含在 CustomResourceDefinition 模式定义中。在 v1.27 中，ValidationRule 新增了可选字段 messageExpression 提供可读性的失败信息。在 v1.28 中添加可选字段“reason”和“fieldPath”，允许用户在验证失败时指定原因和字段路径。

```yaml
x-kubernetes-validations:
- rule: "self.x <= self.maxLimit"
  messageExpression: '"x exceeded max limit of " + string(self.maxLimit)'
  reason: "xExceededMaxLimit"
  fieldPath: "spec.x"
```

### [API Machinery] KEP-3488: 基于通用表达式语言 (CEL) 的准入控制 Beta

基于通用表达式语言 (CEL) 的准入控制是可定制的，对于 kube-apiserver 接受到的请求，可以使用 CEL 表达式来决定是否接受或拒绝请求，可作为 Webhook 准入控制的一种替代方案。在 v1.28 中，CEL 准入控制被升级为 Beta，同时添加了一些新功能，包括但不限于：

1. `ValidatingAdmissionPolicy` 类型检查现在可以正确处理 CEL 表达式中的 “authorizer” 变量。
2. `ValidatingAdmissionPolicy` 支持对 `messageExpression` 字段进行类型检查。
3. 在接受单个请求的过程中，`ValidatingAdmissionPolicy` 准入插件将对每个唯一的授权者表达式执行不超过一次的授权检查。 对相同授权者表达式的所有评估将产生相同的决策。

   ```yaml
   # 例子
   expression: authorizer.path('/healthz').check('GET').allowed()
   messageExpression: authorizer.path('/healthz').check('GET').reason()
  ```

4. kube-controller-manager 组件新增 `ValidatingAdmissionPolicy` 控制器，用来对 `ValidatingAdmissionPolicy` 中的 CEL 表达式做类型检查，并将原因保存在状态字段中。
5. 支持需要权限检查的准入控制用例，通过使用 “authorizer” 变量及其方法来，可以实现以下场景 (包括但不限于):
    - 验证只有具有特定权限的用户才能设置特定字段。
    - 验证只有负责 finalizer 的控制器才能将其从 finalizers 字段中删除。
6. 支持变量组合，可以在 `ValidatingAdmissionPolicy` 中定义变量，然后在定义其他变量时中使用它。
7. 新增 CEL 库函数支持对 Kubernetes 的 resource.Quantity 类型进行解析。

### [API Machinery] KEP-3488: 基于通用表达式语言 (CEL) 的准入 Webhook 匹配条件 Beta

将 CEL 表达式过滤器引入 Webhook，以允许请求打向 Webhook 的范围变得更小。向准入 Webhook 添加“匹配条件”，作为定义 Webhook 范围的现有规则的扩展。 matchCondition 是一个 CEL 表达式，必须评估为 true 才能将准入请求发送到 Webhook。 如果 matchCondition 的计算结果为 false，则将跳过该请求的 Webhook（隐式允许）。

`ValidatingAdmissionPolicy` 是一项令人兴奋的新功能，我们希望它能够大大减少对准入 Webhook 的需求，但它并不是有意尝试涵盖所有可能的用例，而是旨在改善那些无法迁移的 webhooks 的情况。

### [API Machinery] KEP-4020: Kube APIServer 混合版本互操作代理 Alpha

当集群具有多个混合版本的 apiserver 组件时（例如在升级/降级期间或运行时配置更改和部署回滚发生时），并非每个 apiserver 都可以为每个版本的每个资源提供服务。

为了解决这个问题，过滤器被添加到聚合器的处理程序链中，聚合器将客户端代理到能够处理其请求的 apiserver组件。

当在集群上执行升级或降级时，在一段时间内，apiserver 处于不同的版本，并且能够提供不同的内置资源集（不同的组、版本和资源都是可能的）。

### [API Machinery] KEP-4008: CRD Validation Ratcheting Alpha

如果 Patch 请求没有更改任何不合法的字段，这将允许 CR 验证失败。将验证逻辑从控制器转移到前端的能力，是提高 Kubernetes 项目可用性的长期目标。

### [API Machinery] KEP-4080: 新增通用控制平面存储库 Alpha

使 kube-apiserver 构建在一个新的临时存储库上，该存储库使用 k8s.io/apiserver，但具有更大、精心选择的 kube-apiserver 功能子集，以便可重用。

这个过程是渐进的：我们将从一个新的存储库开始，它不会向 k8s.io/apiserver 添加任何内容，然后逐步将通用功能从 kube-apiserver 移动到它下面。新的存储库被命名为 k8s.io/generic-controlplane。

### [节点] KEP-4009: 设备插件 API 添加对 CDI 标准设备的支持 Alpha

CDI 提供了一种将复杂设备注入容器的标准化方法（即逻辑上需要注入多个 /dev 节点才能工作的设备）。这项新功能使插件开发人员能够利用 1.27 中添加到 CRI 的 CDIDevices 字段将 CDI 设备直接传递到支持 CDI 的运行时（其中 containerd 和 crio-o 是最新版本）。

有关 CDI 本身的更多信息，请访问 https://github.com/container-orchestrated-devices/container-device-interface

### [节点] KEP-753: 原生支持 Sidecar 容器 Alpha

它为 init 容器引入了 restartPolicy 字段，并使用这个字段来指示 init 容器是 sidecar 容器。 Kubelet 将按照 restartPolicy=Always 的顺序与其他 init 容器一起启动 init 容器，但它不会等待其完成，而是等待容器启动完成。

启动完成的条件是启动探测成功（或者未定义启动探测）并且 postStart 处理程序完成。 此条件用 ContainerStatus 类型的字段 Started 表示。 有关选择此信号的注意事项，请参阅 “Pod 启动完成条件” 部分。

字段 restartPolicy 仅在 init 容器上被接受。现在唯一支持的值是 “Always”。不会定义其他值。此外，该字段可为空，因此默认值为“无值”。容器的 restartPolicy 的其他值将不被接受，容器将遵循当前实现的逻辑。

Sidecar 容器不会阻止 Pod 完成 - 如果所有常规容器都已完成，Sidecar 容器将被终止。 在 sidecar 启动阶段，重启行为将类似于 init 容器。如果 Pod restartPolicy 为 Never，则启动期间失败的 sidecar 容器将不会重新启动，整个 Pod 将失败。如果 Pod restartPolicy 为 Always 或 OnFailure，则会重新启动。一旦 sidecar 容器启动（postStart 完成且启动探测成功），即使 Pod restartPolicy 为 Never 或 OnFailure，这些容器也会重新启动。此外，即使在 Pod 终止期间，sidecar 容器也会重新启动。

为了最大限度地减少 sidecar 容器的 OOM 杀死，这些容器的 OOM 调整将匹配或超过 Pod 中常规容器的 OOM 分数调整。 

其他 init 容器不同，Sidecar 容器还支持以下配置：

- Sidecar 容器的 PostStart 和 PreStop 生命周期处理程序
- 所有探针（启动、就绪、活跃）

Sidecar 的就绪探测将有助于确定整个 Pod 的就绪情况。

### [节点] KEP-2400: 节点 Swap 内存支持 Beta

在 Kubernetes 1.22 之前，节点不支持使用内存交换，如果在节点上检测到交换，kubelet 将默认无法启动。每个系统管理员或 Kubernetes 用户在设置和使用 Kubernetes 方面都采用同一种方式：禁用交换空间。

从 1.22 开始，可以在每个节点上启用内存交换 Alpha 功能支持。此更改允许管理员选择在 Linux 节点上配置交换，将块存储的一部分视为额外的虚拟内存。要在节点上启用交换，必须在 kubelet 上启用 NodeSwap 功能门，并且 --fail-swap-on 命令行标志或 failSwapOn 配置设置必须设置为 false。

在 1.28 版本，该功能被升级为 Beta，功能门默认为关闭，该功能仅支持应用于 Bustable 类型的容器组且仅支持 Cgroup v2 (对 v1 的支持将被删除)。对于 BestEffort 类型的容器组，在存在节点压力的情况下应该被驱逐，Swap 交换区将被禁用。

Burstable 类型的容器组的 Swap 限制的公式为：`(<容器内存请求>/<节点内存容量>)*<节点交换容量>`。此外，内存限制等于内存请求的容器也无法访问 Swap 交换区。

### [Instrumentation] KEP-3498: 指标静态分析扩展机制 GA

静态分析扩展机制升级到指标稳定性框架, 它包括对 BETA 级指标的支持和指标文档的自动生成。有关各个稳定性级别（以及指标本身）的更多信息，请参见 https://kubernetes.io/docs/reference/instrumentation/metrics/

### [Apps] KEP-3939: Job API 新增 Pod 替换策略支持 Alpha

Kubernetes 1.28 为 Job API 添加了一个新字段，允许用户指定是否希望控制平面在前一个 Pod 开始终止时立即创建新 Pod（现有行为），或者仅在现有 Pod 完全终止时创建新 Pod（新的、 可选行为）。

许多常见的机器学习框架（例如 Tensorflow 和 JAX）需要每个索引都有唯一的 Pod。 在旧的行为中，如果属于索引作业的 pod 进入终止状态（由于抢占、驱逐或其他外部因素），则会创建一个替换 pod，但由于与旧 pod 发生冲突，该 pod 立即无法启动 尚未关闭。

在前一个 Pod 完全终止之前出现替换 Pod 也可能会导致资源稀缺或预算紧张的集群出现问题。 这些资源可能很难获取，因此可能需要很长时间才能找到资源，并且只有在现有 Pod 终止后才能找到节点。 如果启用集群自动缩放程序，过早创建替换 Pod 可能会产生不需要的扩展。

要了解更多信息，请访问 https://kubernetes.io/docs/concepts/workloads/controllers/job/#delayed-creation-of-replacement-pods

### [Apps] KEP-3850: 每个索引的 Job 重试 BackOff 限制 Alpha

目前，索引作业的索引共享单个回退限制。 当作业达到此共享退避限制时，作业控制器将整个作业标记为失败，并清理资源，包括尚未运行完成的索引。因此，现有的实现并没有涵盖工作量真正达到要求的情况。例如，如果将索引作业用作一组长时间运行的集成测试的基础，则每次测试运行将只能发现单个测试失败。

[embarrassingly parallel](https://en.wikipedia.org/wiki/Embarrassingly_parallel)：每个索引是
完全独立于其他指标。在 1.28中，新增`backoffLimitPerIndex`等字段，扩展 Job API 以支持索引作业，其中回退限制是针对每个索引的，并且尽管某些索引失败，作业仍可以继续执行。有关更多信息，请阅读 Kubernetes 文档中的[处理 Pod 和容器故障](https://kubernetes.io/docs/concepts/workloads/controllers/job/#handling-pod-and-container-failures)。

### [存储] KEP-2268: 节点非优雅关闭功能 GA

节点非优雅关闭功能在 v1.28 中移至 GA。如果原始节点意外关闭或可能由于硬件故障或操作系统无响应而最终处于不可恢复状态，则此功能允许有状态工作负载在不同节点上重新启动。目前，此功能需要人为干预才能生效，但我们正在寻找可能的途径来努力使其自动化。当原始节点因故障导致无法优雅关闭时，集群管理员可以通过向原始节点打 `node.kubernetes.io/out-of-service=out-of-service=nodeshutdown:NoExecute` 或 `node.kubernetes.io/out-of-service=out-of-service=hardwarefailure:NoExecute` 污点来触发此功能。

### [存储] KEP-3333: 追溯默认 StorageClass 分配功能 GA

追溯默认 StorageClass 分配功能在 v1.28 中移至 GA。此功能允许为现有未绑定持久卷的声明追溯分配默认的存储类，从而使更改默认 StorageClass 变得更加容易。

### [Windows] KEP-2371: 新增对 Windows 节点的支持 Beta

它允许用户指定 Kubelet PodAndContainerStatsFromCRI featuregate，仅从 CRI 获取 Windows Pod 和容器统计信息。

### [Windows] SIG 新项目: windows-service-proxy

它是基于 [Kpng](https://github.com/kubernetes-sigs/kpng) 构建的, 用于支持在 Windows 节点实现的服务代理的实验项目。Kpng 同样也是一 Kubernetes SIG 下的项目, 它旨在针对目前 kube-proxy 存在的一些问题，重新思考实现架构，为 Kubernetes 提供一个新的服务代理实现的实验项目。

## 2. DaoCloud 社区贡献

本次发布中， DaoCloud 重点贡献了 sig-node，sig-scheduling, sig-storage 和 kubeadm 相关内容，具体功能点如下：

- [API] kube-apiserver 不再允许用户重新启用已被弃用属于 policy/v1beta1 组的 API 资源
- [网络] 添加对 Pod 容器端口中的冲突端口更新/修补的警告
- [网络] Pod API 中添加了对 “status.hostIPs” 字段的 Alpha 支持，并添加了对 “status.hostIPs” 的向下 API 支持。要使用其中任何一个，都需要启用 Alpha 功能门 “PodHostIPs”。
- [调度] 当调度器执行 Reserve 插件时，如果插件的调度结果为不可调度时，调度器将此插件的名称记录在失败的插件列表中，以便当观测到相关资源事件时，重新计算决定是否将 Pod 放入到可调度队列中。
- [kubeadm] `crictl pull` 应当使用 `-i` 设置容器镜像服务的端点
- [Apps] 当 Cronjob 控制器在协调 Cronjob 创建 Job 资源时，如果因命名空间的状态是 Terminating 而导致失败时，直接返回错误, 而不是重试
- [Apps] 修复 `kubectl create job test-job --from=cronjob/a-cronjob` 命令未更新 Cronjob `status.lastSuccessfulTime` 的问题
- [Apps] `force_delete_pods_total` 和 `force_delete_pod_errors_total` 指标对所有 pod 删除行为进行计数。
- [存储] 弃用对 Ceph RBD 卷的 CSI 迁移的支持
- [Instrumentation] kube-controller-manager 和 kube-scheduler 使用结构化和上下文日志记录。

在 v1.28 发布过程中，DaoCloud 参与近百个问题修复和功能研发，作为作者约有 98 个提交，详情请见[贡献列表](https://www.stackalytics.io/cncf?project_type=cncf-group&release=all&metric=commits&module=github.com/kubernetes/kubernetes&date=120)（该版本的两百多位贡献者中有来自 DaoCloud 的 15 位）。
在 Kubernetes v1.28 的发布周期中，DaoCloud 的多名研发工程师取得了不少成就。
其中，其中，李信翻译了很多近期的官网博客和文档，成为了 SIG-Docs-Zh 的维护者。 DaoCloud 还有两位中文文档维护者要海峰和刘梦姣。
殷纳成为 SIG-scheduler 的 Approver。徐俊杰成为 kubeadm 项目的Approver。
蔡威成为 Containerd 社区的 Reviewer，并且是 KCD 北京站 2023 的组织者和主持人。在KCD 北京，张世明带来了社区热点项目 Kwok（Kubernetes WithOut Kubelet） 的相关分享，主题为 “Kwok 低成本模拟集群”，同时蒋兴彦作为另一场讲演的共同讲演者, 向与会者分享了 “Karmada 跨集群弹性伸缩场景与实现剖析” 的主题演讲。

### Kubecon 2023

在即将在上海举行的 KubeCon China 2023 全球峰会中，DaoCloud 将为大家带来十几个云原生相关的主题，包括多个维护者主题， AI 相关的云原生实践，还有多云和金融汽车等落地实践等，尽情期待。
刘齐均、潘远航和徐俊杰入选 2023 Kubecon 评审委员会; 同时韩小鹏也被邀请参加 IstioCon 的评审委员会。

#### 开源展台

**[Clusterpedia](https://clusterpedia.io)**

名字借鉴自 Wikipedia，同样也展现了 Clusterpedia 的核心理念 —— 多集群的百科全书。通过聚合多集群资源，在兼容 Kubernetes OpenAPI 的基础上额外提供了更加强大的检索功能，让用户更快更方便的在多集群中获取到想要的任何资源。当然 Clusterpedia 的能力并不仅仅只是检索查看，未来还会支持对资源的简单控制，就像 wiki 同样支持编辑词条一样。

**[Merbridge](https://merbridge.io)**

Merbridge 专为服务网格设计，使用 eBPF 代替传统的 iptables 劫持流量，能够让服务网格的流量拦截和转发能力更加高效。

**[Hwameistor](https://hwameistor.io)**

它是一款 Kubernetes 原生的容器附加存储 (CAS) 解决方案，将 HDD、SSD 和 NVMe 磁盘形成本地存储资源池进行统一管理，使用 CSI 架构提供分布式的本地数据卷服务，为有状态的云原生应用或组件提供数据持久化能力。

**其他**

Containerd、Kubespray、Piraeus、SIG-Scheduling、SIG-Node 等展台，同样也有 DaoCloud 的同学在现场为大家解答问题，欢迎大家前来交流。

#### 主题演讲

**27 号 11 点 Kay Yan, Kante Yin, Alex Zheng, XingYan Jiang**

- 🌟 [SIG集群生命周期：Kubespray的新功能 | SIG Cluster Lifecycle: What's New in Kubespray - Kay Yan, DaoCloud & Peng Liu, Jinan Inspur Data Technology Co, Ltd.](https://kccncosschn2023.sched.com/event/1PTJt/sigzhong-shi-chang-potodaepkubesprayzha-xia-sig-cluster-lifecycle-whats-new-in-kubespray-kay-yan-daocloud-peng-liu-jinan-inspur-data-technology-co-ltd) Maintainer Track | 3夹层 3M5A会议室 | 3M ROOM 3M5A
- [使用KubeRay和Kueue在Kubernetes中托管Sailing Ray工作负载 | Sailing Ray Workloads with KubeRay and Kueue in Kubernetes - Jason Hu, Volcano Engine & Kante Yin, DaoCloud](https://kccncosschn2023.sched.com/event/1PTGw/zhi-kuberayrekueuenanokubernetesfa-sailing-raydu-zhe-sailing-ray-workloads-with-kuberay-and-kueue-in-kubernetes-jason-hu-volcano-engine-kante-yin-daocloud) | 3层 307会议室| 3F ROOM 307 
- [金融行业云原生存储的最佳实践 | Best Practice for Cloud Native Storage in Finance Industry - Yang Cao & Jie Chu, Shanghai Pudong Development Bank; Alex Zheng, Daocloud2层](https://kccncosschn2023.sched.com/event/1PTFU/shu-gun-chan-chang-zha-huan-best-practice-for-cloud-native-storage-in-finance-industry-yang-cao-jie-chu-shanghai-pudong-development-bank-alex-zheng-daocloud) 会议室 2 | 2F ROOM 2
- [突破集群边界，在大规模上自动扩展工作负载 | Break Through Cluster Boundaries to Autoscale Workloads Across Them on a Large Scale - Wei Jiang, Huawei Cloud & XingYan Jiang, DaoCloud](https://kccncosschn2023.sched.com/event/1PTJ7/di-wu-zhong-shi-yi-dui-daelsnanomao-kek-du-zhe-break-through-cluster-boundaries-to-autoscale-workloads-across-them-on-a-large-scale-wei-jiang-huawei-cloud-xingyan-jiang-daocloud) 3层 305A会议室| 3F ROOM 305A

**27号 2:45 p.m. Iceber Gu**

- 🌟 [项目更新和深入探讨：containerd | Project Update and Deep Dive: Containerd - Wei Fu, Microsoft & Iceber Gu, DaoCloud](https://kccncosschn2023.sched.com/event/1PTKB/tu-ju-recheng-daepcontainerd-project-update-and-deep-dive-containerd-wei-fu-microsoft-iceber-gu-daocloud) Maintainer Track | 3夹层 3M3会议室 | 3M ROOM 3M3

**27号 2:55 pm - 3:00pm Peter Pan**

- [⚡闪电演讲: 带有人工智能能力的云原生 Kubernetes 运维 | Lightning Talk: Sailing Kubernetes Operation with AI Power - Peter Pan, Daocloud](https://kccncosschn2023.sched.com/event/1PTFI/clcong-shu-xi-xia-xia-zha-chang-kubernetes-ai-gu-lightning-talk-sailing-kubernetes-operation-with-ai-power-peter-pan-daocloud) 3层 302会议室| 3F ROOM 302

**27号 3:50 p.m. Mengjiao Liu, Kante Yin**

- 🌟 [SIG 仪器仪表：介绍、深入探讨和最新发展 | SIG Instrumentation: Intro, Deep Dive and Recent Developments - Mengjiao Liu, DaoCloud & Shivanshu Raj Shrivastava, Tetrate](https://kccncosschn2023.sched.com/event/1PTJq/sig-jnai-daeptao-cheng-re-sig-instrumentation-intro-deep-dive-and-recent-developments-mengjiao-liu-daocloud-shivanshu-raj-shrivastava-tetrate) Maintainer Track | 3夹层 3M3会议室 | 3M ROOM 3M3
- 🌟 [SIG-Scheduling 介绍与深入探讨 | SIG-Scheduling Intro & Deep Dive - Qingcan Wang, Shopee & Kante Yin, DaoCloud](https://kccncosschn2023.sched.com/event/1PTKH/sig-scheduling-tao-cheng-sig-scheduling-intro-deep-dive-qingcan-wang-shopee-kante-yin-daocloud) Maintainer Track | 3夹层 3M5A会议室 | 3M ROOM 3M5A 

**27号 4:40 p.m. MinJie Huang & WenJie Song**

- [构建一个主动-主动的高可用Kubernetes控制平面集群 | Building an Active-Active HA Kubernetes Control Plane Cluster - MinJie Huang & WenJie Song, DaoCloud; Jiashun Dai, SAIC General Motors](https://kccncosschn2023.sched.com/event/1PTFj/zha-pan-zhi-kubernetesxiao-zhong-shi-building-an-active-active-ha-kubernetes-control-plane-cluster-minjie-huang-wenjie-song-daocloud-jiashun-dai-saic-general-motors) 3层 301明珠厅| 3F THE PEARL HALL 301 

**28号 11:50 a.m. Paco Xu、Michael Yao**

- 🌟 [Kubernetes SIG节点介绍和深入探讨 | Kubernetes SIG Node Intro and Deep Dive - Paco Xu, DaoCloud](https://kccncosschn2023.sched.com/event/1PTJk/kubernetes-sigze-tao-recheng-kubernetes-sig-node-intro-and-deep-dive-paco-xu-daocloud) Maintainer Track | 3夹层 3M3会议室 | 3M ROOM 3M3
- [Kubernetes 文档和本地化 | Kubernetes Documentation and Localization - Michael Yao, DaoCloud & Xin Li, Qihoo](https://kccncosschn2023.sched.com/event/1PTFs/kubernetes-repico-kubernetes-documentation-and-localization-michael-yao-daocloud-xin-li-qihoo-360) 3602层 会议室 3 | 2F ROOM 3

**28号 2:45 p.m. Hongbing Zhang, Shiming Zhang, Paco Xu**

- 🌟 [云原生边缘计算与KubeEdge：更新与未来 | Cloud Native Edge Computing with KubeEdge: Updates and Future - Fei Xu, Huawei Cloud & Hongbing Zhang, Shanghai DaoCloud Network Technology](https://kccncosschn2023.sched.com/event/1PTKc/chang-yi-sui-dou-zhao-kubeedgedaep-cloud-native-edge-computing-with-kubeedge-updates-and-future-fei-xu-huawei-cloud-hongbing-zhang-shanghai-daocloud-network-technology) Maintainer Track | 3夹层 3M5A会议室 | 3M ROOM 3M5A
- 🌟 [深入研究：KWOK | Deep Dive: KWOK - Shiming Zhang, DaoCloud & Hao Liang, Tencent](https://kccncosschn2023.sched.com/event/1PTJn/ye-ge-daepkwok-deep-dive-kwok-shiming-zhang-daocloud-hao-liang-tencent) Maintainer Track | 3夹层 3M3会议室 | 3M ROOM 3M3
- [如何在大型集群中加速 Pod 的启动？ | How Can Pod Start-up Be Accelerated on Nodes in Large Clusters? - Paco Xu, DaoCloud & Byron Wang, Birentech](https://kccncosschn2023.sched.com/event/1PTFR/nanonfzhong-shi-fu-pod-zha-dyags-how-can-pod-start-up-be-accelerated-on-nodes-in-large-clusters-paco-xu-daocloud-byron-wang-birentech) 2层 会议室 1 | 2F ROOM 1

## 3. 其他需要了解的功能

- [apps] kubernetes 控制器弃用 `--volume-host-cidr-denylist` 和 `--volume-host-allow-local-loopback` 命令行参数。
- [apps] 为从 CronJobs 调度的 Job 对象添加了新的注释 batch.kubernetes.io/cronjob-scheduled-timestamp
- [apps] StatefulSet 将索引设置为 Pod 标签 statefulset.kubernetes.io/pod-index
- [apps] kube-controller-manager：LegacyServiceAccountTokenCleanUp 功能门现在可作为 Alpha 使用（默认情况下关闭）。 启用后，legacy-service-account-token-cleaner 控制器循环会删除在 `--legacy-service-account-token-clean-up-period` 指定的时间内未使用的服务帐户令牌密钥（默认为一年），服务帐户令牌是从 ServiceAccount 对象的 .secrets 列表中引用的，而不是从 pod 中引用的。
- [apps] 添加了对 pod hostNetwork 字段选择器的支持
- [APIMachinery] client-go：在观察大量不经常更改的对象时，改进了反射器缓存的内存使用
- [APIMachinery] 添加了 ConcientListFromCache 功能门，允许 apiserver 从缓存提供一致的资源列表。
- [CLI] 在 `registry.k8s.io/kubectl` 中为 `kubectl` 添加了一个与其他镜像相同的架构的容器镜像 (linux/amd64 linux/arm64 linux/s390x linux/ppc64le)
- [CLI] 向 kubectl 添加了新的命令行参数 --interactive。 新的命令行参数允许用户以交互方式确认每个资源的删除请求。
- [网络] 设置 hostNetwork: true 并声明端口的 Pod 会自动设置 hostPort 字段。 以前，这种情况会发生在 Deployment、DaemonSet 或其他工作负载 API 的 PodTemplate 中。 现在，只有在创建实际 Pod 时才会设置 hostPort。
- [节点] kubelet: `--azure-container-registry-config` 被弃用并在未来的版本中会被删除。请使用 `--image-credential-provider-config` 和 `--image-credential-provider-bin-dir` 来设置。 
- [节点] 如果使用 cgroups v2，则将通过 memory.oom.group 为容器 cgroup 启用 cgroup 感知 OOM 杀手。这会导致 cgroup 中的进程被视为一个单元，并在 cgroup 中任何进程发生 OOM 终止时同时终止。
- [调度] kube-scheduler: 删除了 `--lock-object-namespace` 和 `--lock-object-name`。请使用 `--leader-elect-resource-namespace` 和 `--leader-elect-resource-name` 或 ComponentConfig 来配置这些参数.
- [调度] 在 “KubeSchedulerConfiguration” 中添加了新的配置选项 “delayCacheUntilActive”，当 “kube-scheduler” 发生 leader 身份变化时，可以在内存效率和调度速度之间提供权衡。
- [Auth] KMSv1 已弃用，以后只会接收安全更新。 请改用 KMSv2。在未来版本中，设置 --feature-gates=KMSv1=true 以使用已弃用的 KMSv1 功能。
- [存储] GCE PD 树内插件在 v1.28 中删除（CSI 迁移完成）。
- [存储] 从 pvc.Status 中删除了 resizeStatus 枚举并替换为 AllocatedResourceStatus
- [节点] PodResources API GA
- [节点] 实现了 Container Runtime 的 cgroup 驱动程序的自动检测，以便在 kubelet 中使用。

## 4. 版本标志

本次发布的主题是 Planternetes（这个新造词源于 Plant + Kuberenetes，意为绿植、环保、生机盎然的容器云编排）。

![logo](kubernetes-1.28.png)

每个 Kubernetes 版本都是我们数以千计的社区人员辛勤工作的结晶。这个版本背后的人有行业资深人士、前辈和社区新人，他们向我们贡献了独特的经验，创造了具有全球影响力的集体工作。

就像花园一样，我们的版本有不断变化的成长、挑战和机遇。这个主题颂扬了我们为取得今天的成绩而付出的心血和努力。和谐共处，我们成长得更好。

## 5. 升级注意事项

- 自定义调度程序插件开发人员需要采取的行动。 这是调度框架中 EnqueueExtension 的重大变化。 EnqueueExtension 中的 EventsToRegister 将返回值从 ClusterEvent 更改为 ClusterEventWithHint。 ClusterEventWithHint 允许每个插件通过名为 QueueingHintFn 的回调函数过滤掉更多无用的事件。 当调度队列收到集群事件时，在将每个 Pod 从不可调度的 pod 池移动到 activeQ/backoffQ 之前，会调用上一个调度周期中拒绝每个 Pod 的插件的 QueueingHintFn。为了向后兼容，nil QueueingHintFn 被视为始终返回 QueueAfterBackoff。 因此，如果您只想保留现有行为，可以注册 ClusterEventWithHint，其中不包含 QueueingHintFn。 但是，从调度性能的角度来看，注册适当的 QueueingHintFn 当然更好。
- Ceph RBD 树内插件已在 v1.28 中弃用，并计划在 v1.31 中删除（没有计划进行 CSI 迁移, 建议使用 RBD 模式的 [Ceph CSI](https://github.com/ceph/ceph-csi) 第三方存储驱动程序作为替代方案）。
- Ceph FS 树内插件已在 v1.28 中弃用，并计划在 v1.31 中删除（没有计划进行 CSI 迁移）。建议使用 [Ceph CSI](https://github.com/ceph/ceph-csi) 第三方存储驱动程序作为替代方案）。

## 6. 历史文档

- [近两年功能增加最多！Kubernetes 1.27 正式发布](https://mp.weixin.qq.com/s/maDEiCGzOPSDkH9dUxIxdA)
- [Kubernetes 正式发布 v1.26，稳定性显著提升](https://mp.weixin.qq.com/s/qwzmeIM4INz-_BK_gbwOxw)
- [Kubernetes 1.25 正式发布，多方面重大突破](https://mp.weixin.qq.com/s/aRmLBYpk0MhLJAwY85DyuA)
- [Kubernetes 1.24 走向成熟的 Kubernetes](https://mp.weixin.qq.com/s/vqH8ueaZeEeZbx_axNVSjg)
- [Kubernetes 1.23 正式发布，有哪些增强？](https://mp.weixin.qq.com/s/A5GBv5Yn6tQK_r6_FSyp9A)
- [Kubernetes 1.22 颠覆你的想象：可启用 Swap，推出 PSP 替换方案，还有……](https://mp.weixin.qq.com/s/9nH2UagDm6TkGhEyoYPgpQ)
- [Kubernetes 1.21 震撼发布 | PSP 将被废除，BareMetal 得到增强](https://mp.weixin.qq.com/s/amGjvytJatO-5a7Nz4BYPw)

## 7. 参考

1. Kubernetes 1.28 发布团队 <https://github.com/kubernetes/sig-release/blob/master/releases/release-1.28>
2. Kubernetes 1.28 变更日志 <https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.27.md>
3. Kubernetes 1.28 主题讨论 <https://github.com/kubernetes/sig-release/discussions/2271>