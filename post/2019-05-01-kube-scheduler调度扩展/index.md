
Kubernetes 自带了一个默认调度器kube-scheduler，其内置了很多节点预选和优选的调度算法，一般调度场景下可以满足要求。但是在一些特殊场景下，默认调度器不能满足我们复杂的调度需求。我们就需要对调度器进行扩展，以达到调度适合业务场景的目的。



## 背景

中间件redis容器化后，需要两主不能在同一个节点上，一对主从不能在同一节点上；elasticsearch容器化后，两个data实例不能在同一节点上。在这类场景下，默认调度器内置的预选、优选算法不能满足需求，我们有以下三种选择：

- 将新的调度算法添加到默认调度程序中，并重新编译镜像，最终该镜像运行的实例作为kubernetes集群调度器；

- 参考kube-scheduler实现满足自己业务场景的调度程序，并编译镜像，将该程序作为独立的调度器运行到kubernetes集群内，需要用该调度器调度的pod实例，在spec.schedulerName里指定该调度器；

  ![](http://pqek9gnem.bkt.clouddn.com/20190426155726.png)

- 实现“调度扩展程序“：默认调度器kube-scheduler在进行预选时会调用该扩展程序进行过滤节点；在优选时会调用该扩展程序进行给节点打分，或者在bind操作时，调用该扩展器进行bind操作。



**对上述三种方式进行评估：**

第一种：将自己的调度算法添加到默认调度器kube-scheduler中，对原生代码侵入性较高，而且随着kubernetes版本升级，维护成本也较高；

第二种：默认调度器里内置了很多优秀调度算法，如：检查节点资源是否充足；端口是否占用；volume是否被其他pod挂载；亲和性；均衡节点资源利用等，如果完全使用自己开发的调度器程序，可能在达到了实际场景调度需求同时，失去更佳的调度方案，除非集成默认调度器中的算法到自己独立调度程序中，但这无疑是不现实的；

第三种：通过启动参数的policy配置，选用某些默认调度器中的预选、优选调度算法的同时，也可以调用外部扩展调度程序的算法，计算得到最优的调度节点，无需修改kube-scheduler代码，只需要在启动参数中增加配置文件即可将默认调度程序和扩展调度程序相互关联。

可以参考：

<https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/scheduler_extender.md> 



故采用第三种：实现扩展调度程序的方案。



## 整体架构



![](http://pqek9gnem.bkt.clouddn.com/20190426171921.png)

kube-scheduler在调度pod实例时，首先获取到Node1、Node2、Node3三个节点信息，进行默认的预选阶段，筛选满足要求的节点，其次再调用扩展程序中的预选算法，选出剩下的节点，假设预选阶段Node3上资源不足被过滤掉，预选结束后只剩Node1和Node2；Node1和Node2进入kube-scheduler默认的优选阶段进行节点打分，其次再调用扩展调度程序中的优选算法进行打分，kube-scheduler会将所有算法的打分结果进行加权求和，获得分数最高的节点作为pod最终bind节点，然后kube-scheduler调用apiserver进行bind操作。 



## 实现步骤

### 实现扩展调度程序代码

编写扩展调度器程序代码，根据实际业务调度场景编写预选逻辑、优选逻辑：

![](http://pqek9gnem.bkt.clouddn.com/20190427101526.png)

实现预选接口，入参为schedulerapi.ExtenderArgs，出参为schedulerapi.ExtenderFilterResult：

![](http://pqek9gnem.bkt.clouddn.com/20190427100915.png)

实现优选接口，入参为schedulerapi.ExtenderArgs，出参为schedulerapi.HostPriorityList：

![](http://pqek9gnem.bkt.clouddn.com/20190427101929.png)

暴露http接口：

![](http://pqek9gnem.bkt.clouddn.com/20190427101845.png)



**参考：**

<https://github.com/ll837448792/k8s-scheduler-extender-example> 



### 默认调度器部署

由于kubernetes集群内已经有了一个名为default-scheduler的默认调度器，为了不影响集群正常调度功能，下面会创建一个名为my-kube-scheduler的调度器，这个调度器和default-scheduler除了启动参数不一样外，镜像无差别。



1、创建一个名为my-scheduler-config的configmaps，data下的config.yaml文件指定了调度器的一些参数，包括leader选举，调度算法策略的选择（指定另一个configmaps），以及指定调度器的名称为my-kube-scheduler。

相应的创建一个my-scheduler-policy的configmaps，里面指定了选择哪些预选、优选策略，以及外部扩展调度程序的urlPrefix、扩展预选URI、扩展优选URI、扩展pod优先级抢占URI、扩展bind URI、扩展优选算法的权重等。

以保证my-kube-scheduler和扩展调度程序的通信。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-scheduler-config
  namespace: kube-system
data:
  config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1alpha1
    kind: KubeSchedulerConfiguration
    schedulerName: my-kube-scheduler
    algorithmSource:
      policy:
        configMap:
          namespace: kube-system
          name: my-scheduler-policy
    leaderElection:
      leaderElect: false
      lockObjectName: my-kube-scheduler
      lockObjectNamespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-scheduler-policy
  namespace: kube-system
data:
 policy.cfg : |
  {
    "kind" : "Policy",
    "apiVersion" : "v1",
    "predicates" : [
      {"name" : "PodFitsHostPorts"},
      {"name" : "PodFitsResources"},
      {"name" : "NoDiskConflict"},
      {"name" : "MatchNodeSelector"},
      {"name" : "HostName"}
    ],
    "priorities" : [
      {"name" : "LeastRequestedPriority", "weight" : 1},
      {"name" : "BalancedResourceAllocation", "weight" : 1},
      {"name" : "ServiceSpreadingPriority", "weight" : 1},
      {"name" : "EqualPriority", "weight" : 1}
    ],
    "extenders" : [{
      "urlPrefix": "http://10.168.107.12:80/scheduler",
      "filterVerb": "predicates/always_true",
      "prioritizeVerb": "priorities/zero_score",
      "preemptVerb": "preemption",
      "bindVerb": "",
      "weight": 1,
      "enableHttps": false,
      "nodeCacheCapable": false
    }],
    "hardPodAffinitySymmetricWeight" : 10
  }
```

2、在my-kube-scheduler yaml文件中将configmaps：my-scheduler-config以文件的形式挂载到容器内/my-scheduler目录下，并在启动参数中指定--config=/my-scheduler/config.yaml，使用和默认调度器一样的镜像。

![](http://pqek9gnem.bkt.clouddn.com/20190426180012.png)

增加挂载：

![](http://pqek9gnem.bkt.clouddn.com/20190426194520.png)

![](http://pqek9gnem.bkt.clouddn.com/20190426194610.png)

### 扩展调度器镜像制作和部署

1、编译扩展调度程序my-scheduler-extender镜像，以下为Dockerfile：

![](http://pqek9gnem.bkt.clouddn.com/20190426200003.png)

推送my-scheduler-extender镜像到harbor：

![](http://pqek9gnem.bkt.clouddn.com/20190426200105.png)



2、创建外部扩展程序my-scheduler-extender的deployment，如下为yaml描述：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-scheduler-extender
  namespace: kube-system
  labels:
    app: my-scheduler-extender
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-scheduler-extender
  template:
    metadata:
      labels:
        app: my-scheduler-extender
    spec:
      containers:
      - name: my-scheduler-extender
        image: 192.168.26.46/k8s-deploy/my-scheduler-extender:v1.0
        imagePullPolicy: Always
        livenessProbe:
          httpGet:
            path: /version
            port: 80
        readinessProbe:
          httpGet:
            path: /version
            port: 80
        ports:
          - containerPort: 80
```



### 验证

查看my-kube-scheduler pod日志，加载到了policy里的extender信息，获取到了扩展调度器的接口地址：

![](http://pqek9gnem.bkt.clouddn.com/20190426200608.png)



创建一个nginx的pod，指定schedulerName为my-kube-scheduler：

![](http://pqek9gnem.bkt.clouddn.com/20190426195151.png)



查看扩展调度器pod日志，发现默认调度器会调用extender扩展调度器，如下为extender日志打印的入参、出参：

![](http://pqek9gnem.bkt.clouddn.com/20190426200413.png)



从而可以通过编写扩展调度程序，对默认调度器的预选和优选算法进行扩展。
---

