# Kubernetes v1.36 正式发布稿（对齐 release announcement 最新草稿）

> 写作基线：截至 **2026-04-20** 的上游公开信息，重点对齐 `kubernetes/website#55151`（updated at 2026-04-19）与 `kubernetes/sig-release#2958`。v1.36 计划发布时间为 **2026-04-22（周三）**，请以 GA 当日 CHANGELOG 与 release notes 为准。

## 一、升级必读（前瞻内容精简）

### 弃用与移除

- `Service.spec.externalIPs`：v1.36 开始弃用并给出告警，计划 v1.43 移除。建议迁移到 `LoadBalancer`、`NodePort` 或 `Gateway API`（KEP-5707）。
- `gitRepo` 卷驱动：v1.36 起永久禁用且不可重新启用。建议迁移为 `initContainer`、镜像构建打包或外部 `git-sync`（KEP-5040）。

### 生态风险提示

- Ingress NGINX 已于 2026 年 3 月 24 日退役，不再发布后续修复与安全更新。现网可继续运行，但应尽快推进迁移。

## 二、与 #55151 最新版本对齐的发布主线

根据 `kubernetes/website#55151` 当前草稿，v1.36 发布叙事已形成以下结构：

- 增强总量：**70**。
- 升级到 Stable：**18**。
- 升级到 Beta：**25**。
- 升级到 Alpha：**25**。

### Spotlight（官方草稿当前重点）

- Stable：Fine-grained Kubelet API authorization（KEP-2862）。
- Beta：Resource Health Status（KEP-4680）。
- Alpha：Workload Aware Scheduling（WAS）系列能力（含 KEP-4671/5547/5832/5732/5710）。

## 三、重点功能解读

### 重点一：细粒度 Kubelet API 鉴权进入稳定（KEP-2862）

该能力在 v1.36 稳定化后，`kubelet` API 不再只能依赖粗粒度授权模型，运维与观测场景可按请求类型（如 `exec`、`logs`、`metrics`、`port-forward`）做更细粒度权限控制。对平台团队来说，这直接降低了“为了一个调试能力而放开整组高风险权限”的常见问题，是最小权限治理的一次实质推进。

### 重点二：ServiceAccount Token 外部签名稳定化（KEP-740）

v1.36 将外部签名能力推进到稳定阶段，允许集群把 ServiceAccount Token 的签名职责委托给外部签名系统（如 HSM/KMS），而不必在控制面本地长期持有签名私钥。这一点对合规与密钥集中治理要求较高的组织尤其关键。

结合 #55151 当前稿，稳定化后的关键价值不仅是“签名外置”，还包括更完整的控制面集成路径：apiserver 能发现并缓存外部签名方公钥、校验非本地签发 token，并与既有认证/RBAC 流程保持一致。对于已有外部 IAM 体系的平台，这意味着可以把 Kubernetes token 签发纳入统一密钥生命周期、轮换与审计机制。

### 重点三：Volume Group Snapshot 稳定化（KEP-3476）

多卷应用的恢复一致性长期是生产痛点。v1.36 将 Volume Group Snapshot 推进到稳定后，平台可以对一组相关 PVC 做一致性快照与恢复，避免“单卷可恢复、多卷逻辑不一致”的问题。对训练平台、数据库及复杂有状态系统，这项能力更适合直接纳入 RTO/RPO 演练，而不是只做功能可用性验证。

## 四、其他重要更新（结合 #55151 + #2958）

### Stable 侧

- Mutating Admission Policies（KEP-3962）进入稳定，进一步降低常见场景对外置 webhook 的依赖。
- Declarative Validation（KEP-5073）进入稳定，校验规则更声明式、可维护。
- 移除 gogo protobuf 依赖（KEP-5589）完成稳定落地。
- Node Log Query（KEP-2258）稳定化，节点日志排障链路更统一。
- User Namespaces（KEP-127）稳定化，容器内身份与宿主身份隔离更易落地。
- PSI（KEP-4205）与 OCI VolumeSource（KEP-4639）稳定化，提升节点资源观测与数据分发路径能力。

### Beta 侧

- IP/CIDR 校验改进（KEP-4858）收紧非规范与歧义写法。
- `kubectl` 用户偏好与集群配置分离（KEP-3104，`kuberc`）。
- Suspended Job 资源可变（KEP-5440）。
- Constrained Impersonation（KEP-5284）。
- DRA 一组能力进入 beta（设备分区/污点容忍/状态可见性/扩展资源路径等）。
- Statusz（KEP-4827）与 Flagz（KEP-4828）进入 beta 并默认启用。
- Mixed Version Proxy（KEP-4020）进入 beta。
- Memory QoS（KEP-2570）在 cgroup v2 语义下进一步完善。

### Alpha 侧

- Staleness mitigation for controllers（KEP-5647）。
- HPA Scale to Zero（KEP-2021）持续推进。
- DRA Alpha 新增方向（与 Workload/可见性/原生资源适配相关）。
- Native Histogram Support（KEP-5808）。
- Manifest-based Admission Control Config（KEP-5793）。
- CRI List Streaming（KEP-5825）。

### 来自 #2958 的增量补充（3/31 与 4/2）

- Node Log Query 在 SIG Windows 侧被明确标注为 v1.36 稳定能力，强调了跨 Linux/Windows 的统一节点日志排障路径。
- SIG Node 强调 v1.36 的 DRA 是多能力线并行推进，同时补充了 CRI list streaming、Memory QoS、User Namespaces、PSI、OCI VolumeSource 等节点侧能力成熟度提升信号。

### 升级风险前置：SELinux 兼容准备

虽然 SELinux 相关潜在破坏性变更主要面向后续版本窗口讨论，但其准备动作应在 1.36 周期尽早完成。建议提前盘点工作负载的 SELinux 标签与策略假设，避免升级窗口集中暴露兼容性问题。

> 说明：EvictionRequest API、HPA fallback external metrics、Deployment Pod Replacement Policy 相关稿件当前仍以占位探索为主，暂不作为本版 release 主线展开。

## 五、建议的升级动作

1. 先完成 `externalIPs` 与 `gitRepo` 使用面扫描，建立迁移清单和截止时间。
2. 入口层尽快制定 Ingress NGINX 迁移方案与回退预案。
3. 在 staging 验证：Kubelet 鉴权策略、外部 token 签名链路、准入策略变更路径。
4. 对有状态关键业务执行组快照恢复演练，量化 RTO/RPO。
5. 对大规模节点场景补充验证：Node Log Query、PSI、Memory QoS、CRI List Streaming 相关行为。
6. GA 当日按最终 CHANGELOG/release notes 做一次差异复核再发版。

## 六、参考链接

- <https://github.com/kubernetes/website/pull/55151>
- <https://github.com/kubernetes/sig-release/discussions/2958>
- <https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/README.md>
- <https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.36.md>
- <https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/release-notes/release-notes-draft.md>
- <https://kep.k8s.io/740>
- <https://kep.k8s.io/2862>
- <https://kep.k8s.io/3476>
