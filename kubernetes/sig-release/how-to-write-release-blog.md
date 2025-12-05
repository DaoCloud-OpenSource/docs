# 如何写 Kubernetes 发布 Blog

本文档介绍如何撰写 Kubernetes 版本发布博客，包括撰写原因、信息收集方法和注意事项。

## 为什么需要写额外的发布 Blog？

官方已经有 release blog，也会有中文翻译，那么为何还需要额外写一篇呢？尤其是早期 Kubernetes release blog 有时候比较简略，但新版本为何还需要额外撰写？

主要原因有两个：

### 1. 结合国内需求和客户场景

我们会结合目前国内以及我们客户的需求，高亮一些内容。相对而言，社区公有云厂商较多，也会涉及到不少 cloud provider 相关的内容，这部分对我们来说并不是重点。我们会更关注：

- 国内用户关注的功能特性
- 私有云和混合云场景下的重要更新
- 与我们客户实际需求相关的功能改进

### 2. 介绍国内活动和贡献情况

重点会介绍：

- KubeCon 中国站的相关内容
- KCD (Kubernetes Community Days) 活动
- DaoCloud 和国内社区的贡献情况

## 如何收集信息

撰写发布博客时，可以从以下渠道收集信息：

### 1. 官方 Major Theme / Highlights

访问 [Kubernetes SIG Release Discussions](https://github.com/kubernetes/sig-release/discussions) 查看官方收集的主要主题和亮点。

### 2. Known Issues

关注已知问题，了解版本中需要注意的问题：

- [Known Issues 列表](https://github.com/kubernetes/kubernetes/issues?q=is%3Aissue%20state%3Aclosed%20known%20issue%20in%3Atitle)

### 3. Release Candidate (RC) 版本的 CHANGELOG

查看 RC 版本的变更日志：

- [CHANGELOG 目录](https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG)

### 4. Release 看板

关注 Release 看板了解进展：

- [Kubernetes Release 看板](https://github.com/orgs/kubernetes/projects/229)

### 5. Sneak Peek 博客

近期 release team 还会发布 sneak peek 博客，提前介绍部分重要更新，例如：

- [Kubernetes v1.35 Sneak Peek](https://kubernetes.io/blog/2025/11/26/kubernetes-v1-35-sneak-peek/)

### 6. Release Logo

Release logo 要等官方 blog 发布。如果关注 [kubernetes/website](https://github.com/kubernetes/website) 仓库的 PR，可能可以提前几个小时了解到。

### 7. Feature Blog

关注功能特性博客。个别功能如果非常重要，可以适当参考对应 blog 的描述：

- [Feature Blog PR 列表](https://github.com/kubernetes/website/pulls?q=is%3Apr+is%3Aopen+label%3Aarea%2Fblog)

## 撰写注意事项

### ⚠️ 重要提醒

**尽量在 release 正式发布后，再发布我们的文章**，尤其是包含 logo 等信息。按说是需要尊重社区，做好保密的。

### 内容建议

1. **突出重点**：根据国内用户和客户需求，选择性地突出重要功能
2. **深度解读**：对于重要特性，可以提供比官方 blog 更详细的解读
3. **实践指导**：结合实际使用场景，提供配置示例和最佳实践
4. **中文本地化**：使用通俗易懂的中文表述，避免生硬翻译

### 文档结构参考

可以参考已有的版本发布文档，例如：

- [v1.34 Release](./v1.34/release.md)
- [v1.33 Release](./v1.33/release.md)
- [v1.32 Release](./v1.32/release.md)

典型的文档结构包括：

1. **版本概述**：发布时间、主题、整体改进数量
2. **发布主题和 Logo**：介绍版本主题和 logo 含义
3. **GA 和稳定的功能**：详细介绍晋升为稳定版的功能
4. **Beta 阶段的功能**：介绍进入 Beta 测试的新功能
5. **Alpha 阶段的功能**：简要介绍新的实验性功能
6. **弃用和移除**：说明即将弃用或已移除的功能
7. **升级注意事项**：提醒用户升级时需要注意的事项
8. **社区贡献**：介绍国内社区和 DaoCloud 的贡献

## 相关资源

- [Kubernetes 官方博客](https://kubernetes.io/blog/)
- [Kubernetes Release 页面](https://kubernetes.io/releases/)
- [SIG Release](https://github.com/kubernetes/sig-release)
- [Kubernetes Enhancement Proposals (KEPs)](https://github.com/kubernetes/enhancements)

## 总结

撰写 Kubernetes 发布博客是一项需要细心和耐心的工作。通过收集全面的信息、结合国内实际需求、提供深度解读，我们可以为中文用户提供更有价值的内容，帮助他们更好地理解和使用 Kubernetes 的新版本。
