# 目录
- [目录](#目录)
- [Docker Docs](#docker-docs)
- [Why Docker](#why-docker)
    - [Docker vs. Virtual Machines](#docker-vs-virtual-machines)
    - [Docker优点](#docker优点)
- [Docker 架构](#docker-架构)
    - [1. Client客户端](#1-client客户端)
    - [2. Docker Daemon](#2-docker-daemon)
    - [3. Host主机](#3-host主机)
    - [4. Image镜像](#4-image镜像)
    - [5. Container容器](#5-container容器)
    - [6. 镜像分层](#6-镜像分层)
    - [7. Volume数据卷](#7-volume数据卷)
    - [8. Registry注册中心](#8-registry注册中心)
- [Docker image 创建](#docker-image-创建)
    - [1. 创建Dockerfile](#1-创建dockerfile)
    - [2. 使用命令创建镜像](#2-使用命令创建镜像)
- [Docker 常用命令](#docker-常用命令)
  - [参考原文](#参考原文)


# Docker Docs

链接：https://docs.docker.com/

# Why Docker

### Docker vs. Virtual Machines

| 特性 | 容器 | 虚拟机 |
| --- | --- | --- |
| 启动 | 秒级 | 分钟级 |
| 硬盘使用 | 一般为MB | 一般为GB |
| 性能 | 接近原生 | 弱于 |
| 系统支持量 | 单机支持上千个容器 | 一般几十个 |


### Docker优点

- 更高效的利用系统资源
- 更快速的启动时间
- 一致的运行环境
- 持续交付和部署
- 更轻松的迁移
- 更轻松的维护和扩展

# Docker 架构

![Docker Framework](/docker/images/7868545-060cc40d94102469.jpg)

### 1. Client客户端

Docker 是一个客户端-服务器（C/S）架构程序。Docker 客户端只需要向 Docker 服务器或者守护进程发出请求，服务器或者守护进程将完成所有工作并返回结果。Docker 提供了一个命令行工具 Docker 以及一整套 RESTful API。你可以在同一台宿主机上运行 Docker 守护进程和客户端，也可以从本地的 Docker 客户端连接到运行在另一台宿主机上的远程 Docker 守护进程。

### 2. Docker Daemon

宏观来讲，mainDaemon()完成创建一个daemon进程，并使其正常运行。

从功能的角度来说，mainDaemon()实现了两部分内容：

- 第一，创建Docker运行环境
- 第二，服务于Docker Client，接收并处理相应请求。

### 3. Host主机

一个物理或者虚拟的机器用于执行 Docker 守护进程和容器。

*ps. 守护进程（daemon）：生存期长的一种进程，没有控制终端。它们常常在系统引导装入时启动，仅在系统关闭时才终止。*

### 4. Image镜像

Docker 镜像就是一个 Linux 的文件系统（Root FileSystem），这个文件系统里面包含可以运行在 Linux 内核的程序以及相应的数据。

通过镜像启动一个容器，一个镜像就是一个可执行的包，其中包括运行应用程序所需要的所有内容：包含代码，运行时间，库，环境变量和配置文件等。

Docker 把 App 文件打包成为一个镜像，并且采用类似多次快照的存储技术，可以实现：

- 多个 App 可以共用相同的底层镜像（初始的操作系统镜像）
- App 运行时的 IO 操作和镜像文件隔离
- 通过挂载包含不同配置/数据文件的目录或者卷（Volume），单个 App 镜像可以用来运行无数个不同业务的容器。

### 5. Container容器

镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的类和实例一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。

| Docker | 面向对象 |
| --- | --- |
| 容器 | 对象 |
| 镜像 | 类 |

### 6. 镜像分层

Docker 支持通过扩展现有镜像，创建新的镜像。实际上，Docker Hub 中 99% 的镜像都是通过在 base 镜像中安装和配置需要的软件构建出来的。

![pic](https://mrhelloworld.com/resources/articles/docker/12180844322018196a29c55c8de4a2.png)

*ps. Base镜像*</br>
*base 镜像有两层含义。不依赖其他镜像，从 scratch 构建。其他镜像可以之为基础进行扩展。所以，能称作 base 镜像的通常都是各种 Linux 发行版的 Docker 镜像，比如 Ubuntu, Debian, CentOS 等。*

当容器启动时，一个新的可写层被加载到镜像的顶部。这一层通常被称作“容器层”，“容器层”之下的都叫“镜像层”。

![pic](https://mrhelloworld.com/resources/articles/docker/121808444920186f41cc40362cc7ef.png)

所有对容器的改动 - 无论添加、删除、还是修改文件都只会发生在容器层中。只有**「容器层是可写的，容器层下面的所有镜像层都是只读的」**

| 文件操作 | 说明 |
| --- | --- |
| 添加文件 | 在容器中创建文件时，新文件被添加到容器层中。 |
| 读取文件 | 在容器中读取某个文件时，Docker 会从上往下依次在各镜像层中查找此文件。一旦找到，立即将其复制到容器层，然后打开并读入内存。 |
| 修改文件 | 在容器中修改已存在的文件时，Docker 会从上往下依次在各镜像层中查找此文件。一旦找到，立即将其复制到容器层，然后修改之。 |
| 删除文件 | 在容器中删除文件时，Docker 也是从上往下依次在镜像层中查找此文件。找到后，会在容器层中「记录下此删除操作」。（只是记录删除操作） |

### 7. Volume数据卷

实际上我们的容器就好像是一个简易版的操作系统，只不过系统中只安装了我们的程序运行所需要的环境，前边说到我们的容器是可以删除的，那如果删除了，容器中的程序产生的需要持久化的数据怎么办呢？容器运行的时候我们可以进容器去查看，容器一旦删除就什么都没有了。

所以数据卷就是来解决这个问题的，是用来将数据持久化到我们宿主机上，与容器间实现数据共享，简单的说就是将宿主机的目录映射到容器中的目录，应用程序在容器中的目录读写数据会同步到宿主机上，这样容器产生的数据就可以持久化了，比如我们的数据库容器，就可以把数据存储到我们宿主机上的真实磁盘中。

### 8. Registry注册中心

Docker 用 Registry 来保存用户构建的镜像。Registry 分为公共和私有两种。Docker 公司运营公共的 Registry 叫做 Docker Hub。用户可以在 Docker Hub 注册账号，分享并保存自己的镜像。

Docker 公司提供了公共的镜像仓库 [https://hub.docker.com](https://hub.docker.com/)（Docker 称之为 Repository）提供了庞大的镜像集合供使用。

一个 Docker Registry 中可以包含多个仓库（Repository）；每个仓库可以包含多个标签（Tag）；每个标签对应一个镜像。

通常，一个仓库会包含同一个软件不同版本的镜像，而标签对应该软件的各个版本。我们可以通过 **「<仓库名>:<标签>」** 的格式来指定具体是这个软件哪个版本的镜像。如果不给出标签，将以 **「latest」** 作为默认标签。

# Docker image 创建

### 1. 创建Dockerfile

```
FROM golang:alpine

WORKDIR /src

COPY . .

RUN go mod download (没有mod文件不写这一行）

RUN go build -o main main.go

ENTRYPOINT [ "./main" ]
```

### 2. 使用命令创建镜像

    docker build --tag=<image's name>:<tag>

# Docker 常用命令

1. docker images 查看镜像列表
2. docker ps 查看容器列表
3. docker stop <container ID> 停止目标容器
4. docker start <container ID> 开启目标容器
5. docker restart <container ID> 重启目标容器
6. docker rm <container ID> 删除目标容器
7. docker rmi <image's name> 删除目标镜像
8. docker logs <container ID> 查看目标容器日志


## 参考原文

1. https://docs.docker.com/
2. https://www.cnblogs.com/mrhelloworld/p/docker2.html
