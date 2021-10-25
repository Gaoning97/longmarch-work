---
layout: post
title: Kubernetes微服务学习之Kubernetes基本概念和应用(上)
tags: K8s
categories: K8s
---
# k8s架构简介
    k8s平台主要是解决集群资源调度的问题，当有请求过来时，进行分发请求
    具有监控节点、自愈、管理集群网络、保证pod互联的能力
![image](http://oss.longmarch.work/k8s-2-1.png)

## k8s Master节点与Worker节点

- Master节点是Kubernetes 的主节点。集群所有的控制命令都传递给Master组件，在Master节点上运行。kubectl命令在其他Node节点上无法执行。

- Master节点上面主要由四个模块组成：etcd、api server、controller manager、scheduler。
```doc
api server：负责对外提供restful的Kubernetes API服务，其他Master组件都通过调用api server提供的rest接口实现各自的功能，如controller就是通过api server来实时监控各个资源的状态的。

etcd：是 Kubernetes 提供的一个高可用的键值数据库，用于保存集群所有的网络配置和资源对象的状态信息，也就是保存了整个集群的状态。数据变更都是通过api server进行的。整个kubernetes系统中一共有两个服务需要用到etcd用来协同和存储配置，分别是：
1）网络插件flannel，其它网络插件也需要用到etcd存储网络的配置信息；
2）kubernetes本身，包括各种资源对象的状态和元信息配置

scheduler：监听新建pod副本信息，并通过调度算法为该pod选择一个最合适的Node节点。会检索到所有符合该pod要求的Node节点，执行pod调度逻辑。调度成功之后，会将pod信息绑定到目标节点上，同时将信息写入到etcd中。一旦绑定，就由Node上的kubelet接手pod的接下来的生命周期管理。如果把scheduler看成一个黑匣子，那么它的输入是pod和由多个Node组成的列表，输出是pod和一个Node的绑定，即将这个pod部署到这个Node上。Kubernetes目前提供了调度算法，但是同样也保留了接口，用户可以根据自己的需求定义自己的调度算法。

controller manager：负责维护集群的状态，比如故障检测、自动扩展、滚动更新等。每个资源一般都对应有一个控制器，这些controller通过api server实时监控各个资源的状态，controller manager就是负责管理这些控制器的。当有资源因为故障导致状态变化，controller就会尝试将系统由“现有状态”恢复到“期待状态”，保证其下每一个controller所对应的资源始终处于期望状态。比如我们通过api server创建一个pod，当这个pod创建成功后，api server的任务就算完成了。而后面保证pod的状态始终和我们预期的一样的重任就由controller manager去保证了。
controller manager 包括运行管理控制器（kube-controller-manager）和云管理控制器（cloud-controller-manager）

```

![image](http://oss.longmarch.work/k8s-2-3.png)

- worker节点 

![image](http://oss.longmarch.work/k8s-2-2.png)

## k8s执行流程案例

![image](http://oss.longmarch.work/k8s-2-4.png)

## K8s总体架构

![image](http://oss.longmarch.work/k8s-2-5.png)

## K8s各组件作用

![image](http://oss.longmarch.work/k8s-2-6.png)

# k8s虚拟机抽象Pod

## k8s和Pod
![image](http://oss.longmarch.work/k8s-2-7.png)

- 容器是应用加上操作系统的包装 是一种资源隔离抽象
- pod是容器的包装 是一种虚拟机抽象
- k8s是管理pod虚拟机的数据中心抽象 
-  所以应用、容器、 pod、k8s是层层包装关系  **最里面是应用，容器包装应用   pod包装容器，k8s是总体资源的管理者**
## PetClinic项目模拟Pod应用
![image](http://oss.longmarch.work/k8s-2-8.png)

### Pod发布规范
![image](http://oss.longmarch.work/k8s-2-9.png)

### Pod相关指令
```sheel
kubectl apply -f petclinic-pod.yml (部署petclinic项目pod)

kubectl get all (查看正在运行中的所有k8s服务)

kubectl get po petclinic (查看Petclinic pod的运行状态)

kubectl describe pod petclinic (查看petclinic pod 运行详情)

kubectl delete po petclinic (删除指定petclinic pod)
```

## 如何访问Pod
![image](http://oss.longmarch.work/k8s-2-10.png)

![image](http://oss.longmarch.work/k8s-2-11.png)

![image](http://oss.longmarch.work/k8s-2-12.png)

## pod小结
- Pod是K8s云平台的虚拟机资源
- Pod的发布规范
    - kind -> Pod
    - spec -> 容器镜像
- 通过Port-Forward可本机访问Pod

# K8s反向代理Service

## 传统反向代理与K8s Service反向代理对比

### 传统反向代理
![image](http://oss.longmarch.work/k8s-3-1.png)

### K8s NodePort Service反向代理

![image](http://oss.longmarch.work/k8s-3-2.png)
- k8s中 service将流量反向路由到后端pod 是通过selector和lable机制来实现 
- label是打标签机制，可以给pod打上标签 标签就是键值对 比如app： petclinic
- selector是路由选择机制  如果selector 上的标签和后台pod的标签能够匹配上 那么运行时这个service就会将流量路由到匹配的pod上 
- 对比nginx传统静态
service selector labels 能力要强很多 路由转发 负载均衡 提供稳定访问点  

### Label和Selector

![image](http://oss.longmarch.work/k8s-3-3.png)

## Service 发布
### Service 发布样例规范
![image](http://oss.longmarch.work/k8s-3-4.png)
### PetClinic Service发布规范
![image](http://oss.longmarch.work/k8s-3-5.png)
### Petclinic-pod.yml
![image](http://oss.longmarch.work/k8s-3-6.png)

## Service小结
- Service 是K8s提供的反向代理机制
- NodePort是Service的一种类型,可将  Service暴露给外网
- Label是K8S中的一种打标签的机制
- Selector是K8S中的路由选择定位机制


# 通过Service实现蓝绿发布

## 主要学习内容
- 蓝绿发布基本原理
- 基于Service/Selector/Label实现PetClinic的蓝绿发布

## Selector/Label和蓝绿发布
    蓝绿发布(Blue-Green-Delpoyment) ： 是一种应用升级发布方式 这种方式可以瞬间切换到新版本，也可以瞬间回退到老版本，可以做到用户体验不中断。

![image](http://oss.longmarch.work/k8s-3-7.png)

![image](http://oss.longmarch.work/k8s-3-8.png)

## 蓝绿发布配置文件
### petclinic-pod.yml
![image](http://oss.longmarch.work/k8s-3-9.png)

### petclinic-service.yml
![image](http://oss.longmarch.work/k8s-3-10.png)

## K8S Service命令
```sheel
kubectl get all  查看本机k8s环境 查看所有运行的资源
kubectl apply -f . 把当前目录下所有发布文件一次性发布
kubectl  get po --show-labels 展示所有pod 并且展示pod的labels
kubectl describe  svc petclinic 查看petclinic服务的执行详情
kubectl  delete po --all 删除所有pod
kubectl  delete svc--all 删除所有service
```

## 蓝绿发布小结
- 蓝绿发布是一种应用升级发布方式
- 通过Service配合Label/Selector可以实现蓝绿发布

# 应用集群抽象ReplicaSet

## 学习内容
- K8s ReplicaSet副本集原理
- 通过ReplicaSet实现Petclinic应用集群

## 传统应用集群和K8s应用集群对比
- 传统应用集群

![image](http://oss.longmarch.work/k8s-3-11.png)

- k8s 应用集群

![image](http://oss.longmarch.work/k8s-3-12.png)

- 在k8s中通过ReplicaSet 实现应用集群的高可用部署

![image](http://oss.longmarch.work/k8s-3-13.png)

```doc
pod在k8s中是不固定的概念 pod可能会宕机 如果pod宕机 是不会自动恢复的 

ReplicaSet 是对一组pod组成的应用集群的包装  可以保证他所包装的应用集群的高可用 如果有pod宕机，ReplicaSet会自动负责重启pod 保证应用集群的数量 保证应用集群的高可用

因为service ReplicaSet 等其他概念 都是有软件定义的抽象概念 没有直接的物理实体对应 所以K8S又被称为软件定义的数据中心，可以直接运行在笔记本电脑上，和传统物理数据中心有根本差别。
```

## ReplicaSet发布配置文件

### ReplicaSet发布规范样例

![image](http://oss.longmarch.work/k8s-3-14.png)

![image](http://oss.longmarch.work/k8s-3-15.png)

## ReplicaSet相关命令行脚本
![image](http://oss.longmarch.work/k8s-3-16.png)
```sheel
kubectl describe rs pecinic (查看 ReplicaSet执行详情)

kubectl delete rs petclinic （删除ReplicaSet，所关联的pod都会宕机 随后被删除）

```

## ReplicaSet学习小结
- K8s中的ReplicaSet副本集可以实现应用集群
- 发布规范
  - Kind：ReplicaSet
  - apiVersion:extensions/吧deta1
- ReplicaSet具有自愈(Self-healing)能力

# 滚动发布抽象Deployment

## 主要学习内容
- 滚动发布(Rolling Update)原理
- K8s Deployment原理
- 使用Deployment实现应用的滚动发布

## 简单了解滚动发布
### 什么是滚动发布 Rolling Update
    一种高级发布策略,按批次依次替换老版本,逐步升级到新版本。发布过程中,应用不中断,用户体验平滑。

![image](http://oss.longmarch.work/k8s-3-17.png)

## 蓝绿发布VS滚动发布

![image](http://oss.longmarch.work/k8s-3-18.png)

## 简单了解K8s抽象Deployment
![image](http://oss.longmarch.work/k8s-3-19.png)

    K8s中的Deployment可以认为是一种滚动发布抽象 他在ReplicaSet的基础上再做一次封装 Deployment是将ReplicaSet,将滚动发布的流程和ReplicaSet一起包装起来

## Deployment滚动发布流程
![image](http://oss.longmarch.work/k8s-3-20.png)

```doc
**Deployment滚动发布流程**：通过Deployment调度实现滚动发布，应用的老版本V1.0.0已发布，对应的ReplicaSet是V1.0.0，可以通过Deployment进行升级发布应用的V1.0.1，发布后Deployment会创建一个新的ReplicaSetV1.,0.1，之后Deployment会依次滚动，不断的创建并且拉入新的蓝色版本的Pod。拉出并且关闭老的绿色版本的Pod，直到新的蓝色版本的Pod全部上线，绿色版本的Pod全部下线。

这个过程当中，Deployment会保证始终有Pod处于可用状态，服务不会中断，另外前置的Service抽象会屏蔽掉内部的Pod地址的变化，让消费方Client对整个发布过程无感知，如果在滚动发布的过程中，蓝色Pod有问题，健康检查不通过，Deployment会自动终止并且回退发布，即使发布成功，后续也可以根据需要通过Deployment回退到某个版本。

所以Deployment是一种更灵活的发布机制。如果不使用Deployment，仅通过ReplicaSet也可以通过手工方式完成滚动发布，做法就是Deployment做的事情通过手工来完成，但是过程非常的繁琐，而且容易出错。Deployment的抽象机制的引入把滚动发布的实现机制给封装和自动化了。
```

## Deployment发布配置文件

### Deployment发布规范样例
![image](http://oss.longmarch.work/k8s-3-21.png)

### petClinic应用Deployment发布规范
![image](http://oss.longmarch.work/k8s-3-23.png)
```doc
spec 下的selector下的matchLoabels的标签要和template下面的labels一致 表示 这个Deployment要选择和管理的是带有 app：petclinic的这些pod 如果遗漏了 selector下的matchLoabels的标签，kubectl会提示验证失败

minReadySeconds： 表示pod起来后，要等待10秒才算就绪，可以方便我们查看滚动发布的效果

maxSurge: 表示在更新过程中可以额外创建多少个pod  默认是25%

maxUnavailabe:表示在更新过程中最多多少个pod可以处于不可用状态，默认是25%.
```

### PetClinic的Service发布规范

![image](http://oss.longmarch.work/k8s-3-24.png)

## Deloyment 命令脚本
```sheel
kubectl rollout history deployment/petclinic 查看petclinic 版本发布历史

kubectl rollout  undo deployment/petclinic 版本回退 （也是滚动式的）

kubectl rollout status deployment/petclinic  动态显示版本滚动发布状态

kubectl rollout  undo deployment/petclinic -- to-revision=2 回退到指定版本

kubectl  delete deploy petclinic  删除指定deployment
```
## Deployment学习小结
- 滚动发布是一种高级发布机制,支持按批次滚动发布，用户体验不中断,适用于版本兼容发布.蓝绿发布则适用于版本不兼容发布。
- K8s中的Deployment是对ReplicaSet+滚动发布流程的一种包装。
- 发布机制类似ReplicaSet
    - kind: Deployment
    - selector -> matchLabels