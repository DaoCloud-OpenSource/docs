# Kubernetes v1.36 正式发布稿（精简版草案）

> 本文为正式发布用稿草案，基于截至 **2026-03-30** 的上游公开信息整理。Kubernetes v1.36 计划于 **2026-04-22（周三）** 发布，GA 当日请以官方 CHANGELOG 与 release notes 为准。

Kubernetes v1.36 的主线很清晰：一方面继续推进高风险能力的退场与迁移，另一方面在安全、可扩展性和资源利用效率上做实质增强。

## 本版最需要关注的三件事

1. 明确迁移路径：`Service.spec.externalIPs` 启动弃用，`gitRepo` 卷驱动在 v1.36 永久禁用。
2. 强化安全基线：ServiceAccount Token 外部签名、细粒度 Kubelet API 鉴权等能力进入稳定阶段。
3. 面向大规模集群：API Machinery、调度与可观测性方向有一批面向规模化场景的新能力或升级。

## 升级前必须处理的弃用与移除

### `Service.spec.externalIPs` 开始弃用

- v1.36 起开始出现弃用告警，计划在 v1.43 移除。
- 原因是长期存在安全风险（如 CVE-2020-8554 相关攻击面）。
- 建议迁移到 `LoadBalancer`、`NodePort` 或 `Gateway API`。

参考：<https://kep.k8s.io/5707>

### `gitRepo` 卷驱动永久禁用

- 自 v1.11 起已弃用。
- v1.36 起禁用且不可重新启用。
- 建议迁移为 `initContainer` 拉取、镜像构建期打包，或使用外部 `git-sync`。

参考：<https://kep.k8s.io/5040>

### Ingress NGINX 已退役（生态风险提示）

- Ingress NGINX 在 2026 年 3 月退役，不再提供后续修复与安全更新。
- 现有部署不会立刻失效，但建议尽快迁移到 Gateway API 或其他受维护方案。

参考：<https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/>

## v1.36 核心增强（重点节选）

### 安全与鉴权

- **KEP-740（Stable）**：ServiceAccount Token 支持外部签名（KMS/HSM）。
- **KEP-2862（Stable）**：Kubelet API 细粒度鉴权，支持按请求类型做最小权限控制。
- **KEP-5018（Stable）**：DRA ResourceClaim 支持 AdminAccess，便于设备运维与巡检。
- **KEP-5284（Beta）**：Constrained Impersonation，支持“受约束的自我降权式模拟”。

### API Machinery

- **KEP-3962（Stable）**：Mutating Admission Policy（CEL）升级到 GA。
- **KEP-5073（Stable）**：原生类型声明式校验能力增强（validation-gen + CEL）。
- **KEP-5589（Stable）**：完成 gogo protobuf 依赖移除。
- **KEP-4020（Beta）**：Mixed Version Proxy（默认开启）改善升级期跨版本请求路由。
- **KEP-5793（Alpha）**：Manifest-Based Admission Control Config。
- **KEP-5866（Alpha）**：Server-side Sharded List/Watch。
- **KEP-5647（Alpha）**：Stale Controller Mitigation。

### 存储与节点侧能力

- **KEP-3476（GA 目标）**：Volume Group Snapshot，提升多卷一致性快照与恢复能力。
- **KEP-5538（GA 目标）**：CSI 通过 `secrets` 字段接收 SA token。
- **KEP-4876（GA 目标）**：Mutable `CSINode` Allocatable，提升 attach 限制动态感知。
- **KEP-1710（GA 目标）**：SELinux relabeling 挂载优化，降低启动延迟。
- **KEP-5055（Beta）**：DRA 设备污点/容忍，避免设备被误用。
- **KEP-4815**：DRA 可分区设备，支持单加速器多逻辑单元共享。

### 网络与可扩缩

- **KEP-4858（Beta）**：IP/CIDR 校验收紧，拒绝非规范和歧义格式。
- **SIG Scalability 信号**：官方可支持资源大小测试上限由 800MB 提升至 1.5GB。

## 来自 #2958 的新增更新（截至 2026-03-30）

相比 3 月中旬的早期快照，`kubernetes/sig-release#2958` 在 3 月下旬新增了多项重点：

- **SIG Auth（3/26）**：补充了 KEP-740、KEP-2862、KEP-5018 的稳定化信息。
- **SIG Autoscaling（3/30）**：新增 KEP-2021（HPA scale to zero，Alpha）亮点。
- **SIG Scheduling（3/30）**：强调 Workload Aware Scheduling（WAS）为当前重点方向，涵盖 KEP-5832、5732、5729、5710、5547、4671。
- **SIG Instrumentation（3/30）**：KEP-4827/4828 升级到 Beta（默认启用），并新增 KEP-5808（Prometheus Native Histogram，Alpha）。
- **SIG API Machinery（3/30）**：新增一组稳定化与 Alpha 能力说明，重点包括 KEP-3962、5073、5589、4020、5793、5866、5647。

参考：<https://github.com/kubernetes/sig-release/discussions/2958>

## 升级检查清单（建议）

1. 扫描并替换 `externalIPs` 与 `gitRepo` 相关配置。
2. 盘点 Ingress 入口能力，制定 Ingress NGINX 迁移计划。
3. 在 staging 环境验证 API Machinery 与调度相关变更（尤其是自研 controller / scheduler 插件）。
4. 对 CSI/DRA 工作负载做回归，重点验证快照恢复、资源编排和设备分配路径。
5. 升级当日对照 `CHANGELOG-1.36.md` 与 release notes 做最终差异核验。

## 参考链接

- <https://kubernetes.io/blog/2026/03/30/kubernetes-v1-36-sneak-peek/>
- <https://github.com/kubernetes/sig-release/discussions/2958>
- <https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/README.md>
- <https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.36.md>
- <https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/release-notes/release-notes-draft.md>
