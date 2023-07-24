## 定义

Batch job 是提交给系统的一组预定义处理动作。这中间用户和系统的交互很少或者没有。Job 不需要用户交互就可以自己运作。实现自动化。

## 优点

- 允许多用户共享计算机资源
- 可以把作业处理转移到计算机资源不太繁忙的时段
- 避免计算资源闲置，而且无须时刻有人工监视和干预
- 在昂贵的高端计算机上，使昂贵的资源保持高使用率，以减低平均开销

## Kubernetes Batch jobs

### YAML

```yaml
apiVersion: batch/v1  # api版本，group/version                                                                                                                                                                                                                     
kind: Job  # api resources                                                                                                                                                                                                                                   
metadata: # 描述和标识资源的信息                                                                                                                                                                                                                                      
	annotations:  # 键值对形式，存储与资源相关的其他信息                                                                                                                                                                                                                                   
		kubectl.kubernetes.io/last-applied-configuration: |                                                                                                                                                                                            
			{"apiVersion":"batch/v1","kind":"Job","metadata":{"annotations":{},"creationTimestamp":null,"name":"completionmode-job","namespace":"default"},"spec":{"completionMode":"Indexed","completions":4,"template":{"metadata":{"creationTimestamp":null},"spec":{"containers":[{"command":["sh","-c","echo completeMode"],"image":"nginx","name":"completionmode-job","resources":{}}],"restartPolicy":"Never"}}},"status":{}}                                                             
	creationTimestamp: "2023-07-04T09:46:23Z"  # 创建资源时间戳                                                                                                                                                                                                    
	generation: 1  # 随着资源变化而递增，用于追踪资源更新和变化                                                                                                                                                                                                                               
	labels:  # 用于资源分类和选择                                                                                                                                                                                                                                        
		controller-uid: ce980c9e-a3aa-4a79-8d15-74934b99211c                                                                                                                                                                                         
		job-name: completionmode-job                                                                                                                                                                                                               
	name: completionmode-job  # job的名字                                                                                                                                                                                                                     
	namespace: default  # job所在的namespace                                                                                                                                                                                                                           
	resourceVersion: "1987903"  # 标识资源版本，由apiServer维护                                                                                                                                                                                                                   
	uid: ce980c9e-a3aa-4a79-8d15-74934b99211c  # 资源唯一标识符，即使重命名uid也不变                                                                                                                                                                                                  
spec:  # 描述对象期望状态                                                                                                                                                                                                                                          
	backoffLimit: 6  # 回退限制                                                                                                                                                                                                                              
	completionMode: Indexed  # 完成的模式 Indexed/NonIndexed                                                                                                                                                                                                                      
	completions: 4  # pod 最终需要完成数量                                                                                                                                                                                                                             
	parallelism: 1  # pod 可并行数量                                                                                                                                                                                                                            
	selector:  # 选择器                                                                                                                                                                                                                                     
		matchLabels:  # 匹配的label                                                                                                                                                                                                                                  
			controller-uid: ce980c9e-a3aa-4a79-8d15-74934b99211c                                                                                                                                                                                     
	suspend: false  # 挂起状态                                                                                                                                                                                                                             
	template:  # 创建模板                                                                                                                                                                                                                                    
		metadata:  # 模板信息描述                                                                                                                                                                                                                                    
			creationTimestamp: null  # 创建时间戳                                                                                                                                                                                                                    
			labels:  # 标签                                                                                                                                                                                                                                     
				controller-uid: ce980c9e-a3aa-4a79-8d15-74934b99211c                                                                                                                                                                                         
				job-name: completionmode-job                                                                                                                                                                                                             
		spec:  # 期望状态                                                                                                                                                                                                                                        
			containers:  # 容器描述                                                                                                                                                                                                                                 
			- command: # 容器内输入指令                                                                                                                                                                                                                                   
				- sh                                                                                                                                                                                                                                         
				- -c                                                                                                                                                                                                                                         
				- echo completeMode                                                                                                                                                                                                                          
				image: nginx  # 需要拉取的镜像                                                                                                                                                                                                                               
				imagePullPolicy: Always  # 镜像拉取规则 1.IfNotPresent 2.Always 3.Never                                                                                                                                                                                                                   
				name: completionmode-job  # 容器名称                                                                                                                                                                                                                  
				resources: {}  # 指定资源对象，限制资源使用。request代表最小需求，limit代表最大限制                                                                                                                                                                                                                             
				terminationMessagePath: /dev/termination-log   # 指定在容器终止时记录终止消息的文件路径                                                                                                                                                                                              
				terminationMessagePolicy: File  # 容器终止策略，1.File 2.FallbackToLogsOnError 3.FallbackToLogsOnFailure                                                                                                                                                                                                          
			dnsPolicy: ClusterFirst  # 指定容器如何解析域名 1.ClusterFirst 2.Default 3.None 4.ClusterFirstWithHostNet                                                                                                                                                                                                                     
			restartPolicy: Never  # 重启策略 1.Never 2.OnFailure 3.Always                                                                                                                                                                                                                        
			schedulerName: default-scheduler  # 指定调度器的属性                                                                                                                                                                                                         
			securityContext: {}  # 指定了容器或pod安全上下文的属性，可以增加或限制容器或Pod的权限                                                                                                                                                                                                                      
			terminationGracePeriodSeconds: 30  # 指定pod的grace period的时间                                                                                                                                                                                                   
status:  # 资源的运行状态、条件和事件等信息                                                                                                                                                                                                                                      
	completedIndexes: 0-3  # indexed pod                                                                                                                                                                                                                      
	completionTime: "2023-07-04T09:47:30Z"  # 完成时间戳                                                                                                                                                                                                      
	conditions:  # Job当前状态                                                                                                                                                                                                                                
	- lastProbeTime: "2023-07-04T09:47:30Z"  # 最后一次探测的时间戳                                                                                                                                                                                                      
		lastTransitionTime: "2023-07-04T09:47:30Z"  # 最后一次转换的时间戳，例如状态的转换（Pod的Ready条件从True变为False））                                                                                                                                                                                                  
		status: "True"  # condition的当前状态                                                                                                                                                                                                                             
		type: Complete  # condition的类型                                                                                                                                                                                                                           
	startTime: "2023-07-04T09:46:23Z" # 资源启动时间                                                                                                                                                                                                          
	succeeded: 4  # 作业完成的数量
```

### dnsPolicy

1. "ClusterFirst"（默认值）：在该策略下，Pod中的容器首先尝试使用集群的DNS解析器进行域名解析。如果无法解析，则会尝试使用其他可用的名称解析插件。
2. "Default"：该策略继承自所在节点的DNS配置。Pod中的容器将使用节点的DNS解析配置进行域名解析。
3. "None"：在该策略下，Pod中的容器将不会进行DNS解析。容器将无法使用DNS服务解析域名，必须使用直接的IP地址来访问其他服务。
4. "ClusterFirstWithHostNet"：这个策略仅适用于使用HostNetwork的Pod。在这种情况下，容器将使用集群的DNS解析器进行域名解析。

### terminationMessagePolicy

1. File"：指定终止消息将被写入容器文件系统中的指定路径。这是默认策略，通过设置**`terminationMessagePolicy: File`**来启用。
2. "FallbackToLogsOnError"：指定在无法将终止消息写入指定路径时，Kubernetes将尝试从容器的日志中提取终止消息。如果无法从日志中提取终止消息，则提供一个默认的终止消息。通过设置**`terminationMessagePolicy: FallbackToLogsOnError`**来启用此策略。
3. "FallbackToLogsOnFailure"：指定无论能否写入指定路径，Kubernetes都将尝试从容器的日志中提取终止消息。如果无法从日志中提取终止消息，则不提供默认的终止消息。通过设置**`terminationMessagePolicy: FallbackToLogsOnFailure`**来启用此策略。

### Simple Job

只运行一次，这个 Job 会使用 pod 来工作。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job-example
spec:
  template:
    terminationGracePeriodSeconds: 0
    spec:      
      containers:
      - name: my-job
        image: alpine
        imagePullPolicy: IfNotPresent
        restartPolicy: Never
        command: ['sh', '-c', 'echo First Batch Job; sleep 10']
```

### Sequential Jobs

用于按顺序运行一批 jobs 而不是运行一个很长而且失败率很高的 job。

- `spec.completions` 字段设置非0正数

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sequential-job
spec:
  completions: 4
  template:
    restartPolicy: Never
    terminationGracePeriodSeconds: 0
    spec:      
      containers:
      - name: my-job
        image: alpine
        imagePullPolicy: IfNotPresent
        command: ['sh', '-c', 'echo Sequential Job ; sleep 10']
```

### Parallel Jobs

用于同时运行一批 Jobs。jobs 必须是空转的。*（空转：指没有任何负载时）*

- `spec.parallelism` 如果未设置，则默认为 1。 如果设置为 0，则 Job 相当于启动之后便被暂停，直到此值被增加。
- `spec.completions` 和 `spec.parallelism` 这两个属性都不设置时，均取默认值 1。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-job
spec:
  template:
    restartPolicy: Never
    terminationGracePeriodSeconds: 0
    spec:      
      containers:
      - name: my-job
        image: alpine
        imagePullPolicy: IfNotPresent
        command: ['sh', '-c', 'echo Job Pod is Running ; sleep 10']
   completions: 4
   parallelism: 2
```

### 完成模式

`.spec.completionMode` 中设置完成模式

- **NonIndexed**（默认）：Completed pod 数量达到 `.spec.completions` 所 设值时认为 Job 已经完成。
- **Indexed**：Job 的 Pod 会获得对应的完成索引，取值为 0 到 `.spec.completions` -1。索引获取方式：
    1. Pod 注解 `batch.kubernetes.io/job-completion-index`。
    2. 对于容器化的任务，在环境变量 `JOB_COMPLETION_INDEX` 中。
    3. 作为 Pod 主机名的一部分，遵循模式 `$(job-name)-$(index)`。

### Job 终止与清理

- Job 完成后不会创建新的 pod，已完成的 pod 也不会清除（用于查看 pod 日志）
- 如果 pod 失败，job 会重试直到达到上限`.spec.backoffLimit`
- `.spec.activeDeadlineSeconds` 也用于终止 Job，该字段用于设置一个秒数，这个字段代表 job 的生命周期。`.spec.activeDeadlineSeconds` 优先级大于 `.spec.backoffLimit`
- **TTL 机制**
    - 自动清理已完成的 Job
    - 通过`.spec.ttlSecondsAfterFinished` 字段。例如 *ttlSecondsAfterFinished: 100* 代表该 job 在结束 100 秒之后，可以成为被自动删除的对象
    - 会同时删除所有依赖的对象，包括 pod 和 job 本身
    

### Pod 失效策略

- Pod 失效策略使用 `.spec.podFailurePolicy` 字段来定义， 它能让你的集群根据容器的退出码和 Pod 状况来处理 Pod 失效事件。
    
    ```
    apiVersion: batch/v1
    kind: Job
    metadata:
    name: job-pod-failure-policy-example
    spec:
    completions: 12
    parallelism: 3
    template:
    spec:
    restartPolicy: Never
    containers:
          -name: main
    image: docker.io/library/bash:5
    command: ["bash"]# 模拟一个触发 FailJob 动作的错误的示例命令args:
            - -c
            - echo "Hello world!" && sleep 5 && exit 42
    backoffLimit: 6
    podFailurePolicy:
    rules:
        -action: FailJob
    onExitCodes:
        operator: NotIn
        values: [0]
    containerName: main# 可选operator: In# In 和 NotIn 二选一values: [42]
        -action: Ignore# Ignore、FailJob、Count 其中之一onPodConditions:
          -type: DisruptionTarget# 表示 Pod 失效
    ```

    -  `spec.podFailurePolicy.rules[*].containerName ` 将规则限制到特定容器
    -  `spec.podFailurePolicy.rules[*].action ` 当 Pod 失效策略发生匹配时要采取的操作
        - FailJob：表示 Pod 的任务应标记为 Failed，并且所有正在运行的 Pod 应被终止。
        - Ignore：表示 .spec.backoffLimit 的计数器不应该增加，应该创建一个替换的 Pod。
        - Count：表示 Pod 应该以默认方式处理。.spec.backoffLimit 的计数器应该增加。
    
- 也可以使用  `.spec.backoffLimit` 限制回退次数。

### 挂起 Job

- `.spec.suspend: true` 即可挂起 Job
- 所有正在运行且状态不是`Completed` 的 Pod 将被终止
- Job 的 `status` 可以用来确定 Job 是否被挂起，或者曾经被挂起。通过该 Job 的 yaml 文件或者 describe 可查看。

## 参考文档

1. [https://zh.wikipedia.org/zh-cn/批处理任务](https://zh.wikipedia.org/zh-cn/%E6%89%B9%E5%A4%84%E7%90%86%E4%BB%BB%E5%8A%A1)
2. https://medium.com/analytics-vidhya/batch-workloads-in-kubernetes-1dd06bd36056