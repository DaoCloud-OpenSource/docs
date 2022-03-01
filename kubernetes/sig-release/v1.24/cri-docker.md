# CRI-Dockerd

Kubernetes v1.24 将于今年 4月正式发布,  Docker Shim 被移除了, CRI Dockerd 来啦!!!

如果您现在 Kubernetes 节点还是用 Docker 那您将无法升级到 1.24 版本, 因为 Docker Shim 已经被移除了.

您可以选择从 Docker 切换到 Containerd, 不过所有 Pod 都需要重建一次

这里给您提供另外的选择, 从 Docker Shim 切换到 CRI-Dockerd

## CRI-Dockerd 是什么呢?

CRI-Dockerd 其实就是被移除的 Docker Shim 独立出来的一个项目为了解决历史遗留的节点升级 Kubernetes 的问题

我们发布了一个 CRI Docker 的安装卸载脚本, 方便您维护您的集群

您可以把您的使用 Docker 的 Kubernetes 升级到 1.24 版本, 只要您的节点提前切换到 CRI Dockerd 即可

## 从 Docker Shim 切换到 CRI Docker
``` bash
wget -O install.sh https://raw.githubusercontent.com/klts-io/setup-cri-dockerd/main/install.sh
./install.sh
```

## 回退
``` bash
wget -O uninstall.sh https://raw.githubusercontent.com/klts-io/setup-cri-dockerd/main/uninstall.sh
./uninstall.sh
```

CRI-Dockerd 项目地址 https://github.com/Mirantis/cri-dockerd

安装脚本项目地址 https://github.com/klts-io/setup-cri-dockerd


## DockerShim 移除 常见问题

### 为什么要移除 Dockershim
Kubernetes 的早期版本仅适用于特定的容器运行时：Docker 引擎。后来，Kubernetes 增加了对使用其他容器运行时的支持。创建 CRI 标准是为了实现编排器（如 Kubernetes）和许多不同的容器运行时之间的互操作性。 Docker Engine 没有实现该接口 (CRI)，因此 Kubernetes 项目创建了兼容代码来帮助过渡，并使 dockershim 代码成为 Kubernetes 本身的一部分。

dockershim 代码一直是一个临时解决方案（因此得名：shim）。您可以在 [Dockershim Removal Kubernetes Enhancement Proposal](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2221-remove-dockershim) 中阅读有关社区讨论和规划的更多信息。事实上，维护 dockershim 已经成为 Kubernetes 维护者的沉重负担。

此外，在这些较新的 CRI 运行时中实现了与 dockershim 基本不兼容的功能，例如 cgroups v2 和用户命名空间。取消对 dockershim 的支持将允许这些领域的进一步发展。

### 我还能在 Kubernetes 1.23 中使用 Docker Engine 吗？
是的，如果使用 Docker Engine 作为运行时，1.20 在 kubelet 启动时打印的一个警告日志。您在 1.23 之前的所有版本中都会看到此警告。 dockershim 将在 Kubernetes 1.24 移除。

### 我仍然可以使用 Docker Engine 作为我的容器运行时吗？
如果您在自己的 PC 上使用 Docker 来开发或测试容器：没有任何变化。无论您为 Kubernetes 集群使用什么容器运行时，您仍然可以在本地使用 Docker。容器使这种操作性成为可能。

如果是 Kubernetes 中还是要继续使用 Docker 可以尝试该适配器 [cri-dockerd](https://github.com/Mirantis/cri-dockerd) 和我们为您提供的维护脚本 [setup-cri-dockerd](https://github.com/klts-io/setup-cri-dockerd)。 

### 我现有的容器镜像是否仍然有效？
是的，从 docker build 生成的图像将适用于​​所有 CRI 实现。您现有的所有镜像仍将完全相同不需要做任何改动。

### 私人镜像是否仍然有效？
是的。所有 CRI 运行时都支持在 Kubernetes 中使用的相同的 pull secrets 配置，无论是通过 PodSpec 还是 ServiceAccount。

### Docker 和容器是一回事吗？
Docker 普及了 Linux 容器模式，并在开发底层技术方面发挥了重要作用，但是 Linux 中的容器已经存在了很长时间。容器生态系统已经发展到比 Docker 广泛得多。 OCI 和 CRI 等标准帮助许多工具在我们的生态系统中发展壮大，其中一些替代了 Docker 的某些方面，而另一些则增强了现有功能。

### 今天有没有人在生产中使用其他运行时的例子？
所有 Kubernetes 项目生成的工件（Kubernetes 二进制文件）在每个版本中都经过验证。

此外，kind 项目使用 containerd 已经有一段时间了，并且已经看到其用例的稳定性有所提高。每天都会多次使用 Kind 和 containerd 来验证对 Kubernetes 代码库的任何更改。其他相关项目也遵循类似的模式，展示了其他容器运行时的稳定性和可用性。
