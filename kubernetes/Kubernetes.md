# 目录
- [目录](#目录)
- [Kubernetes Doc](#kubernetes-doc)
- [Kubernetes前身](#kubernetes前身)
  - [Borg](#borg)
    - [Borg vs Kubernetes](#borg-vs-kubernetes)
- [Why Kubernetes](#why-kubernetes)
- [Kubernetes 架构](#kubernetes-架构)
  - [kubectl](#kubectl)
    - [常用命令](#常用命令)
  - [Master](#master)
    - [1. API Server](#1-api-server)
    - [2. Scheduler](#2-scheduler)
    - [3. Controller-manager](#3-controller-manager)
    - [4. etcd](#4-etcd)
  - [Node](#node)
    - [1. Kubelet](#1-kubelet)
    - [2. Kube-proxy](#2-kube-proxy)
    - [3. Pod](#3-pod)
  - [NameSpace](#namespace)
  - [Controller](#controller)
    - [1. Deployment](#1-deployment)
    - [2. Job](#2-job)
    - [3. CronJob](#3-cronjob)
    - [4. DaemonSet](#4-daemonset)
    - [5. StatefulSet](#5-statefulset)
  - [Service](#service)
    - [1. Service](#1-service)
    - [2. Ingress](#2-ingress)
  - [Storage](#storage)
    - [1. Secret （投射卷）](#1-secret-投射卷)
    - [2. Volume](#2-volume)
    - [3. ConfigMap （投射卷）](#3-configmap-投射卷)
  - [HPA (HorizontalPodAutoscaler)](#hpa-horizontalpodautoscaler)
- [Kubernetes 对象](#kubernetes-对象)
  - [资源对象](#资源对象)
- [yaml文件](#yaml文件)
  - [属性说明](#属性说明)
  - [数据结构](#数据结构)
  - [常用apiVersion](#常用apiversion)
  - [kind](#kind)
  - [metadata](#metadata)
  - [spec](#spec)
- [参考原文](#参考原文)


# Kubernetes Doc

链接：https://kubernetes.io/zh-cn/docs/home/

# Kubernetes前身

## Borg

### Borg vs Kubernetes
1. Borg 没有 pod 概念 (Borg 的 outermost containers 叫 alloc)
2. Borg 没有像 Kubernetes 一样大的社区
3. Kubernetes 更加具有灵活性和延展性

# Why Kubernetes

- 自动装箱：根据配置文件自动部署应用容器
- 水平扩展：通过简单命令对应用容器进行规模扩大或剪裁（HorizontalPodAutoscaler）
- 自我修复：容器失败/Node出问题时自动对容器进行重新部署和调度
- 服务发现：不需要知道每个IP和端口，只需要通过服务的名字就可以使用服务（Service）
- 负载均衡：可以合理分配负载
- 自动发布(默认滚动发布模式)和回滚：不停机更新 & 可以回滚到历史版本
- 集中化配置管理和密钥管理：（Secret）
- 存储编排：可以存储在本地目录、网络存储（NFS、Cluster、Ceph等）、公共云存储服务
- 任务批处理运行：提供一次性、定时任务（Job、CronJob）
- 开源

# Kubernetes 架构

![framework](/kubernetes/images/2233058-20210805105202343-1017707449.png)

## kubectl

用户客户端</br></br>
### 常用命令
1. kubectl create -f <file's name> 创建新资源（重复配置会报错）（需要yaml/yml/json文件）
2. kubectl apply -f <file's name> 配置应用于资源（重复配置不会报错）（需要yaml/yml/json文件）
3. kubectl run <name> --image=<image's name> 创建容器镜像
4. kubectl get pods / nodes / ingresses (ing） / deployments / secrets / namespaces (ns) / services (svc) ... 列出pods/nodes/ingresses/deployments/secrets/namespaces/services...
6. kubectl get pods -A 列出所有pods
7. kubectl get pods -o wide 列出pods并显示详细信息
8. kubectl get pods --ns=<namespace> 列出指定namespace的pod
9. kubectl get pods --watch 实时查看pods的信息
10. kubectl delete pod <pod's name> 删除目标pod
11. kubectl describe pod <pod's name> 显示pod详细信息


## Master

它主要负责接受客户端的请求，安排容器的执行并且运行控制循环，将集群的状态向目标状态进行迁移

### 1. API Server

1. API Server 为 REST 操作提供服务

2. **所有组件都通过该前端进行交互**

3. 提供了**资源的唯一入口**，并提供认证、授权、访问控制、API注册和发现等
   
 *RESTful:  REST，全名 Representational State Transfer (表现层状态转移)，他是一种设计风格，一种软件架构风格，而不是标准，只是提供了一组设计原则和约束条件。RESTful 只是转为形容詞，就像那么 RESTful API 就是满足 REST 风格的，以此规范设计的 API。*

### 2. Scheduler

负责资源的调度，按照预定的调度策略将Pod调度到相应的节点上（选择具有合适资源的node去运行对应pod）

### 3. Controller-manager

1. 是一个守护进程，是一个永不停止的循环

2. 处理集群中常规后台任务，一个资源一个控制器

3. 负责维护集群的状态，比如故障检测、自动扩展、滚动更新等

4. 维护副本数目，满足期望值

**内部结构：**
- Replication Controller
- Node Controller
- ResourceQuota Controller
- NameSpace Controller
- ServiceAccount Controller
- Token Controller
- Service Controller
- EndPoint Controller

### 4. etcd

1. 存储系统，用于保存集群相关的数据

2. 保存整个集群的状态

3. 键值对存储数据库

- **v2**: Memory
- **v3**: Database
- **HTTP Server**: 采用HTTP协议
- **Raft**: 读写信息存储
- **WAL**： 生成日志（有备份）
- **Store**： 持久化存储

## Node

### 1. Kubelet

1. kubelet 是在每个节点上运行的主要 “节点代理”。它可以使用以下方式之一向 API 服务器注册：

- 主机名（hostname
- 覆盖主机名的参数
- 特定于某云驱动的逻辑

2. 负责维护容器的生命周期，同时也负责Volume(CVI)和网络（CNI）的管理

*CNI（容器网络接口）是一个云原生计算基金会项目，它包含了一些规范和库，用于编写在 Linux 容器中配置网络接口的一系列插件。CNI 只关注容器的网络连接，并在容器被删除时移除所分配的资源。*

3. **接受Master发出的指令并操作管理对应容器/创建pod**

### 2. Kube-proxy

1. 提供网络代理，负载均衡等操作

2. 写入规则到 IPVS、IPTABLES 实现服务映射访问

*IPTABLES 是Linux中的代理，IPTABLES + NETFILTER 是Linux的防火墙。*

### 3. Pod

- Pod是K8S里能够被运行的**最小的逻辑单元(原子单元)** - **最小部署单元**
- 1个Pod里面可以**运行多个容器**,它们共享UTS+NET +IPC名称空间
- 可以把Pod理解成豌豆荚,而同- -Pod内的每个容器是一 颗颗豌豆
- 一个Pod里运行多个容器,又叫:边车(SideCar)模式
- 一组容器的集合共享网络生命周期是短暂的

Pod内有自己的 IP address、 Volume、 Containerized Apps


**为什么需要Pod：**

因为Pod可以搭载多个containers，在运维时可以成组地操作多个containers。


**重启策略：**

在配置文件的RestartPolicy中可以修改
- Always：当容器失效时，由kubelet自动重启该容器。
- OnFailure：当容器终止运行且退出码不为0时，由kubelet自动重启该容器。
- Never：不论容器运行状态如何，kubelet都不会重启该容器。


**创建pod：**

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

![pic](/kubernetes/images/k8s.PNG)

### 1. Deployment

用于**无状态服务**

*无状态服务不会在本地存储持久化数据.多个服务实例对于同一个用户请求的响应结果是完全一致的.这种多服务实例之间是没有依赖关系,比如web应用,在k8s控制器 中动态启停无状态服务的pod并不会对其它的pod产生影响.*

Deployment 为 Pod 和 ReplicaSet 提供了一个声明式定义（declarative）方法，用来替代以前的 ReplicationController 来方便的管理应用。典型的应用场景包括：

- 定义 Deployment 来创建 Pod 和 ReplicaSet
- 滚动升级和回滚应用
- 扩容和缩容
- 暂停和继续 Deployment

  *RC(Replication Controller)是一种用于保证应用程序副本数量的对象。它通过监视当前的副本数量，并在数量低于预期时创建新的副本，反之删除多余的副本。*

  *RS(Replica Set)是 RC 的替代者，提供了更灵活的选择器功能。它使用标签选择器来确定要保留的副本数量，并在必要时创建或删除副本。*

**创建deployment**

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

### 2. Job

Job 负责批处理任务，即**仅执行一次的任务**，它保证批处理任务的一个或多个 Pod 成功结束。

**创建Job**
```
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

### 3. CronJob

CronJob 创建基于**时隔重复调度**的 Job。

**创建CronJob**
```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.28
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure

```



**Cron 时间表语法**

.spec.schedule 字段是必需的。该字段的值遵循 Cron 语法：

```
 ┌───────────── 分钟 (0 - 59)
 │ ┌───────────── 小时 (0 - 23)
 │ │ ┌───────────── 月的某天 (1 - 31)
 │ │ │ ┌───────────── 月份 (1 - 12)
 │ │ │ │ ┌───────────── 周的某天 (0 - 6)（周日到周一；在某些系统上，7 也是星期日）
 │ │ │ │ │                          或者是 sun，mon，tue，web，thu，fri，sat
 │ │ │ │ │
 │ │ │ │ │
 * * * * *
 ```
例如 0 0 13 * 5 表示此任务必须在每个星期五的午夜以及每个月的 13 日的午夜开始。

### 4. DaemonSet

1. *DaemonSet* 可以保证集群中所有的或者部分的节点都能够运行同一份 Pod 副本，每当有新的节点被加入到集群时，Pod 就会在目标的节点上启动，如果节点被从集群中剔除，节点上的 Pod 也会被垃圾收集器清除

2. 提供基础服务和守护进程

**适用于：当需要每一个node都运行一个进程去做某些任务的时候， 例如集群存储、日志收集和监控等**

**创建DaemonSet**
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # 这些容忍度设置是为了让该守护进程集在控制平面节点上运行
      # 如果你不希望自己的控制平面节点运行 Pod，可以删除它们
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log

```

### 5. StatefulSet

StatefulSet是为了解决**有状态服务**的问题（对应Deployments和ReplicaSets是为无状态服务而设计）

**适用于：数据库等需要持久存储的模块**

*有状态服务需要在本地存储持久化数据,典型的是分布式数据库的应用,分布式节点实例之间有依赖的拓扑关系.比如,主从关系. 如果K8S停止分布式集群中任 一实例pod,就可能会导致数据丢失或者集群的crash.*

**创建StatefulSet**

1. 创建StorageClass

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```

2. 创建PersistentVolume （根据需求创建相应数量，需要大于等于StatefulSet的Replicas数量）
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-0
  labels:
    type: local
spec:
  storageClassName: standard
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-1
  labels:
    type: local
spec:
  storageClassName: standard
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

3. 创建StatefulSet (volumeClaimTemplates用于自动创建pvc）
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web11111
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        storageClassName: standard
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 1Gi
```

## Service

### 1. Service

**ClusterIP:** 通过集群内部IP暴露服务，只能在集群内部使用。

**NodePort：** 可以从集群外部访问。使用公共IP。

**LoadBalancer：** 可以向外部暴露服务。需要Cloud Provider（云供应商）。

**ExternalName：** 把集群外部的服务引入到集群内部。

### 2. Ingress

1. Ingress 是对集群中服务的**外部访问**进行管理的 API 对象，典型的访问方式是 HTTP。

2. Ingress 可以提供负载均衡、SSL 终结和基于名称的虚拟托管。

**注意：集群中必须有一个正在运行的 Ingress Controller 才可以使用 Ingress**

Nginx Ingress Controller 安装文档：https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/

## Storage

### 1. Secret （投射卷）

Secret 解决了密码、token、密钥等敏感数据的配置问题，而不需要把这些敏感数据暴露到镜像或者 Pod Spec 中。Secret 可以以 Volume 或者环境变量的方式使用。

Secret 有三种类型：

- **Service Account** ：用来访问 Kubernetes API，由 Kubernetes 自动创建，并且会自动挂载到 Pod 的 `/run/secrets/[kubernetes.io/serviceaccount](http://kubernetes.io/serviceaccount)` 目录中；
- **Opaque** ：base64 编码格式的 Secret，用来存储密码、密钥等
- **[kubernetes.io/dockerconfigjson](http://kubernetes.io/dockerconfigjson)** ：用来存储私有 docker registry 的认证信息。

**创建secret**

```
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
data:
  username: bXktYXBw
  password: Mzk1MjgkdmRnN0pi
```

### 2. Volume

Kubernetes 中的卷有明确的寿命——与封装它的 Pod 相同。所以，卷的生命比 Pod 中的所有容器都长，当这个容器重启时数据仍然得以保存。当然，**当 Pod 不再存在时，卷也将不复存在**。也许更重要的是，Kubernetes 支持多种类型的卷，Pod 可以同时使用任意数量的卷。

**【卷的种类】**

1. 持久卷：独立于Pod的生命周期
    1. PV： 是集群中的一块存储，可以由管理员事先制备， 或者使用存储类（Storage Class）来动态制备。 持久卷是集群资源，就像节点也是集群资源一样
       ```
       apiVersion: v1
       kind: PersistentVolume
       metadata:
         name: pv-0
         labels:
           type: local
       spec:
         storageClassName: standard
         capacity:
           storage: 10Gi
         accessModes:
           - ReadWriteOnce
         hostPath:
           path: "/mnt/data"
       ```
    3. PVC：表达的是用户对存储的请求。概念上与 Pod 类似。 Pod 会耗用节点资源，而 PVC 申领会耗用 PV 资源。Pod 可以请求特定数量的资源（CPU 和内存）
       ```
       apiVersion: v1
       kind: PersistentVolumeClaim
       metadata:
         name: task-pv-claim
       spec:
         storageClassName: manual
         accessModes:
           - ReadWriteOnce
         resources:
           requests:
             storage: 3Gi
       ```
3. 投射卷：可以将若干现有的卷源映射到同一个目录之上
4. 临时卷：会遵从 Pod 的生命周期，与 Pod 一起创建和删除

### 3. ConfigMap （投射卷）

1. ConfigMap 是一种 API 对象，用来将非机密性的数据保存到键值对中（存储机密数据请使用 Secret ）。使用时， Pods 可以将其用作环境变量、命令行参数或者存储卷中的配置文件。

2. ConfigMap 将你的环境配置信息和 容器镜像 解耦，便于应用配置的修改。

```
kind: ConfigMap
apiVersion: v1
metadata:
  creationTimestamp: 2023-6-19T12:06:00Z
  name: example-config
  namespace: default
data:
  example.property.file: |
    property.1=value-1
    property.2=value-2
    property.3=value-3
```

## HPA (HorizontalPodAutoscaler)

根据CPU利用率，平行扩展和裁剪Pod数量

ex1. 维持扩缩目标中的 Pods 的平均资源利用率在 60% (*利用率是 Pod 的当前资源用量与其请求值之间的比值*)

```
type: Resource
resource:
  name: cpu
  target:
    type: Utilization
    averageUtilization: 60
```

ex2. 确保所有 Pod 中 application 容器的平均 CPU 用量为 60%

```
type: ContainerResource
containerResource:
  name: cpu
  container: application
  target:
    type: Utilization
    averageUtilization: 60
```
   



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


# 参考原文
1. https://kubernetes.io/zh-cn/docs/home/  
2. https://jimmysong.io/kubernetes-handbook/concepts/
3. https://www.cnblogs.com/caodan01/p/15102328.html
4. https://juejin.cn/post/7107251448034885639、
