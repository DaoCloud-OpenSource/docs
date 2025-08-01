# 迎风破浪的三只熊——Kubernetes v1.34 发布，看点全解析

太平洋时间 2025 年 8 月 27 日，主题为 Of Wind & Will 的 Kubernetes v1.34 正式发布。

此版本距离上版本发布时隔 4 个月，是 2025 年的第二个版本。

与之前的版本类似，Kubernetes v1.34 版本引入了多项新的 Alpha、Beta 和 Stable 的功能。一贯交付高质量版本的承诺凸显了 Kubernetes 社区的开发周期实力和社区的活跃支持。

此版本包含 58 项改进。在这些改进功能中，有 23 项已晋升为 Stable，有 24 项进入 Beta 阶段，有 11 项为 Alpha 阶段。

## 发布主题和 Logo

Kubernetes v1.34 的发布主题是 Of Wind & Will：它描述地是三只熊驾着一艘木船远航，船帆上绘有熊掌与舵轮的旗帜，迎风破浪，行于大海之上。

![](./k8s-1.34.png)

## GA 和稳定的功能

GA 全称 General Availability，即正式发布。Kubernetes 的进阶路线通常是 Alpha、Beta、Stable (即 GA)、Deprecation/Removal 这四个阶段。下文择取了部分特性详述。如果对其他特性感兴趣，请移步至具体的 KEP 页面了解进展和详情。

### KEP-4033 节点 cgroup 驱动程序的自动配置

一直以来，为新运行的 Kubernetes 集群配置正确的 cgroup 驱动程序是用户的一个痛点。 在 Linux 系统中，存在两种不同的 cgroup 驱动程序：cgroupfs 和 systemd。 过去，kubelet 和 CRI 实现（如 CRI-O 或 containerd）需要配置为使用相同的 cgroup 驱动程序， 否则 kubelet 会报错并退出。 这让许多集群管理员头疼不已。

在 v1.28.0 版本中，SIG Node 社区引入了 KubeletCgroupDriverFromCRI 特性门控， 它指示 kubelet 向 CRI 实现询问使用哪个 cgroup 驱动程序。在两个主要的 CRI 实现（containerd 和 CRI-O）增加对该功能的支持这段期间，Kubernetes 经历了几次小版本发布之后，我们等待每个 CRI 实现发布的主要版本并在主流操作系统中打包，此功能已于 Kubernetes 1.34.0 正式发布。

除了设置特性门控之外，集群管理员还需要确保 CRI 实现版本足够新：

- containerd：最低 v2.0.0
- CRI-O：最低 v1.28.0

然后，集群管理员应该确保配置其 CRI 实现使用了预期的 cgroup 驱动程序。

<!-- 下面这段文字, 公众号排版时, 重点标记出来 -->

Kubernetes SIG Node 已正式同意最终支持 containerd v1.y 的时间表。v1.35.0 将是最后一个提供此支持的 Kubernetes 版本。为了帮助管理员实现未来的过渡，新的检测机制现已推出。您可以使用 kubelet_cri_losing_support 监控指标来确定集群中是否有节点正在使用即将过期的 containerd 版本。如果此指标的版本标签为 1.36.0，则表示该节点的 containerd 运行时版本不足以满足即将到来的需求。因此，管理员需要在将 kubelet 升级到 v1.36.0 之前，先将 containerd 升级到 v2.0 或更高版本。

### KEP-4381 DRA 的核心功能

动态资源分配 (DRA) 提供了一种灵活的方式来对 Kubernetes 集群中的 GPU 或自定义硬件等设备进行分类、请求和使用。自 v1.30 版本发布以来，DRA 一直基于使用对 Kubernetes 核心不透明的结构化参数来声明设备。这项增强功能的灵感源自存储卷的动态配置。具有结构化参数的 DRA 依赖于一组支持 API 类型： resource.k8s.io 下的 ResourceClaim、DeviceClass、ResourceClaimTemplate 和 ResourceSlice API 类型，同时使用新的 resourceClaims 字段扩展了 Pod 的 .spec 字段。resource.k8s.io/v1 API 已升级为稳定版本，现在默认可用, 但DRA 的核心功能仍可通过 `DynamicResourceAllocation` 特性门控来禁用该功能。更多信息请见[DRA 官方文档](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)。

<!-- 下面这段文字, 公众号排版时, 重点标记出来 -->

使用 DRA 时，请注意以下故障排除步骤：

1. 当工作负载没有按预期启动时，可以从 Job → Pods → ResourceClaims 逐级查看，并使用 kubectl describe 检查每一层的对象，看看是否有状态字段或事件能解释工作负载未启动的原因。

2. 当创建 Pod 失败并出现 `must specify one of: resourceClaimName, resourceClaimTemplateName` 错误时，请检查 `pod.spec.resourceClaims` 中的每个条目是否恰好设置了这两个字段中的一个。如果检查无误，那么可能是集群中安装了针对 Kubernetes 1.32 之前版本 API 构建的 Pod 变更（mutating）webhook。这种情况下，请与集群管理员一起排查确认。

Kubernetes 社区提供的 DRA 驱动实现：

- [CNI DRA 驱动 (实验阶段)](https://github.com/kubernetes-sigs/cni-dra-driver): Kubernetes 中网络接口的配置和报告缺乏标准化的原生方法, 该项目旨在填补这一空白，它结合了 CNI API 和 DRA API，提供了一种更现代化、更集成 Kubernetes 的机制来配置和报告 Pod 上的网络接口。
- [CPU DRA 驱动 (实验阶段)](https://github.com/kubernetes-sigs/dra-driver-cpu): 由于 Kubelet 的限制, 它不允许在同一个节点上同时存在 独占（pinned）核心 和 共享（shared）核心。该项目旨在解决这一限制，通过 DRA框架能提供这种精细化的资源控制，还能将节点相关的属性暴露给调度器。

其他 DRA 驱动实现：

- [Net DRA 驱动 (Google)](https://github.com/google/dranet): 它使用动态资源分配 (DRA) 为 Kubernetes 中要求苛刻的应用程序提供高性能网络。
- [GPU DRA 驱动 (NVIDIA)](https://github.com/NVIDIA/k8s-dra-driver-gpu): 它提供灵活而强大的 GPU 分配和动态重新配置。
- [Net DRA 驱动 (DaoCloud)](https://github.com/spidernet-io/spiderpool/issues/4754): 它提供灵活而强大的网络分配和动态重新配置, 包括静态网卡, 动态网卡的分配以及报告 RDMA 拓扑相关信息。更多信息请见[KubeCon 快闪演讲](https://www.youtube.com/watch?v=Cx5c-IueP78)。

### 更新总览

- [KEP-24 为 Kubernetes 添加 AppArmor 支持](https://kep.k8s.io/24)
- [KEP-647 kube-apiserver 支持 Tracing 功能](https://kep.k8s.io/647)
- [KEP-1790 允许用户通过减少请求的容量大小, 从卷扩容失败中恢复](https://kep.k8s.io/1790)
- [KEP-2340 监视缓存提供一致的数据视图, 提高 Kubernetes 对于 Get 和 List 请求的可扩展性和性能](https://kep.k8s.io/2340)
- [KEP-2381 Kubelet 支持 Tracing 功能](https://kep.k8s.io/2381)
- [KEP-2400 节点 Swap 内存支持](https://kep.k8s.io/2400)
- [KEP-3331 添加结构化身份验证配置](https://kep.k8s.io/3331)
- [KEP-3751 允许用户在配置/绑定持久卷后动态修改卷属性, 例如 IOP 和吞吐量, 可变卷属性名及值规范来自存储系统, 不同存储系统间可能不通用](https://kep.k8s.io/3751)
- [KEP-3902 节点控制器负责向节点添加污点并驱逐 Pod。但在 1.29 版本之后，基于污点的驱逐实现已从节点控制器移出，并被移至一个单独的、独立的组件，名为 taint-eviction-controller。用户可以选择通过在 kube-controller-manager 中设置 `--controllers=-taint-eviction-controller` 来禁用基于污点的驱逐。](https://kep.k8s.io/3902)
- [KEP-3939 为 Job API 添加了一个新字段，允许用户指定是否希望控制平面在前一个 Pod 开始终止时立即创建新 Pod（现有行为），或者仅在现有 Pod 完全终止时创建新 Pod（新的、 可选行为）](https://kep.k8s.io/3939)
- [KEP-3960 PreStop 生命周期钩子添加新的休眠操作，允许容器在终止前暂停指定的时长, 提供一种更直接的方式来管理优雅关闭，改进容器的整体生命周期管理，并在 Pod 终止期间处理来自尚未完成端点终止的客户端的新连接。](https://kep.k8s.io/3960)
- [KEP-4033 增加了容器运行时指示 kubelet 使用哪个 cgroup 驱动程序的能力, 无需在 kubelet 配置中指定 cgroup 驱动程序, 并消除 kubelet 和运行时之间 cgroup 驱动程序配置不一致的可能性](https://kep.k8s.io/4033)
- [KEP-4247 调度器通过将 QueueingHint 引入 EventsToRegister，调度队列根据 QueueingHint 的结果对 Pod 重新排队, 有助于减少无用的调度重试，从而提高调度吞吐量](https://kep.k8s.io/4247)
- [KEP-4369 环境变量中允许使用特殊字符](https://kep.k8s.io/4369)
- [KEP-4381 借助结构化参数，kube-scheduler 和 Cluster Autoscaler 可以自行处理和模拟声明分配，而无需依赖第三方驱动程序](https://kep.k8s.io/4381)
- [KEP-4427 dnsConfig.searches 支持对 DNS 搜索字符串进行宽松验证，允许包含 “.” 和 “_” 字符](https://kep.k8s.io/4427)
- [KEP-4568 启用弹性监视缓存初始化以避免控制平面过载](https://kep.k8s.io/4568)
- [KEP-4601 鉴权属性将扩展为包括列表、监视和删除集合中的字段选择器和标签选择器。这将允许鉴权者在做出鉴权决定时使用这些选择器](https://kep.k8s.io/4601)
- [KEP-4633 仅允许对配置的端点进行匿名身份验证](https://kep.k8s.io/4633)
- [KEP-4818 PreStop Hook 的睡眠操作允许零值](https://kep.k8s.io/4818)
- [KEP-5080 根据逻辑依赖关系和安全考虑，引入命名空间删除的删除优先级，优先删除某些资源类型。该特性被移植到 1.30 - 1.32 版本](https://kep.k8s.io/5080)
- [KEP-5100 在 Windows kube-proxy 中支持 DSR 和 Overlay 网络](https://kep.k8s.io/5100)
- [KEP-5116 Kubernetes API 服务器提供的集合（列表响应）实现流式编码，流式编码过程可以显著减少内存使用量，从而提高可扩展性和成本效益](https://kep.k8s.io/5116)


## 进入 Beta 阶段的功能

Beta 阶段的功能是指那些已经经过 Alpha 阶段的功能，且在 Beta 阶段中添加了更多的测试和验证，通常情况下是默认启用的。下文择取了部分特性详述。如果对其他特性感兴趣，请移步至具体的 KEP 页面了解进展和详情。

### KEP-3962 变更性准入策略

它提供了一种声明式、进程内变更准入 webhook 的替代方案。此功能利用 CEL 的对象实例化和 JSON Patch 策略，并结合服务器端应用的合并算法。这允许管理员直接在 API 服务器中定义变异规则，从而大大简化了准入控制。变异准入政策在 v1.32 中作为 alpha 版本引入，并在 v1.34 中升级为 beta 版本。

以下是一个 MutatingAdmissionPolicy 的示例。 此策略会将变更新建的 Pod，为其添加一个边车容器（如果以前没有边车容器的话）。

```yaml
apiVersion: admissionregistration.k8s.io/v1beta1
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
    - patchType: "ApplyConfiguration"
      applyConfiguration:
        expression: >
          Object{
            spec: Object.spec{
              initContainers: [
                Object.spec.initContainers{
                  name: "mesh-proxy",
                  image: "mesh/proxy:v1.0.0",
                  args: ["proxy", "sidecar"],
                  restartPolicy: "Always"
                }
              ]
            }
          }          

```

### KEP-4876 可变附加卷的最大数量

之前，节点上报告的挂载容量与实际挂载容量不匹配可能会导致永久性调度错误和工作负载卡住。这种情况发生在 CSI 驱动程序启动后卷槽被占用时，导致 kube-scheduler 将有状态 Pod 分配给缺乏必要容量的节点。这种不匹配可能由多种情况造成，例如：

1. 绕过 Kubernetes 和 CSI 驱动程序, 由管理员或外部控制器手动附加卷。
2. 当一个节点上使用多个 CSI 驱动程序时，一个驱动程序的操作会影响其他驱动程序的可用容量。

这些情况可能会导致 CSI 驱动程序报告的初始容量随着时间的推移变得不准确，从而导致调度程序基于过时的信息做出决策。这会导致 Pod 被调度到容量不足的节点，最终卡在 ContainerCreating 状态。通过使         `CSINode.Spec.Drivers[*].Allocatable.Count` 字段可变并引入动态更新机制，我们可以确保调度程序始终拥有更准确地代表实际状态的信息，从而显著提高有状态 Pod 调度的可靠性。

### 更新总览


- [KEP-127 支持用户命名空间，进程能够在具有与主机不同的用户和组 ID 的 pod 中运行。](https://kep.k8s.io/127)
- [KEP-740  提供服务帐户令牌与外部签名进行集成方法，易于更新令牌和提供安全性](https://kep.k8s.io/740)
- [KEP-1287 支持了 Pod 资源原地升级（In-Place Update）功能，允许在不重新创建 Pod 或重新启动容器的情况下更改资源](https://kep.k8s.io/1287)
- [KEP-2837 允许在 Pod 级别设置资源请求和限制](https://kep.k8s.io/2837)
- [KEP-3015 添加一种方法来向 kube-proxy 发出信号，表明它应该尽可能将流量传输到本地端点，以提高效率](https://kep.k8s.io/3015)
- [KEP-3104 实现 .kuberc 文件，将用户偏好设置与集群配置分开](https://kep.k8s.io/3104)
- [KEP-3157 Informer 可以获取数据流来启动缓存](https://kep.k8s.io/3157)
- [KEP-3243 Deployment 滚动更新过程中的调度优化, 滚动更新时，PodTopologySpread 调度策略可以区分调度 Pod 标签的值, 使分布更加均匀](https://kep.k8s.io/3243)
- [KEP-3476 提供了一组新的 API, 支持一致的组快照，该快照允许在同一时间点从多个卷拍摄快照，以实现写入顺序一致性](https://kep.k8s.io/3476) 
- [KEP-3695 PodResources API 的增强，包括由动态资源分配 (DRA) 驱动程序分配的资源, 以允许节点监控代理访问。](https://kep.k8s.io/3695)
- [KEP-3962 添加了使用 CEL 表达式声明的更改准入策略，作为更改准入 Webhook 的替代方案](https://kep.k8s.io/3962)
- [KEP-4205 集成 PSI 指标，并使用这些指标设置节点条件，用于防止在遇到严重资源限制的节点上调度 Pod](https://kep.k8s.io/4205)
- [KEP-4412 允许通过绑定到工作负载的服务账户，生成用于授权镜像拉取的令牌，避免需要长期存在的持久密钥](https://kep.k8s.io/4412)
- [KEP-4639 新增基于 OCI 镜像的只读卷, 并支持子目录挂载](https://kep.k8s.io/4639)
- [KEP-4802 初步支持 Windows 节点的体面关闭功能](https://kep.k8s.io/4802)
- [KEP-4816 允许工作负载通过 ResourceClaim 指定可配合使用的替代设备类型，而无需用户修改编排清单](https://kep.k8s.io/4816)
- [KEP-4876 允许 CSI 驱动程序可以在运行时动态调整可附加卷的最大数量](https://kep.k8s.io/4876)
- [KEP-4988 改进 apiserver watch cache 以提供分页](https://kep.k8s.io/4988)
- [KEP-5018 允许在特权模式下创建 ResourceClaims 和 ResourceClaimTemplates，以授予对其他用户正在使用的设备的访问权限，以执行管理任务，例如监控设备的运行状况或状态](https://kep.k8s.io/5018)
- [KEP-5067 利用现有的元数据 Generation 字段，并在 Pod 状态中添加一个新的 status.observedGeneration 字段，允许 Pod 状态字段表示当前哪些 Pod 更新反映在 Pod 状态中](https://kep.k8s.io/5067)
- [KEP-5073 使用 validation-gen 生成验证代码，实现 Kubernetes 原生类型的声明式验证。](https://kep.k8s.io/5073)
- [KEP-5109 引入一个新的 CPU 管理器静态策略选项，该选项尽最大努力通过 L3（最后一级）缓存来调整 CPU 资源](https://kep.k8s.io/5109)
- [KEP-5278 除了使用 NominatedNodeName 来指示正在进行的抢占之外，调度器还可以在绑定周期开始时指定它，以向其他组件显示预期的 Pod 位置。此外，其他组件也可以将 NominatedNodeName 放在待处理的 Pod 上，以指示该 Pod 优先调度到特定节点](https://kep.k8s.io/5278)
- [KEP-5299 kube-scheduler 调度期间的所有 API 调用都异步进行, 使调度周期不受阻塞 API 调用的影响](https://kep.k8s.io/5299)


## 进入 Alpha 阶段的功能

Alpha 阶段的功能是指那些刚刚被引入的功能，这些功能是默认关闭的，需要用户手动开启。

### KEP-5295 KYAML

KYAML 是面向 Kubernetes 的 YAML 子集/编码，旨在提供更安全、更少歧义的 YAML 编码。它可用于编写 Helm Chart 的模版和 Kubernetes 资源清单。下面是它的简单示例，它将输出一个 Pod 的 KYAML 清单：

```yaml
$ kubectl get -o kyaml pods
---
{
  apiVersion: "v1",
  items: [{
    apiVersion: "v1",
    kind: "Pod",
    metadata: {
      annotations: {
        "kubernetes.io/limit-ranger": "LimitRanger plugin set: cpu request for container serve-hostname-xqnl5",
      },
      creationTimestamp: "2025-05-09T21:14:39Z",
      generateName: "hostnames-77b655d8d-",
      generation: 1,
      labels: {
        app: "hostnames",
        pod-template-hash: "77b655d8d",
      },
      name: "hostnames-77b655d8d-qs6sb",
      namespace: "default",
      ownerReferences: [{
        apiVersion: "apps/v1",
        blockOwnerDeletion: true,
        controller: true,
        kind: "ReplicaSet",
        name: "hostnames-77b655d8d",
        uid: "bd95a01f-ad62-4983-a3a4-dd1f9b1ca226",
      }],
      resourceVersion: "37706",
      uid: "afd43648-4a75-4616-ba72-2915b599fef9",
    },
    spec: {
      containers: [{
        image: "k8s.gcr.io/serve_hostname:v1.4",
        imagePullPolicy: "IfNotPresent",
        name: "serve-hostname-xqnl5",
        resources: {
          requests: {
            cpu: "100m",
          },
        },
        terminationMessagePath: "/dev/termination-log",
        terminationMessagePolicy: "File",
        volumeMounts: [{
          mountPath: "/var/run/secrets/kubernetes.io/serviceaccount",
          name: "kube-api-access-g57rz",
          readOnly: true,
        }],
      }],
      dnsPolicy: "ClusterFirst",
      enableServiceLinks: true,
      nodeName: "kubernetes-minion-group-bd63",
      preemptionPolicy: "PreemptLowerPriority",
      priority: 0,
      restartPolicy: "Always",
      schedulerName: "default-scheduler",
      securityContext: {},
      serviceAccount: "default",
      serviceAccountName: "default",
      terminationGracePeriodSeconds: 30,
      tolerations: [{
        effect: "NoExecute",
        key: "node.kubernetes.io/not-ready",
        operator: "Exists",
        tolerationSeconds: 300,
      }, {
        effect: "NoExecute",
        key: "node.kubernetes.io/unreachable",
        operator: "Exists",
        tolerationSeconds: 300,
      }],
      volumes: [{
        name: "kube-api-access-g57rz",
        projected: {
          defaultMode: 420,
          sources: [{
            serviceAccountToken: {
              expirationSeconds: 3607,
              path: "token",
            },
          }, {
            configMap: {
              items: [{
                key: "ca.crt",
                path: "ca.crt",
              }],
              name: "kube-root-ca.crt",
            },
          }, {
            downwardAPI: {
              items: [{
                fieldRef: {
                  apiVersion: "v1",
                  fieldPath: "metadata.namespace",
                },
                path: "namespace",
              }],
            },
          }],
        },
      }],
    },
    status: {
      conditions: [{
        lastProbeTime: null,
        lastTransitionTime: "2025-05-09T21:14:41Z",
        status: "True",
        type: "PodReadyToStartContainers",
      }, {
        lastProbeTime: null,
        lastTransitionTime: "2025-05-09T21:14:39Z",
        status: "True",
        type: "Initialized",
      }, {
        lastProbeTime: null,
        lastTransitionTime: "2025-05-09T21:14:41Z",
        status: "True",
        type: "Ready",
      }, {
        lastProbeTime: null,
        lastTransitionTime: "2025-05-09T21:14:41Z",
        status: "True",
        type: "ContainersReady",
      }, {
        lastProbeTime: null,
        lastTransitionTime: "2025-05-09T21:14:39Z",
        status: "True",
        type: "PodScheduled",
      }],
      containerStatuses: [{
        allocatedResources: {
          cpu: "100m",
        },
        containerID: "containerd://49d6e2a850d3569145de92a1138b377a904954b48b23905af00b15ec5393de77",
        image: "k8s.gcr.io/serve_hostname:v1.4",
        imageID: "sha256:b56f22170982bc46dee99f7157be69dce727d3b89ee5698d15fda7a14a124200",
        lastState: {},
        name: "serve-hostname-xqnl5",
        ready: true,
        resources: {
          requests: {
            cpu: "100m",
          },
        },
        restartCount: 0,
        started: true,
        state: {
          running: {
            startedAt: "2025-05-09T21:14:41Z",
          },
        },
        volumeMounts: [{
          mountPath: "/var/run/secrets/kubernetes.io/serviceaccount",
          name: "kube-api-access-g57rz",
          readOnly: true,
          recursiveReadOnly: "Disabled",
        }],
      }],
      hostIP: "10.40.0.9",
      hostIPs: [{
        ip: "10.40.0.9",
      }],
      phase: "Running",
      podIP: "10.64.3.4",
      podIPs: [{
        ip: "10.64.3.4",
      }],
      qosClass: "Burstable",
      startTime: "2025-05-09T21:14:39Z",
    },
  }],
  kind: "List",
  metadata: {
    resourceVersion: "",
  },
}
```

- [KEP-3721 支持从文件实例化容器的环境变量。该文件必须位于 emptyDir 卷中。该环境变量文件可以由 emptyDir 卷中的 initContainer 创建。kubelet 将从 emptyDir 卷中的指定文件实例化容器中的环境变量，但不会挂载该文件。请注意，该卷只会挂载到环境变量生产者容器（initContainer），而使用该环境变量的普通容器不会挂载该卷。](https://kep.k8s.io/3721)
- [KEP-4317 为 Pod 提供了使用 mTLS 向 kube-apiserver 进行身份验证的内置功能。它由易于外部项目复用的组件构建而成，以便使用 X.509 证书提供更多功能](https://kep.k8s.io/4317)
- [KEP-4680 引入了一种机制，使 Kubelet 能够监控并报告通过动态资源分配 (DRA) 分配的设备的健康状况。这为集群运维人员和控制器提供了直接在 Pod 状态中查看设备健康状况的关键可见性，从而能够更好地排除故障并修复由硬件故障导致的 Pod 故障。](https://kep.k8s.io/4680)
- [KEP-4762 在 podSpec 中引入了一个新的字段 hostnameOverride, 允许用户将任意的完全限定域名 (FQDN) 设置为 Pod 的主机名](https://kep.k8s.io/4762)
- [KEP-4944 添加 Pod 安全准入（PSA）策略，以阻止在 ProbeHandler 和 LifecycleHandler 中设置 .host 字段, 防止 SSRF 攻击](https://kep.k8s.io/4944)
- [KEP-5004 通过 DRA 驱动程序处理扩展资源请求, 使现有应用程序无需修改即可运行。](https://kep.k8s.io/5004)
- [KEP-5007 DRA 引入一种机制 ( BindingConditions 和 BindingFailureConditions)，允许调度程序推迟 Pod 绑定，直到确认外部资源已准备就绪](https://kep.k8s.io/5007)
- [KEP-5075 允许 ResourceSlice 中的单个设备在不同的独立 ResourceClaims 之间共享，这些 ResourceClaims 由可分配容量值进行管理（前提是驱动程序允许）。这使得 DRA 设备可以由独立的工作负载共享，而不是像现在这样只能由协作的工作负载共享](https://kep.k8s.io/5075)
- [KEP-5295 kubectl 引入 KYAML，一种更安全、更少歧义的 YAML 子集/编码](https://kep.k8s.io/5295)
- [KEP-5311 放宽 Service 的名称验证, 允许服务名称以数字开头。](https://kep.k8s.io/5311)
- [KEP-5307 引入了容器重启规则，以便 kubelet 可以在容器退出时应用这些规则。这将允许用户配置容器的特殊退出代码，使其被视为非故障状态，即使 Pod 的 restartPolicy=Never 也能原地重启容器。](https://kep.k8s.io/5307)

## DaoCloud 社区贡献

上半年，DaoCloud 参与多个问题修复和功能研发，并取得了不少成就。其中：

- [徐俊杰（pacoxu）](https://github.com/pacoxu) 当选 CNCF TAG Workloads Foundation Chair 席位
- [要海峰（windsonsea）](https://github.com/windsonsea) 和 [李信 (my-git9)](https://github.com/my-git9) 成为 Kubernetes 子项目 [Kueue](https://github.com/kubernetes-sigs/kueue) 文档站的 Reviewer；
- [颜开（yankay）](https://github.com/yankay)成为了 Kubernetes 子项目 [LeaderWorkerSet](https://github.com/kubernetes-sigs/lws) 的项目 Reviewer。
### 活动更新

介绍即将举行的 Kubernetes 和云原生活动，包括 KubeCon + CloudNativeCon、KCD，希望大家积极参与 Kubernetes 社区以及相关活动。

- - [**KubeCon + CloudNativeCon 北美 2025**](https://events.linuxfoundation.org/kubecon-cloudnativecon-north-america/): 2025年11月10-13日 | 亚特兰大, 美国
- 
[**KCD - Kubernetes Community Days: 杭州**](https://sessionize.com/kcd-hangzhou-and-oicd-2025/): 2025年11月14日 | 杭州 

KCD 杭州议题征集 9月21日截止，欢迎大家投稿。

## 发行说明

上述内容就是 Kubernetes v1.34 的主要更新和内容啦，更多的发布说明可以查看 [Kubernetes v1.34 版本的完整详细信息](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.34.md)。

我们下次版本发布时再见！

## 历史文档

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
2. Kubernetes 1.34 发布团队 <https://github.com/kubernetes/sig-release/blob/master/releases/release-1.34>
3. Kubernetes 1.34 变更日志 <https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.34.md>
4. Kubernetes 1.34 主题讨论 <https://github.com/kubernetes/sig-release/discussions/2806>
