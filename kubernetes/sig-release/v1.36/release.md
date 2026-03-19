# Kubernetes v1.36 预览：GA 前值得关注的关键变化

截至 **2026 年 3 月 18 日（代码冻结日）**，Kubernetes v1.36 已进入发布前的最后阶段。
本文基于上游发布节奏、发布说明草案与增强特性追踪信息，总结对平台团队最有影响的变化。

> 说明：当前为预览稿。上游计划的 v1.36 GA 发布时间为 **2026 年 4 月 22 日**。

## 发布时间线（v1.36）

- 发布周期开始：2026-01-12
- 增强特性冻结：2026-02-11（AoE）
- 代码/测试冻结：2026-03-18（AoE）
- 文档冻结：2026-04-08（AoE）
- 计划 GA：2026-04-22

参考：<https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/README.md>

## 版本速览

基于当前 `v1.36` 里程碑追踪到的增强特性：

- 78 个 tracked 条目
- 18 个稳定版（Stable/GA）目标
- 25 个 Beta 目标
- 31 个 Alpha 目标

活跃方向集中在 SIG Node、SIG Scheduling、SIG API Machinery、SIG Auth、SIG Storage。

## 高影响变化

### 1) API Machinery 与控制面演进

#### Mutating Admission Policies 进入 GA

`MutatingAdmissionPolicy` 是 v1.36 中可见度很高的毕业项，当前草案方向是默认启用。

- KEP：<https://github.com/kubernetes/enhancements/issues/3962>

#### Kubernetes API protobuf 清理持续推进

历史 `gogo protobuf` 依赖的进一步移除，对依赖内部序列化细节的生态组件有兼容性影响。

- KEP：<https://github.com/kubernetes/enhancements/issues/5589>

### 2) 存储与 DRA 持续加强

#### Volume Group Snapshot 目标 GA

对有状态业务（包括训练/推理状态）而言，组快照能力是关键增强。

- KEP：<https://github.com/kubernetes/enhancements/issues/3476>

#### 可变 CSI Node 可分配上限（Mutable CSINode Allocatable）目标 GA

CSI 驱动可动态调整可附加卷上限，降低静态容量信息导致的调度偏差。

- KEP：<https://github.com/kubernetes/enhancements/issues/4876>

#### CSI 通过 secrets 字段接收 SA token 目标 GA

为 CSI 驱动提供可选的更稳妥令牌传递路径。

- KEP：<https://github.com/kubernetes/enhancements/issues/5538>

### 3) 节点与调度能力演进

#### Node log query 目标 GA

提升原生排障效率，减少节点日志查询链路的操作复杂度。

- KEP：<https://github.com/kubernetes/enhancements/issues/2258>

#### 用户命名空间与节点侧能力继续推进

v1.36 中节点/运行时、授权与调度相关变更多，建议将节点能力与插件兼容性一起验证。

- 参考 KEP：<https://github.com/kubernetes/enhancements/issues/127>

### 4) 网络与校验收敛

#### IP/CIDR 校验改进进入 Beta

上游 highlights 已强调更严格的 canonical 校验方向，用于降低歧义与安全风险。

- KEP：<https://github.com/kubernetes/enhancements/issues/4858>

## 升级与兼容性注意事项

结合当前 changelog 与 release notes 草案，生产升级前建议重点确认：

1. 调度器插件接口：`PreBind` 并行化、`PostFilterResult` 等变化，需自研插件逐项回归。
2. kubeadm 与 flex-volume：kubeadm 已移除集成 flex-volume 支持，仍依赖该路径的环境需提前迁移或自定义方案。
3. `gitRepo` 卷插件：当前方向为默认禁用且不可重新启用。
4. API/client-go 行为变化：informer 与 API machinery 的正确性/性能改动较多，需对自研 controller/operator 做回归测试。

## AI-Infra 行动清单

面向 AI 平台 / AI-Infra 团队（GPU、大规模训练与推理混合负载），建议按以下清单推进：

1. 运行时基线盘点：统一核对 `containerd`、`runc/crun`、cgroup 模式，标记需要先升级运行时的节点池。
2. 调度插件兼容性验证：针对自定义调度插件执行 v1.36 canary，用真实训练/推理任务验证 `PreBind` 与 `PostFilter` 相关逻辑。
3. DRA 能力体检：检查 DRA 相关 CRD/控制器与设备插件，确认现有加速器资源编排流程可平滑过渡。
4. 存储恢复演练：对训练检查点、模型仓库、推理状态卷执行快照与恢复演练，验证恢复时间与一致性目标。
5. 准入策略迁移评估：梳理现有 mutating webhooks，评估可迁移到 `MutatingAdmissionPolicy` 的策略，降低 webhook 运行风险。
6. 控制器回归测试：重点覆盖依赖 client-go informer 行为和 protobuf 序列化假设的控制器。
7. 网络数据规范化：提前清理非规范 IP/CIDR 配置，避免校验收紧后触发发布窗口风险。
8. 分阶段发布策略：采用 `dev -> staging -> 小流量生产` 的渐进升级，设置训练/推理 SLO 与回滚闸门。

## 正式发布前仍需确认

由于当前是预览稿，正式发布前建议再核对：

- 最终主题与 logo 资产
- `CHANGELOG-1.36.md` 的 GA 最终内容
- 发布说明中对破坏性变更/升级注意事项的最终措辞
- 是否出现晚期已知问题（known issues）

## 参考链接

- <https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/README.md>
- <https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/release-team.md>
- <https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/links.md>
- <https://github.com/kubernetes/kubernetes/milestone/69>
- <https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.36.md>
- <https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/release-notes/release-notes-draft.md>
- <https://github.com/kubernetes/sig-release/discussions/2958>

