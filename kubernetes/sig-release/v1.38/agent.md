# Kubernetes v1.38 Agent 视角解读：Pod 成为可复用的执行环境，沙箱隔离与权限护栏再进一步

作者：（待补充）（2026 年 11 月 25 日）

Agent 工作负载和传统在线服务很不一样：一次会话、一次工具调用就需要一个隔离的执行环境，生命周期短、并发高；里面跑的往往是模型生成的代码，应当按不可信代码对待；同时它又需要身份，去调用工具、模型和集群本身。

社区里已经有 [kubernetes-sigs/agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox) 在上层提供 Sandbox、SandboxTemplate、SandboxClaim、SandboxWarmPool 这些抽象。Kubernetes v1.38（预计 2026 年 12 月 16 日发布）则在 Pod、节点和 API Server 这几层补上了一批更底层的原语。先说结论：

1. **更快**：动态容器（Alpha）可以往运行中的 Pod 里增删容器，Pod 可以当作预热好、可复用的执行环境；Pod 级资源和 Pod 级原地调整（GA）提供了资源边界。
2. **更隔离**：网络隔离 Pod、Pod 级 PID 限制、从 OCI 仓库分发 seccomp profile、非特权容器可写 cgroup（均为 Alpha），叠加 v1.36 已 GA 的 user namespaces。
3. **更可控**：当 Agent 自己去操作集群时，kuberc 的凭证插件 allowlist 随 kuberc 进入 GA，WebSocket 方式的 exec 必须有 `create` 权限，条件授权（Alpha）和基于清单文件的准入配置（Beta）能把权限和护栏收得更紧。

需要说明的是，这篇文章里的多数新增能力还是 Alpha，默认关闭；运行时要求因特性而异，网络隔离、OCI seccomp、可写 cgroup 和 cgroup namespace 等能力还需要匹配的 CRI/运行时支持。它们适合先在测试集群验证，不建议直接上生产。本文按 2026 年 11 月 25 日可查到的 v1.38 代码冻结后状态整理；正式发布前仍可能变化，一切以最终 CHANGELOG 和 Release Notes 为准。

> 同系列：[Kubernetes v1.38 前瞻：弃用移除、升级必读与重点增强](prerelease.md)、[Kubernetes v1.38 AI Infra 解读](ai-infra.md)。

## 目录

- 一张表看重点
- 一、Pod 成为可复用的执行环境
- 二、隔离：让不可信代码跑得更放心
- 三、身份与信任：Agent 调用工具和模型时“我是谁”
- 四、Agent 操作集群时的权限与护栏
- 五、高频创建销毁的沙箱：控制面与节点
- 试用与升级注意

## 一张表看重点

| Agent 平台要解决的问题 | v1.38 的变化 | KEP | v1.38 阶段 |
| --- | --- | --- | --- |
| 冷启动慢，想复用预热好的 Pod | 动态容器：通过 `pods/dynamic` 增删主容器 | [5972](https://kep.k8s.io/5972) | Alpha |
| 给新会话挂工作目录、凭据、工具包 | 动态 node-local 临时卷（含 `image` 卷） | [6395](https://kep.k8s.io/6395) | Alpha |
| 给整个沙箱一个资源上限，并且能按需伸缩 | Pod 级资源；Pod 级原地调整 | [2837](https://kep.k8s.io/2837)、[5419](https://kep.k8s.io/5419) | GA |
| 不可信代码不能访问集群网络 | 网络隔离 Pod：`spec.podNetwork: Isolated` | [6313](https://kep.k8s.io/6313) | Alpha |
| 防止 fork 炸弹拖垮节点 | Pod 级 PID 限制：`spec.resources.limits.pids` | [6063](https://kep.k8s.io/6063) | Alpha |
| seccomp profile 的分发与版本管理 | 从 OCI 仓库拉取 seccomp profile | [6061](https://kep.k8s.io/6061) | Alpha |
| 沙箱里还要跑容器（构建、测试） | 非特权容器可写 cgroup；可选择是否隔离 cgroup namespace | [5474](https://kep.k8s.io/5474)、[5714](https://kep.k8s.io/5714) | Alpha |
| 工具服务、模型网关的证书与信任 | `kubernetes.io/service-serving` 签名者等 | [6283](https://kep.k8s.io/6283) | Alpha |
| Agent 调 kubectl 时不执行来路不明的凭证插件 | kuberc 凭证插件 allowlist | [3104](https://kep.k8s.io/3104) | GA |
| 按对象内容收紧 Agent 的权限 | 条件授权 | [5681](https://kep.k8s.io/5681) | Alpha |
| Agent 删不掉的平台护栏 | 基于清单文件的准入配置 | [5793](https://kep.k8s.io/5793) | Beta |
| 沙箱高频创建销毁，控制器要扛得住 | WatchList；服务端分片 List/Watch | [3157](https://kep.k8s.io/3157)、[5866](https://kep.k8s.io/5866) | GA；Beta |

## 一、Pod 成为可复用的执行环境

### 1. 动态容器（KEP-5972，Alpha）

这是 v1.38 里和 Agent 关系最直接的一项。它把调度、初始化和执行解耦：Pod 先调度好、初始化好，需要执行任务时再往里放容器，用完再拿掉。

- **API**：新增 `pods/dynamic` 子资源，一次请求就能向运行中的 Pod 添加或移除主容器，也可以同时调整资源。另有一个只读的 `pods/allocated` 子资源，返回 kubelet 实际分配后的 Pod spec。已添加但还没被 kubelet 分配的容器，状态显示为 `Waiting`，reason 为 `Unallocated`。
- **门控**：`DynamicContainers`，需要在 kube-apiserver 和 kubelet 上开启。
- **场景**：KEP 给出的场景包括 Agent 的 warm pool（在预分配、预初始化好的 Pod 里启动短生命周期的工具执行环境）、Ray 和 Slurm 这类框架把一个 Pod 当作资源预算再细分调度，以及原地升级 sidecar。长期目标是从请求到代码开始执行在 100ms 以内。
- **限制**：只能增删主容器（不含 init 容器）；Pod 必须处于 Running 且已完成初始化；不能改变 Pod 的 QoS 类别；不允许特权容器和 hostPort；不支持 Windows 和开启 swap 的 Pod；同名容器要等前一个完全移除后才能复用。
- **权限**：默认的 `admin`、`edit`、`view` 角色都不包含 `pods/dynamic` 的 update 权限，需要显式授权。

一个 warm pool 可以大致这样工作：

1. 预先创建一批 Pod，只放一个常驻的 supervisor 容器，并用 Pod 级资源设定总预算。
2. 新会话到来时，通过 `pods/dynamic` 加入执行容器，同时挂上本次会话需要的卷（见下一节）。
3. 会话结束后移除执行容器和卷，Pod 回到池子里。

同一个 Pod 里的容器默认共享网络和 IPC 命名空间；`pods/dynamic` 本身不会为每个容器建立租户隔离边界。

**安全与准入提醒**：自定义 webhook 或准入策略若只匹配 `pods`，不会自动匹配 `pods/dynamic`。v1.38 的 fail-closed 准入检查会在某个控制器或策略本来会处理普通 Pod 创建/更新、却没有覆盖动态子资源时拒绝该动态请求，而不是让它绕过校验；仍应显式更新匹配规则，并验证 namespace、object selector 等条件下的行为。

### 2. 动态 node-local 临时卷（KEP-6395，Alpha）

- **能力**：通过同一个 `/dynamic` 子资源，动态添加或移除 `emptyDir`、`configMap`、`secret`、`projected` 和 OCI `image` 卷，不需要重建 Pod。
- **门控**：`DynamicNodeLocalEphemeralVolumes`。
- **限制**：新加的卷需要由通过动态容器新加入的容器挂载。需要注意 KEP 描述的“修改已有容器的 `volumeMounts` 后由 kubelet 重建容器”并未进入 v1.38 首个 Alpha：合入的实现 PR 明确推迟了可变 mount，当前实现会拒绝修改已有容器的 `volumeMounts`。它也不支持把卷热插到正在运行的容器里。

对 Agent 来说，几种卷正好对应几类需求：`emptyDir` 当作会话的工作目录，`secret` 或 `projected` 放本次会话的凭据和配置，`image` 卷把打包成 OCI 镜像的工具或技能以只读方式挂进来（image 卷本身在 v1.36 已 GA）。`secret` 和 `projected` 卷尤其需要按租户隔离；保守做法是在上一个会话的容器结束、卷移除完成后，才把 Pod 交给其他租户复用。这是运维隔离建议，不是 Kubernetes 对跨租户复用提供的安全保证。

### 3. 资源边界：Pod 级资源与原地调整（GA）

动态容器的前提是 Pod 有一个总的资源预算，后加入的容器共享这个预算。v1.38 中 Pod 级资源和 Pod 级原地调整都进入 GA：

- 用 Pod spec 中的 `spec.resources` 设置整个沙箱的 CPU、内存和 hugepages 上限。
- 某个工具需要更多资源时，可以通过 `pods/resize`（或在 `pods/dynamic` 请求里一起）原地调整 Pod 级的 CPU 和内存，不必重建 Pod。原地调整不会改变 QoS 类别，只支持 cgroup v2。

## 二、隔离：让不可信代码跑得更放心

隔离的基础仍然是 RuntimeClass（gVisor、Kata 等）和 user namespaces。user namespaces 在 v1.36 已经 GA，在 Pod spec 中设置 `hostUsers: false` 即可使用。v1.38 在此之上又补了几块。

### 1. 网络隔离 Pod（KEP-6313，Alpha）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: code-runner
spec:
  podNetwork: Isolated   # 默认为 Default
  containers:
  - name: runner
    image: python:3.13-slim
    command: ["sleep", "infinity"]
```

- `spec.podNetwork: Isolated` 的 Pod 运行在只有 loopback 接口的网络命名空间里，不接入 Pod 网络，没有 Pod IP，也永远不会被选进 Service。
- 这类 Pod 不能使用 hostPort，也不能使用依赖网络的探针（HTTP、TCP、gRPC），需要改用 exec 探针。
- 调度器只会把它放到容器运行时上报了新 CRI 能力 `RuntimeFeatures.pod_network_isolated` 的节点上。门控为 `NetworkIsolatedPods`。

NetworkPolicy 依赖 CNI 实现和策略本身写得对不对；`Isolated` 是从根上不给 Pod 接入集群网络，更适合纯计算的代码解释器类沙箱。需要专用网络的场景（比如只走 RDMA），可以再通过 DRA 等方式提供非默认网络。

### 2. Pod 级 PID 限制（KEP-6063，Alpha）

模型生成的代码写出死循环 fork 并不罕见。以前 PID 限制只能在节点上通过 kubelet 的 `podPidsLimit` 统一设置，v1.38 允许每个 Pod 单独设置：

```yaml
spec:
  resources:
    limits:
      cpu: "2"
      memory: 4Gi
      pids: "512"   # Pod 级 PID 上限
```

- 实际生效的是 Pod 设置值和节点 `podPidsLimit` 中较小的那个；只能设置 `limits.pids`，设置 `requests.pids` 会被 API Server 拒绝。
- 门控为 `PerPodPIDLimit`，需要 cgroup v2。KEP 的动机里直接点名了 Agent Sandbox 这类 AI Agent 运行时。

### 3. 从 OCI 仓库分发 seccomp profile（KEP-6061，Alpha）

```yaml
securityContext:
  seccompProfile:
    type: OCI
    oci:
      ref: registry.example.com/sandbox/seccomp@sha256:<digest>
```

- 以前自定义 seccomp profile 必须提前分发到每个节点（`Localhost` 类型），通常要靠 DaemonSet、烤进节点镜像或 Security Profiles Operator。现在可以把 profile 和沙箱镜像一起放在镜像仓库里，按 digest 引用，还可以通过可选的 `oci.baseProfile` 指定基础 profile。
- `oci.ref` 必须是带 registry 主机名的规范 digest 引用，不能使用 tag；容器运行时按镜像拉取的凭据规则（节点凭据、ServiceAccount 或 Pod 的 `imagePullSecrets`）拉取 profile。特权容器不能使用 OCI 类型的 seccomp profile。
- 门控为 `SecurityProfileOCI`，需要容器运行时实现新的 CRI 接口（`PullSecurityProfile` 等 RPC，以及 `seccomp_profile_oci` 运行时能力）。AppArmor 计划在 Beta 阶段支持。

### 4. 沙箱里再跑容器：可写 cgroup 与 cgroup namespace（KEP-5474、KEP-5714，Alpha）

Coding Agent 经常需要在沙箱里构建镜像、跑测试容器，以前往往只能给 `privileged: true`。

- **非特权容器可写 cgroup（KEP-5474，`CgroupOptions`）**：在 cgroup v2 节点上，设置 `securityContext.cgroupOptions.mountMode: ReadWrite` 后，非特权容器可以管理自己的 cgroup 子树。内核的 `nsdelegate` 挂载选项保证它只能管自己的子树。调度器只会把这类 Pod 放到 `status.declaredFeatures` 中包含 `CgroupOptions` 的节点上。
- **可选择是否隔离 cgroup namespace（KEP-5714，`CgroupNamespacesSupport`）**：容器级新增 `hostCgroupNamespace` 字段。为 `false` 时为容器创建以自身 cgroup 为根的新 cgroup namespace，为 `true` 时共享主机的 cgroup namespace，不设置则由运行时决定。这让特权 Pod（比如 BuildKit）也能隔离 cgroup namespace，嵌套容器的资源统计和限制更准确。注意：使用 user namespaces 的 Pod 不能设为 `true`；与 `cgroupOptions.mountMode: ReadWrite` 同时使用时，除非容器是特权容器，否则也不能设为 `true`。

再加上 v1.36 GA 的 user namespaces 和 v1.37 进入 Beta 的 Rootless kubelet，“为了跑嵌套容器而开 privileged”的场景会越来越少。

## 三、身份与信任：Agent 调用工具和模型时“我是谁”

Agent 要调用工具服务（比如集群内的 MCP Server）、模型网关、向量库，需要可验证的身份和可信的 TLS。Pod 证书（KEP-4317）和 ClusterTrustBundle（KEP-3257）在 v1.37 已经 GA，Pod 可以通过 projected 卷拿到 X.509 证书，用来做 mTLS。v1.38 在此基础上推进了一步（KEP-6283，Alpha）：

- **`kubernetes.io/service-serving` 签名者（`KubeServingCertificatesSigner`）**：自动为 Pod 签发服务端证书。证书的 DNS SAN 可以是 Pod 自己的 FQDN，也可以是它引用的 Service 的 FQDN。集群内的工具服务因此可以直接拿到被集群广泛信任的服务端证书。
- **`KubeAPIServerWorkloadsTrust`**：可以选择用集群级的 ClusterTrustBundle 作为 Pod 内 kube-apiserver 服务端证书的信任来源，而不是依赖分发到每个命名空间的 ConfigMap。

另外，v1.38 在后量子密码方向也有进展：Pod 证书可以请求使用 ML-DSA 算法签名（Beta，`PodCertificateMLDSA`）；CertificateSigningRequest 支持 ML-DSA 密钥签名的 PKCS#10 请求（`CertificateSigningRequestMLDSA`）；ServiceAccount token 也可以用 ML-DSA 密钥签发（`ServiceAccountTokenMLDSA`）。这些是独立的能力和门控，不会自动把集群现有证书或 token 的签名算法切换为 ML-DSA。

一个容易忽略的点：沙箱默认不应该拿到访问集群 API 的凭据。agent-sandbox 控制器创建沙箱 Pod 时，如果没有显式设置 `automountServiceAccountToken`，会将其设为 `false`；自己搭平台时也建议这样做。

## 四、Agent 操作集群时的权限与护栏

运维 Agent、SRE Agent、Coding Agent 经常直接调用 kubectl 或 Kubernetes API。v1.38 中相关的变化：

### 1. kuberc GA：限制可执行的凭证插件（KEP-3104）

kubeconfig 里的 `exec` 凭证插件本质上是执行一个外部程序。如果 Agent 会处理用户提供的或来源不可信的 kubeconfig，这就是一个代码执行入口。kuberc 可以限制 kubectl 允许执行哪些凭证插件：

```yaml
apiVersion: kubectl.config.k8s.io/v1
kind: Preference
credentialPluginPolicy: Allowlist   # 可选 AllowAll、DenyAll、Allowlist
credentialPluginAllowlist:
- command: /usr/local/bin/kubelogin  # 可以是文件名，也可以是完整路径
```

- 不设置 `credentialPluginPolicy` 时等同于 `AllowAll`。
- 迁移到 `v1` 时，allowlist 条目要用 `command`，`v1` 不再支持 `name`。
- kuberc 默认位于 `~/.kube/kuberc`，可以用 `--kuberc` 参数或 `KUBERC` 环境变量指定路径，`KUBERC=off` 可以临时禁用。

**注意**：kuberc 是客户端配置，能控制运行环境的进程可以轻易绕过它，所以它不是安全边界。它适合作为 Agent 运行时的默认安全配置，真正的约束仍然要放在 RBAC、授权和准入上。

### 2. exec 和 attach 需要 `create` 权限

v1.38 中 KEP-4006 进入 GA，`AuthorizePodWebsocketUpgradeCreatePermission` 锁定为开启：通过 WebSocket 发起的 `pods/exec`、`pods/attach` 等请求，需要对应子资源的 `create` 权限，不能再临时关闭这项检查。给 Agent 配角色时，只给真正需要进容器的 Agent 授予 `pods/exec` 的 `create`，不要指望只给 `get` 就能把它限制在只读范围。

### 3. 条件授权（KEP-5681，Alpha）

以前授权只能基于元数据做决定（API 组、资源、命名空间、名字）。条件授权让授权决定可以依赖对象内容（标签和字段）：

- 授权阶段，授权器可以返回 `Conditional`，并附上一组条件；到了验证准入阶段，再基于新旧对象计算这些条件，得出最终的允许或拒绝。
- SubjectAccessReview 相关 API 都扩展了条件，用户可以通过 `kubectl auth can-i` 看到自己受哪些限制；新增的 `AuthorizationConditionsReview` API 供树外授权器计算条件。
- 门控为 `ConditionalAuthorization`，需要通过 AuthorizationConfiguration 配置支持条件的 webhook 授权器，并启用 `AuthorizationConditionsEnforcer` 准入控制器。它只适用于会进入准入阶段的请求（例如 create、update、patch、delete 和 connect），不是对所有读取请求追加字段级限制。

放到 Agent 场景里，就可以表达“这个 Agent 的 ServiceAccount 可以更新 Deployment，但只限带某个标签的对象”这类策略；API Server 的 `AuthorizationConditionsEnforcer` 负责执行授权器返回的条件，无需另行编写一套准入策略来复刻这些约束。

### 4. 基于清单文件的准入配置（KEP-5793，Beta）

- kube-apiserver 启动时从清单目录（准入配置中的 `staticManifestsDir`）加载准入 webhook 和策略（ValidatingAdmissionPolicy 等），并在运行时监视文件变化。门控为 `ManifestBasedAdmissionControlConfig`。
- 这些策略在 API Server 开始处理请求之前就已生效，不能通过 Kubernetes API 修改或绕过，还可以用来保护 API 中的准入配置资源本身。
- v1.38 为 MutatingAdmissionWebhook 增加了 `staticOrder` 字段，决定清单中的变更 webhook 和通过 API 注册的 webhook 谁先执行：`StaticFirst`（默认）、`StaticLast` 或 `ByName`。这个字段只存在于本地准入配置文件里，降级到 v1.37 或更早版本前需要删掉，否则 kube-apiserver 无法启动。

对 Agent 平台来说，这意味着即使某个 Agent 拿到了较大的权限，也删不掉平台的底线策略，比如“禁止特权容器”“禁止 hostPath”。

## 五、高频创建销毁的沙箱：控制面与节点

沙箱数量一多、创建销毁一频繁，压力会落到控制器、API Server 和节点上：

- **WatchList（KEP-3157）GA**：informer 通过流式 watch 获取初始数据，不再依赖分页 LIST，降低 kube-apiserver 的内存峰值。
- **服务端分片 List/Watch（KEP-5866，Beta）**：控制器副本只订阅自己负责的分片，由 API Server 在源头过滤，沙箱控制器这类高事件量的控制器可以真正水平扩展。
- **managedFields opt-out（KEP-5958，Alpha）**：客户端可以选择不接收 `metadata.managedFields`，减少传输和反序列化开销。
- **CRI List 流式接口（KEP-5825，Beta，`CRIListStreaming`）**：单节点有上万个容器时，CRI 的 List 响应可能超过 16 MB 的 gRPC 消息上限；新的流式 RPC 逐条返回结果。它解除单条响应的消息大小限制，但 kubelet 目前仍会收集结果，因此不等于降低 kubelet 的整体内存占用；高密度沙箱节点值得关注。

## 试用与升级注意

1. **门控与运行时**：本文多数能力是 Alpha，默认关闭，需要在 kube-apiserver 和 kubelet 上开启对应门控；网络隔离、OCI seccomp、可写 cgroup、cgroup namespace 还依赖容器运行时实现新的 CRI 能力，先确认 containerd 或 CRI-O 的支持情况。
2. **`pods/dynamic` 的权限和准入**：显式授权；自定义准入 webhook 和策略要覆盖这个子资源。
3. **cgroup v2**：Pod 级 PID 限制、可写 cgroup、Pod 级原地调整都要求 cgroup v2；cgroup v1 支持也将在未来版本移除。
4. **网络隔离 Pod 的探针**：HTTP、TCP、gRPC 探针和 hostPort 都不能用，改用 exec 探针。
5. **user namespaces 与 cgroup namespace**：`hostUsers: false` 的 Pod 不能把 `hostCgroupNamespace` 设为 `true`。
6. **kubectl**：kuberc 迁移到 `v1` 时 `name` 改为 `command`；WebSocket 回退相关的 `KUBECTL_REMOTE_COMMAND_WEBSOCKETS`、`KUBECTL_PORT_FORWARD_WEBSOCKETS` 已移除。
7. **容器运行时版本**：v1.38 计划要求运行时实现 CRI `RuntimeConfig` RPC（containerd 2.0+、CRI-O 1.28+），对应 PR 截至 11 月 25 日仍未合入，详见[通用版前瞻](prerelease.md)。

## 参考

1. KEP-5972：Dynamic Containers <https://kep.k8s.io/5972>
2. `pods/dynamic` 子资源（kubernetes/kubernetes#143715）<https://github.com/kubernetes/kubernetes/pull/143715>
3. Dynamic node-local ephemeral volumes（kubernetes/kubernetes#143709）<https://github.com/kubernetes/kubernetes/pull/143709>
4. Network Isolated Pods（kubernetes/kubernetes#143280）<https://github.com/kubernetes/kubernetes/pull/143280>
5. Per-pod PID limits（kubernetes/kubernetes#139277）<https://github.com/kubernetes/kubernetes/pull/139277>
6. OCI seccomp profile（kubernetes/kubernetes#142511、#142509）<https://github.com/kubernetes/kubernetes/pull/142511>
7. ReadWrite cgroups for unprivileged containers（kubernetes/kubernetes#137568）<https://github.com/kubernetes/kubernetes/pull/137568>
8. KEP-5714 cgroup namespaces（kubernetes/kubernetes#143673）<https://github.com/kubernetes/kubernetes/pull/143673>
9. KEP-6283 service-serving signer（kubernetes/kubernetes#139409）<https://github.com/kubernetes/kubernetes/pull/139409>
10. KEP-6283 kube-apiserver trust via ClusterTrustBundle（kubernetes/kubernetes#142789）<https://github.com/kubernetes/kubernetes/pull/142789>
11. KEP-5681：Conditional Authorization <https://kep.k8s.io/5681>
12. KEP-5793：Manifest Based Admission Control Config <https://kep.k8s.io/5793>
13. kuberc 文档 <https://kubernetes.io/docs/reference/kubectl/kuberc/>
14. Kubernetes v1.36: User Namespaces in Kubernetes are finally GA <https://kubernetes.io/blog/2026/04/23/kubernetes-v1-36-userns-ga/>
15. Agent Sandbox 文档 <https://agent-sandbox.sigs.k8s.io/docs/>
16. Enhancements · v1.38 Release Tracking <https://github.com/orgs/kubernetes/projects/269/views/1>
17. Kubernetes v1.38 CHANGELOG <https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.38.md>
18. Dynamic node-local ephemeral volumes 实现（kubernetes/kubernetes#143709）<https://github.com/kubernetes/kubernetes/pull/143709>
19. Agent Sandbox 默认关闭 ServiceAccount token 自动挂载 <https://github.com/kubernetes-sigs/agent-sandbox/blob/main/extensions/controllers/utils.go>
20. Pod 证书支持 ML-DSA（kubernetes/kubernetes#142108）<https://github.com/kubernetes/kubernetes/pull/142108>
21. CertificateSigningRequest 支持 ML-DSA（kubernetes/kubernetes#142196）<https://github.com/kubernetes/kubernetes/pull/142196>
22. ServiceAccount token 支持 ML-DSA（kubernetes/kubernetes#142397）<https://github.com/kubernetes/kubernetes/pull/142397>
