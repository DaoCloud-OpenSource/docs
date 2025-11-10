# K8s 1.35 发布！安装/升级变化巨大，新特性 Gang Scheduling 重磅来袭！

2025 年 12 月 17 日，主题为 Timbernetes 的 Kubernetes v1.35 正式发布。

此版本距离上版本发布时隔 4 个月，是 2025 年的最后一个版本。

与之前的版本类似，Kubernetes v1.35 版本引入了多项新的 Alpha、Beta 和 Stable 的功能。一贯交付高质量版本的承诺凸显了 Kubernetes 社区的开发周期实力和社区的活跃支持。

此版本包含 55 项改进。在这些改进功能中，有 16 项已晋升为 Stable，有 17 项进入 Beta 阶段，有 22 项为 Alpha 阶段。

## 发布主题和 Logo

Kubernetes v1.35 的发布主题是 Timbernetes: 它描述地是三只松鼠守护着这棵树：一位手持 LGTM 卷轴的法师，象征着审阅者；一位手持战斧、举着 Kubernetes 盾牌的战士，代表为发布开辟新枝的发布团队；还有一位提灯的游侠，象征那些为阴暗的 issue 队列带来光亮的分诊者。他们共同指代着一支更为庞大的冒险者队伍。

![](./k8s-1.35.png)

2025 年在「八色魔法」（The Color of Magic，v1.33）的微光中启程，又乘着「风与意志」（Of Wind & Will，v1.34）的疾风前行。至此，我们在年末将双手放在「世界之树」之上 —— 灵感源自北欧神话中的生命之树尤克特拉希尔（Yggdrasil），它联结着众多世界。

如同任何一棵伟大的树，Kubernetes 也以年轮般的节奏生长：一次发布，便添一圈年轮；而其形态，则由全球社区的悉心照料所塑造。树心之处，是环绕地球的 Kubernetes 舵轮，稳稳扎根于那些始终如一的维护者、贡献者与用户之上。正是他们在本职工作、人生变迁与持续的开源守护之间，修剪陈旧的 API，嫁接崭新的特性，让这个世界上规模最大的开源项目之一始终保持健康与活力。

Kubernetes v1.35 为这棵世界树再添一圈年轮 —— 这是一道由无数双手、无数条道路共同塑造的新刻痕。随着根系愈发深厚，社区的枝叶也将伸展得更高、更远。

## 安装/升级注意事项

### 操作系统

随着 1.35 版本的发布，Kubernetes 已弃用 cgroup v1, 移除将遵循 Kubernetes 弃用策略。默认情况下，kubelet 将不再在 cgroup v1 节点上启动。同时, `kubeadm` 将开始验证主机上的 `cgroups` 和 `kubelet` 版本。如果在主机上检测到 `cgroups v1`，并且检测到的目标 `kubelet` 版本为 1.35 或更高版本，则会抛出预检错误。要在 `kubeadm` 和 `kubelet` 版本 1.35 或更高版本中启用 cgroups v1，您必须：

1. 忽略 `kubeadm` 执行 `SystemVerification` 预检时出现的错误。
2. 升级前，编辑 `kube-system/kubelet-config` `ConfigMap` 并添加 `failCgroupV1: false` 字段。

### 容器运行时

1. 推荐用户升级到 containerd 2.0 或更高版本。`kubeadm` 将开始验证主机上的 `containerd` 版本。若检测到已安装容器运行时不满足即将到来的需求，则会在预检时抛出警告提示用户尽快进行升级。用户可以通过 kubelet 提供的 `kubelet_cri_losing_support` 监控指标来确定集群中是否有节点的容器运行时需要进行升级。1.35 版本是最后一个支持 containerd 1.x 版本的版本。

2. 我们预计整个生态系统会采纳新的 CPU 权重计算公式, 建议在使用 1.35 和 cgroupv2 时, 同时升级容器运行时的依赖项, 以避免出现新旧公式更替带来的性能问题。详情请参考 [Issue 131216](https://github.com/kubernetes/kubernetes/issues/131216)。
   - runc（1.3.2+）
   - crun（1.23+）

## GA 和稳定的功能

GA 全称 General Availability，即正式发布。Kubernetes 的进阶路线通常是 Alpha、Beta、Stable (即 GA)、Deprecation/Removal 这四个阶段。下文择取了部分特性详述。如果对其他特性感兴趣，请移步至具体的 KEP 页面了解进展和详情。

### 更新总览

- [KEP-859 Kubectl 发起请求时会携带 Kubectl-Command 和 Kubectl-Session 两个 HTTP Header，用于追踪请求来源和标识会话](https://kep.k8s.io/859)
- [KEP-1287 支持了 Pod 资源原地升级（In-Place Update）功能，允许在不重新创建 Pod 或重新启动容器的情况下更改资源](https://kep.k8s.io/1287)
- [KEP-3015 添加一种方法来向 kube-proxy 发出信号，表明它应该尽可能将流量传输到本地端点，以提高效率](https://kep.k8s.io/3015)
- [KEP-3331 添加结构化身份验证配置](https://kep.k8s.io/3331)
- [KEP-3619 细粒度的补充组控制](https://kep.k8s.io/3619)
- [KEP-3673 为 kubelet 添加节点级限制，以限制并行镜像拉取的数量](https://kep.k8s.io/3673)
- [KEP-3983 添加对 kubelet 插入式配置目录的支持](https://kep.k8s.io/3983)
- [KEP-4006 将 Kubernetes 客户端的双向流协议从 SPDY/3.1 转换为 WebSockets](https://kep.k8s.io/4006)
- [KEP-4210 添加 Kubelet 选项，用于指定镜像在被垃圾回收之前保留的最大期限](https://kep.k8s.io/4210)
- [KEP-4368 允许 Job 的协调过程被委派给外部控制器的 `manage-by` 工作机制](https://kep.k8s.io/4368)
- [KEP-4540 引入了新的 CPUManager 策略选项 strict-cpu-reservation，确保 reservedSystemCPUs 严格保留用于系统守护程序或中断处理，并且不会被 QoS 类为 Burstable 和 BestEffort 的 Pod 使用](https://kep.k8s.io/4540)
- [KEP-4622 向 kubelet 添加一个配置选项，允许在 TopologyManager 中配置 maxAllowableNUMANodes 的值](https://kep.k8s.io/4622)
- [KEP-5067 利用现有的元数据 Generation 字段，并在 Pod 状态中添加一个新的 status.observedGeneration 字段，允许 Pod 状态字段表示当前哪些 Pod 更新反映在 Pod 状态中](https://kep.k8s.io/5067)
- [KEP-5468 用于在 Kubernetes E2E 测试期间, 通过组件指标检测指标不变性的违规情况](https://kep.k8s.io/5468)
- [KEP-3304 `k8s.io/apimachinery/util/resourceversion` 提供了一个帮助函数 `CompareResourceVersion`, 用于允许客户端比较同一类型对象的资源版本大小](https://kep.k8s.io/5504)
- [KEP-5589 删除 Kubernetes API 类型的 gogo protobuf 依赖项](https://kep.k8s.io/5589)

## 进入 Beta 阶段的功能

Beta 阶段的功能是指那些已经经过 Alpha 阶段的功能，且在 Beta 阶段中添加了更多的测试和验证，通常情况下是默认启用的。下文择取了部分特性详述。如果对其他特性感兴趣，请移步至具体的 KEP 页面了解进展和详情。

### 动态调整节点上允许挂载的 CSI 卷的数量

传统上，Kubernetes 的 CSI 驱动在初始化时会报告一个静态的最大卷挂接限制。 然而，在节点的生命周期中，实际的挂接数量可能因各种原因发生变化，例如：

- 在 Kubernetes 控制之外的手动或外部卷挂接/解除挂接操作。
- 动态挂接的网络接口或专用硬件（GPU、NIC 等）消耗可用的插槽。
- 在多驱动场景中，一个 CSI 驱动的操作影响另一个驱动所报告的可用容量。

静态报告可能导致 Kubernetes 将 Pod 调度到看似有容量但实际上没有容量的节点上， 从而导致 Pod 卡在 ContainerCreating 状态。

Kubernetes 现在支持两种机制来更新所报告的节点卷限制：

- 周期性更新： CSI 驱动指定一个时间间隔，定期刷新节点的可分配容量。
- 触发式更新： 当卷挂接因资源耗尽（ResourceExhausted 错误）而失败时触发立即更新。

**示例:**

```yaml
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: example.csi.k8s.io
spec:
  nodeAllocatableUpdatePeriodSeconds: 60
```

此配置指示 kubelet 每隔 60 秒调用一次 CSI 驱动的 NodeGetInfo 方法，以更新节点的可分配卷数。 Kubernetes 强制要求更新时间间隔最小为 10 秒，目的是在准确性与资源消耗间达成平衡。当卷挂接操作因 ResourceExhausted 错误（gRPC 代码 8）而失败时，Kubernetes 会立即更新可分配数量， 而不是等待下一次周期性更新。随后 kubelet 会将受影响的 Pod 标记为 Failed，使其控制器能够重新创建这些 Pod。 这样可以防止 Pod 永久卡在 ContainerCreating 状态。

### 更新总览

- [KEP-127 支持用户命名空间，进程能够在具有与主机不同的用户和组 ID 的 pod 中运行。](https://kep.k8s.io/127)
- [KEP-961 StatefulSet 增加 `MaxUnavailable` 字段，支持同时下线多个 Pod 达到快速滚动更新的目的](https://kep.k8s.io/961)
- [KEP-3077 下文日志记录使函数的调用者能够控制日志记录的所有方面（输出格式、详细程度、附加值和名称）](https://kep.k8s.io/3077)
- [KEP-3721 支持从文件实例化容器的环境变量。该文件必须位于 emptyDir 卷中。该环境变量文件可以由 emptyDir 卷中的 initContainer 创建。kubelet 将从 emptyDir 卷中的指定文件实例化容器中的环境变量，但不会挂载该文件。请注意，该卷只会挂载到环境变量生产者容器（initContainer），而使用该环境变量的普通容器不会挂载该卷。](https://kep.k8s.io/3721)
- [KEP-4192 移动存储版本迁移器到主库](https://kep.k8s.io/4192)
- [KEP-4317 为 Pod 提供了使用 mTLS 向 kube-apiserver 进行身份验证的内置功能。它由易于外部项目复用的组件构建而成，以便使用 X.509 证书提供更多功能](https://kep.k8s.io/4317)
- [KEP-4742 允许在 Pod 中通过 downward API 中使用节点标签](https://kep.k8s.io/4742)
- [KEP-4762 在 podSpec 中引入了一个新的字段 hostnameOverride, 允许用户将任意的完全限定域名 (FQDN) 设置为 Pod 的主机名](https://kep.k8s.io/4762)
- [KEP-4876 允许 CSI 驱动程序可以在运行时动态调整可附加卷的最大数量](https://kep.k8s.io/4876)
- [KEP-4951 HorizontalPodAutoscaler 新增 tolerance 字段来指定允许的指标偏差，当使用比率约为 1 时，使用自定义值可精细地触发扩容操作](https://kep.k8s.io/4951)
- [KEP-5278 除了使用 NominatedNodeName 来指示正在进行的抢占之外，调度器还可以在绑定周期开始时指定它，以向其他组件显示预期的 Pod 位置。此外，其他组件也可以将 NominatedNodeName 放在待处理的 Pod 上，以指示该 Pod 优先调度到特定节点](https://kep.k8s.io/5278)
- [KEP-5295 kubectl 引入 KYAML，一种更安全、更少歧义的 YAML 子集/编码](https://kep.k8s.io/5295)
- [KEP-5307 引入了容器重启规则，以便 kubelet 可以在容器退出时应用这些规则。这将允许用户配置容器的特殊退出代码，使其被视为非故障状态，即使 Pod 的 restartPolicy=Never 也能原地重启容器。](https://kep.k8s.io/5307)
- [KEP-5311 放宽 Service 的名称验证, 允许服务名称以数字开头。](https://kep.k8s.io/5311)
- [KEP-5538 现在，CSI 驱动程序可以通过在 CSIDriver 对象中设置 `spec.serviceAccountTokenInSecrets: true` 来选择通过 secrets 字段而非卷上下文接收服务帐户令牌。这可以防止令牌在日志和其他输出中暴露。](https://kep.k8s.io/5538)
- [KEP-5573 Kubernetes 已弃用 cgroup v1, 移除将遵循 Kubernetes 弃用策略。默认情况下，kubelet 将不再在 cgroup v1 节点上启动。要禁用此设置，集群管理员应在 kubelet 配置文件中将 failCgroupV1 设置为 false](https://kep.k8s.io/5573)
- [KEP-5593 改进 Pod 重启退避逻辑，以更好地匹配它创建的实际负载并满足新兴用例，同时为集群作员提供一个选项，以便为特定节点上的所有容器配置更低的最大回退，最低可至 1 秒](https://kep.k8s.io/5593)

## 进入 Alpha 阶段的功能

Alpha 阶段的功能是指那些刚刚被引入的功能，这些功能是默认关闭的，需要用户手动开启。

### 群组调度（Gang Scheduling）支持

kube-scheduler 被修改以支持 gang scheduling。目前专注于框架支持和构​​建模块，而不是理想的 gang scheduling 算法——它可以作为后续工作。首先从更简单的 gang scheduling 实现开始，kube-scheduler 识别组中的 pod，并等到所有 pod 都达到调度/绑定周期的同一阶段后，才允许该组中任何 pod 继续前进。如果不是所有 pod 都能在超时到期前达到该阶段，则调度程序将停止尝试调度该组，所有 pod 都会释放所有资源。这允许其他工作负载尝试分配这些资源。

因此, 引入了一个名为 Workload 的新类型，用于告知 kube-scheduler 一组 Pod 应该一起调度，以及所有与群组调度相关的策略选项。Pod 的 spec 中包含一个指向其 Workload 的对象引用（可选类型）。

**Workload 示例:**

```
apiVersion: batch/v1
kind: Job
metadata:
  name: job-1
  namespace: ns-1
spec:
  completions: 100
  parallelism: 100
  completionMode: Indexed
  template:
    spec:
      workload:  # 指向其 Workload 的对象引用
        name: job-1
        podGroup: pg1
      restartPolicy: OnFailure
      containers:
      - name: ml-worker
        image: awesome-training-program:v1 
        command: ["python", "train.py"]
        resources:
          limits:
            nvidia.com/gpu: 1
        env:
        - name: JOB_COMPLETION_INDEX
          valueFrom:
            fieldRef:
              fieldPath:
               "metadata.annotations['batch.kubernetes.io/job-completion-index']"
---
apiVersion: scheduling/v1alpha1   
kind: Workload
metadata:
  namespace: ns-1
  name: job-1
spec:
  podGroups:
    - name: "pg1"
      policy:
        gang:
          minCount: 100
```

**`Workload` 与 `PodGroup` 的关系:**

当个 `Workload` 可以包含多个 `PodGroup`，但每个 `PodGroup` 被视为独立的调度单元。如果用户的真正的意图是 "要么两个都运行，要么两个都不运行"，那么它们就应该组成一个单一的 PodGroup，而不应该拆分成两个。`LeaderWorkerSet` 就是一个很好的例子：一个副本由一个 `leader` 和 `N` 个 `worker` 组成，可将它组织为一个 `PodGroup` 作为一个完整的调度单位；

**计划在 1.36 版本实现的功能:**

- 扩展 `Workload` API, 支持更多调度策略.
- 为 `Job`、`StatefulSet`、`JobSet` 和 `LeaderWorkerSet` 自动创建对应的 `Workload` 资源.
- 拓扑感知调度
- 引入单周期 `Workload` 调度阶段, 在此阶段，我们将检查单个调度周期内工作负载中的所有 Pod，并尝试找到最佳放置位置。
- 引入基于树的 `Workload` 调度算法
- 改进 `Workload` 的绑定过程（解决使用提名机制的 kubelet 竞争问题）
- 延迟抢占（引入指定受害者的概念）
- 支持 `Workload` 抢占

### Toleration 新增数值比较运算符（Lt、Gt）支持

许多生产环境的 Kubernetes 集群会混合使用按需节点（ SLA 较高 ）和竞价/抢占式节点（ SLA 较低 ），以在保证关键工作负载可靠性的同时优化成本。平台团队需要一个安全的默认设置，使大多数工作负载远离高风险容量，同时允许特定工作负载通过诸如“SLA ≥ 95%”之类的明确阈值进行选择。为此，我们扩展了 core/v1 Toleration ，使其在匹配节点污点时支持数值比较运算符（Lt、Gt）。这既保留了污点/容忍度机制中广为人知的安全模型，又可实现了基于阈值的 SLA 感知调度。

**示例:**

```
# Spot nodes with 80% SLA get a repelling taint
apiVersion: v1
kind: Node
metadata:
  name: spot-node-1
spec:
  taints:
  - key: node.kubernetes.io/sla
    value: "800"
    effect: NoSchedule
---
# Cost-optimized workload explicitly tolerates SLA > 750
apiVersion: v1
kind: Pod
spec:
  tolerations:
  - key: node.kubernetes.io/sla
    operator: Gt
    value: "750"
    effect: NoSchedule
---
apiVersion: v1
kind: Pod
metadata:
  name: flexible-sla-workload
spec:
  tolerations:
  # Accept nodes with SLA > 900
  - key: node.kubernetes.io/sla
    operator: Equal
    value: "900"
    effect: NoSchedule
  - key: node.kubernetes.io/sla
    operator: Gt
    value: "900"
    effect: NoSchedule
---
# Critical workload will not be scheduled until a suitable high reliability node has capacity
apiVersion: v1
kind: Pod
metadata:
  name: critical-workload
spec:
  tolerations:
  - key: node.kubernetes.io/sla
    operator: Gt
    value: "950"
    effect: NoSchedule
```

### 更新总览

- [KEP-4020 允许在存在多个不同版本的 API 服务器时，由正确的 API 服务器处理资源请求](https://kep.k8s.io/4020)
- [KEP-4671 kube-scheduler 被修改以支持 gang scheduling。](https://kep.k8s.io/4671)
- [KEP-4815 向 DRA 添加对可分区设备的支持](https://kep.k8s.io/4815)
- [KEP-4827 为所有核心组件实现 statusz 端点来公开有关组件基本信息、运行状况和关键指标的标准化实时数据](https://kep.k8s.io/4827)
- [KEP-4828 为所有核心组件实现 flagz 端点来公开有关组件的命令行标志的配置参数](https://kep.k8s.io/4828)
- [KEP-4872 加强 Kubelet Serving 中 Kube-API 服务器的证书验证](https://kep.k8s.io/4872)
- [KEP-5007 DRA 引入一种机制 ( BindingConditions 和 BindingFailureConditions)，允许调度程序推迟 Pod 绑定，直到确认外部资源已准备就绪](https://kep.k8s.io/5007)
- [KEP-5030 修复集群自动扩缩器 (CAS)，使其在扩缩新节点时能够感知节点的卷挂载限制，并防止调度器将 pod 放置在未安装特定 CSI 驱动程序的节点上。](https://kep.k8s.io/5030)
- [KEP-5055 支持将设备标记为受污染可以防止将其用于新 Pod 和/或导致使用它们的 Pod 停止, 用户可以决定是否容忍这些污染。](https://kep.k8s.io/5055)
- [KEP-5075 允许 ResourceSlice 中的单个设备在不同的独立 ResourceClaims 之间共享，这些 ResourceClaims 由可分配容量值进行管理（前提是驱动程序允许）。这使得 DRA 设备可以由独立的工作负载共享，而不是像现在这样只能由协作的工作负载共享](https://kep.k8s.io/5075)
- [KEP-5237 将路由控制器（ k8s.io/cloud-provider ）的协调循环从固定间隔切换到基于 Watch 的方式](https://kep.k8s.io/5237)
- [KEP-5284 引入新的授权规则，以限制对指定资源执行指定操作的模拟](https://kep.k8s.io/5284)
- [KEP-5328 提出了一个节点声明特性框架，用于节点声明特定、特性门控的 Kubernetes 特性的可用性。控制平面将使用此框架进行调度和准入决策，主要用于管理版本偏差。对于调度，kube-scheduler 将利用这些声明的特性来确保 pod 仅被部署到具备成功运行所需特性的节点上。对于 API 请求验证，准入控制器将阻止在缺乏所需特性支持的节点上执行操作。其目的是通过减少对手动配置（例如污点、容忍度和复杂的节点标签方案）的依赖来简化集群操作](https://kep.k8s.io/5328)
- [KEP-5381 将 PersistentVolume.spec.nodeAffinity 字段设为可变字段](https://kep.k8s.io/5381)
- [KEP-5440 允许在 Job 被暂停时修改 PodTemplate 上的容器请求/限制。](https://kep.k8s.io/5440)
- [KEP-5471 扩展了 core/v1 Toleration ，使其在匹配节点污点时支持数值比较运算符（Lt、Gt）。这既保留了污点/容忍度机制中广为人知的安全模型，又可实现了基于阈值的 SLA 感知调度。](https://kep.k8s.io/5471)
- [KEP-5607 允许 Pod 同时使用用户命名空间和主机网络](https://kep.k8s.io/5607)

## 删除和废弃功能#

### kube-proxy 中的 ipvs 模式

已将 kube-proxy 中的 ipvs 模式标记为已弃用。kube-proxy 中的 ipvs 模式已被弃用，并将在未来的 Kubernetes 版本中移除。建议用户迁移到 nftables。

### 重启 kubelet 会改变 pod 状态

确保高可用性并最大限度地减少服务中断是 Kubernetes 集群的关键考量因素。默认情况下，kubelet 重启时会将所有容器的 Start 和 Ready 状态重置为 False。这意味着之前建立的任何成功探测状态都会在重启后丢失。因此，即使在 kubelet 重启前服务运行正常，也可能被错误地标记为不可用。这种重置会导致对服务健康状况的错误判断，并对集群的整体性能产生负面影响，甚至可能触发不必要的警报或负载均衡调整。我们将添加一个已弃用的功能门控： ChangeContainerStatusOnKubeletRestart 。此功能门控默认处于禁用状态。禁用后，Kubelet 在重启后将不会更改容器状态。用户可以启用功能门控 ChangeContainerStatusOnKubeletRestart 来恢复 Kubelet 在重启后更改容器状态的行为。

### 探针旧行为的废弃

`ExecProbeTimeout` 功能门已锁定为默认值。目前无法恢复到忽略执行探测超时的旧行为。

## DaoCloud 社区贡献

上半年，DaoCloud 参与多个问题修复和功能研发，并取得了不少成就。其中：

- [徐俊杰（pacoxu）](https://github.com/pacoxu) 连任 Kubernetes 指导委员会(Steering Committee)席位；
- [蒋兴彦（chaunceyjiang）](https://github.com/chaunceyjiang) 成为 vllm 项目的 Function Call Owner；
- [王昱奇 (noooop)](https://github.com/noooop) 成为 vllm 项目的 Pooling Models Owner；

## 发行说明

上述内容就是 Kubernetes v1.35 的主要更新和内容啦，更多的发布说明可以查看 [Kubernetes v1.35 版本的完整详细信息](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.35.md)。

我们下次版本发布时再见！

## 历史文档

- [迎风破浪的三只熊——Kubernetes v1.34 发布，看点全解析](https://mp.weixin.qq.com/s/adEqoMmWXWpqck6ZbCvLLg)
- [重磅！K8s正式支持Sidecar容器，v1.33版本这些改动将影响你的集群](https://mp.weixin.qq.com/s/a7ZLS59ibSbr-7m1TJpehw)
- [Kubernetes 1.32 还在写 Webhook? 你已经 OUT 了！](https://mp.weixin.qq.com/s?__biz=MzI5ODQ2MzI3NQ==&mid=2247513735&idx=1&sn=e5f844df272b5bb691382fb5f324cbbd&chksm=ed0baa783029653a0e13882cd76ef2dc29eb75ada3a7d213c07a3bff3829abd230cad5d8f340&scene=126&sessionid=1734429261#rd)
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
2. Kubernetes 1.35 发布团队 <https://github.com/kubernetes/sig-release/blob/master/releases/release-1.35>
3. Kubernetes 1.35 变更日志 <https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.35.md>
4. Kubernetes 1.35 主题讨论 <https://github.com/kubernetes/sig-release/discussions/2903>
