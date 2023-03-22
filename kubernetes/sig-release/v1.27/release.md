太平洋时间 2023 年 4 月 11 日，Kubernetes 1.27 正式发布。此版本距离上版本发布时隔 4 个月，是 2023 年的第一个版本。

新版本包含 x 个 enhancements「1.25 版本  个、1.26 版本  个」其中  个功能升级为稳定版， 个已有功能进行优化，另有多达  个全新功能以及  个废弃的功能。

Kubernetes 1.27 版本在多个方面实现重大突破，需要注意的是 详情可以参考本文第五小节「升级注意事项」。

本版本包含了多个 的功能，比如
相关的重要功能会在下一小节进行详细介绍。

### 1. 重要功能

**[Alpha] 容器组支持用户命名空间 User Namespace Support**
**[Alpha] kubelet 支持保存容器当前状态（Checkpoint)**

### 2. 其他需要了解的功能

-  E2E。

### 3. DaoCloud 主要参与功能 

本次发布中， DaoCloud 重点贡献了 sig-node，sig-scheduling 和 kubeadm 相关内容，具体功能点如下：

- 

此外， DaoCloud 还参与了十多个问题修复，在 v1.27 发布过程中总计贡献  个提交，详情请见[贡献列表](https://www.stackalytics.io/cncf?project_type=cncf-group&release=all&metric=commits&module=github.com/kubernetes/kubernetes&date=118)（该版本的两百多位贡献者中有来自 DaoCloud 的  位）。也正是因为「DaoCloud 道客」在 Kubernetes 社区的持续贡献，
1. kwok 成为社区热点
2. 要海峰成为 sig-doc-zh maintainer；（梦姣也是）
3. kubecon 预告： sig-scheduling deep dive + kubeadm deep dive。

### 4. 版本标志

本次发布的主题是 。 Kubernetes 是。
![logo](kubernetes-1.27.png)


### 5. 升级注意事项

本节主要介绍 v1.27 中 API 变化，以及功能的移除以及废弃，废弃的功能通常会在 1-2 个版本之后移除。更多详情请查看[Kubernetes 在 v1.27 中移除的特性和主要变更](https://kubernetes.io/zh-cn/blog/2023/03/17/upcoming-changes-in-kubernetes-v1-27/)

** **

**其他需要注意的变化**

- 废弃

### 6. 历史文档

- [Kubernetes 正式发布 v1.26，稳定性显著提升](https://mp.weixin.qq.com/s/qwzmeIM4INz-_BK_gbwOxw)
- [Kubernetes 1.25 正式发布，多方面重大突破](https://mp.weixin.qq.com/s/aRmLBYpk0MhLJAwY85DyuA)
- [Kubernetes 1.24 走向成熟的 Kubernetes](https://mp.weixin.qq.com/s/vqH8ueaZeEeZbx_axNVSjg)
- [Kubernetes 1.23 正式发布，有哪些增强？](https://mp.weixin.qq.com/s/A5GBv5Yn6tQK_r6_FSyp9A)
- [Kubernetes 1.22 颠覆你的想象：可启用 Swap，推出 PSP 替换方案，还有……](https://mp.weixin.qq.com/s/9nH2UagDm6TkGhEyoYPgpQ)
- [Kubernetes 1.21 震撼发布 | PSP 将被废除，BareMetal 得到增强](https://mp.weixin.qq.com/s/amGjvytJatO-5a7Nz4BYPw)

### 7. 参考

1. https://github.com/kubernetes-sigs/kwok/