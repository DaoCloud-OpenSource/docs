# Kubernetes Doc

链接：https://kubernetes.io/zh-cn/docs/home/  

# Why Kubernetes

- 自动装箱,水平扩展,自我修复
- 服务发现和负载均衡
- 自动发布(默认滚动发布模式)和回滚
- 集中化配置管理和密钥管理
- 存储编排
- 任务批处理运行  

# Kubernetes 架构

![framework](/kubernetes/images/2233058-20210805105202343-1017707449.png)

## kubectl

用户客户端</br></br>
### 常用命令
1. kubectl create -f <file's name> 创建资源（需要yaml/yml/json文件）
2. kubectl run <name> --image=<image's name> 创建资源
3. kubectl get services 列出services
4. kubectl get pods 列出pods
5. kubectl get pods -A 列出所有pods
6. kubectl get pods -o wide 列出pods并显示详细信息
7. kubectl delete pod <pod's name> 删除目标pod
8. kubectl describe pod <pod's name> 显示pod详细信息
9. kubectl get nodes 列出nodes
10. kubectl get ns 列出namespace


## Master

它主要负责接受客户端的请求，安排容器的执行并且运行控制循环，将集群的状态向目标状态进行迁移

### 1. API Server

集群统一入口，以RESTful方式，交给etcd存储

提供了资源的唯一入口，并提供认证、授权、访问控制、API注册和发现等

*RESTful:  REST，全名 Representational State Transfer (表现层状态转移)，他是一种设计风格，一种软件架构风格，而不是标准，只是提供了一组设计原则和约束条件。RESTful 只是转为形容詞，就像那么 RESTful API 就是满足 REST 风格的，以此规范设计的 API。*

### 2. Scheduler

节点调度，选择node节点应用部署

负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上

### 3. Controller-manager

处理集群中常规后台任务，一个资源一个控制器

负责维护集群的状态，比如故障检测、自动扩展、滚动更新等

### 4. etcd

存储系统，用于保存集群相关的数据

保存整个集群的状态

## Node

### 1. Kubelet

Master排到Node节点代表，管理本机容器

负责维护容器的生命周期，同时也负责Volume(CVI)和网络（CNI）的管理

### 2. Kube-proxy

提供网络代理，负载均衡等操作

### 3. Pod

- Pod是K8S里能够被运行的最小的逻辑单元(原子单元) - 最小部署单元
- 1个Pod里面可以运行多个容器,它们共享UTS+NET +IPC名称空间
- 可以把Pod理解成豌豆荚,而同- -Pod内的每个容器是一 颗颗豌豆
- 一个Pod里运行多个容器,又叫:边车(SideCar)模式
- 一组容器的集合共享网络生命周期是短暂的

**生命周期图**

- 挂起（Pending）：Pod 已被 Kubernetes 系统接受，但有一个或者多个容器镜像尚未创建。等待时间包括调度 Pod 的时间和通过网络下载镜像的时间，这可能需要花点时间。
- 运行中（Running）：该 Pod 已经绑定到了一个节点上，Pod 中所有的容器都已被创建。至少有一个容器正在运行，或者正处于启动或重启状态。
- 成功（Succeeded）：Pod 中的所有容器都被成功终止，并且不会再重启。
- 失败（Failed）：Pod 中的所有容器都已终止了，并且至少有一个容器是因为失败终止。也就是说，容器以非0状态退出或者被系统终止。
- 未知（Unknown）：因为某些原因无法取得 Pod 的状态，通常是因为与 Pod 所在主机通信失败
    
    ![kubernetes-pod-life-cycle](/kubernetes/images/kubernetes-pod-life-cycle.jpg)

## NameSpace

- 随着项目增多、人员增加、集群规模的扩大,需要- -种能够隔离K8S内各种"资源”的方法，这就是名称空间
- 名称空间可以理解为K8S内部的虚拟集群组
- 不同名称空间内的"资源”名称可以相同,相同名称空间内的同种“资源”，”名称” 不能相同
- 合理的使用K8S的名称空间,使得集群管理员能够更好的对交付到K8S里的服务进行分类管理和浏览
- K8S里默认存在的名称空间有: default、 kube-system、 kube-public
- 查询K8S里特定“资源”要带上相应的名称空间
    

## Controller

### 1. Delopyment

Deployment 为 Pod 和 ReplicaSet 提供了一个声明式定义（declarative）方法，用来替代以前的 ReplicationController 来方便的管理应用。典型的应用场景包括：

- 定义 Deployment 来创建 Pod 和 ReplicaSet
- 滚动升级和回滚应用
- 扩容和缩容
- 暂停和继续 Deployment

### 2. Job

Job 负责批处理任务，即仅执行一次的任务，它保证批处理任务的一个或多个 Pod 成功结束。

### 3. DaemonSet

*DaemonSet* 确保全部（或者一些）Node 上运行一个 Pod 的副本。当有 Node 加入集群时，也会为他们新增一个 Pod 。当有 Node 从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod。

### 4. StatefulSet

StatefulSet是为了解决有状态服务的问题（对应Deployments和ReplicaSets是为无状态服务而设计）

## Storage

### 1. Secret

Secret 解决了密码、token、密钥等敏感数据的配置问题，而不需要把这些敏感数据暴露到镜像或者 Pod Spec 中。Secret 可以以 Volume 或者环境变量的方式使用。

Secret 有三种类型：

- **Service Account** ：用来访问 Kubernetes API，由 Kubernetes 自动创建，并且会自动挂载到 Pod 的 `/run/secrets/[kubernetes.io/serviceaccount](http://kubernetes.io/serviceaccount)` 目录中；
- **Opaque** ：base64 编码格式的 Secret，用来存储密码、密钥等；**[kubernetes.io/dockerconfigjson](http://kubernetes.io/dockerconfigjson)** ：用来存储私有 docker registry 的认证信息。

### 2. Volume

Kubernetes 中的卷有明确的寿命——与封装它的 Pod 相同。所以，卷的生命比 Pod 中的所有容器都长，当这个容器重启时数据仍然得以保存。当然，当 Pod 不再存在时，卷也将不复存在。也许更重要的是，Kubernetes 支持多种类型的卷，Pod 可以同时使用任意数量的卷。

# Kubernetes 对象

操作 Kubernetes 对象 —— 无论是创建、修改或者删除 —— 需要使用 [Kubernetes API](https://kubernetes.io/zh-cn/docs/concepts/overview/kubernetes-api)。 比如，当使用 `kubectl` 命令行接口（CLI）时，CLI 会调用必要的 Kubernetes API； 也可以在程序中使用[客户端库](https://kubernetes.io/zh-cn/docs/reference/using-api/client-libraries/)， 来直接调用 Kubernetes API。

在 Kubernetes 系统中，**Kubernetes 对象**是持久化的实体。 Kubernetes 使用这些实体去表示整个集群的状态。 具体而言，它们描述了如下信息：

- 哪些容器化应用正在运行（以及在哪些节点上运行）
- 可以被应用使用的资源
- 关于应用运行时行为的策略，比如重启策略、升级策略以及容错策略

## 资源对象

| 类别 | 名称 |
| --- | --- |
| 资源对象 | Pod、ReplicaSet、ReplicationController、Deployment、StatefulSet、DaemonSet、Job、CronJob、HorizontalPodAutoscaling、Node、Namespace、Service、Ingress、Label、CustomResourceDefinition |
| 存储对象 | Volume、PersistentVolume、Secret、ConfigMap |
| 策略对象 | SecurityContext、ResourceQuota、LimitRange |
| 身份对象 | ServiceAccount、Role、ClusterRole |

# yaml文件

## 属性说明

| 属性名称 | 介绍 |
| --- | --- |
| apiVersion | API版本 |
| kind | 资源类型 |
| metadata | 资源元数据 |
| spec | 资源规格 |
| replicas | 副本数量 |
| selector | 标签选择器 |
| template | Pod模板 |
| metadata | Pod元数据 |
| spec | Pod规格 |
| containers | 容器配置 |

## 数据结构

- List
- Map

## 常用apiVersion

- **v1**： Kubernetes API的稳定版本，包含很多核心对象：pod、service等。
- **apps/v1**： 包含一些通用的应用层的api组合，如：Deployments, RollingUpdates, and ReplicaSets。
- **batch/v1**： 包含与批处理和类似作业的任务相关的对象，如：job、cronjob。
- **autoscaling/v1**： 允许根据不同的资源使用指标自动调整容器。**[networking.k8s.io/v1](http://networking.k8s.io/v1)**： 用于Ingress。**[rbac.authorization.k8s.io/v1](http://rbac.authorization.k8s.io/v1)**：用于RBAC

## kind

kind指定这个资源对象的类型，如 pod、deployment、statefulset、job、cronjob

## metadata

metadata常用的配置项有 name,namespace,即配置其显示的名字与归属的命名空间。

## spec

一个嵌套字典与列表的配置项，也是主要的配置项，支持的子项非常多，根据资源对象的不同，子项会有不同的配置。

## yaml file examples

### 创建pod

```
apiVersion: v1  #api版本，一般为v1
kind: Pod  #资源对象类型
metadata:
  name: command-demo  #对象名称
  labels:
    purpose: demonstrate-command  #目的
spec:
  containers:  #container相关信息
  - name: command-demo-container  #container名称
    image: debian  #目标image名称
    command: ["printenv"]
    args: ["HOSTNAME", "KUBERNETES_PORT"]
  restartPolicy: OnFailure
```

### 创建secret

```
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
data:
  username: bXktYXBw
  password: Mzk1MjgkdmRnN0pi
```

### 创建deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # 告知 Deployment 运行 2 个与该模板匹配的 Pod
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

## 参考原文
1. https://kubernetes.io/zh-cn/docs/home/  
2. https://jimmysong.io/kubernetes-handbook/concepts/
3. https://www.cnblogs.com/caodan01/p/15102328.html
4. https://juejin.cn/post/7107251448034885639