---
layout: post
title: Kubernetes微服务学习之Kubernetes基本概念和应用(下)
tags: K8s
categories: K8s
---
# K8s内部反向代理ClusterIp Service
## 主要学习内容
- K8s内部反向代理ClusterIp Service原理
- Petclinic通过ClusterIp Service访问内部MySql Pod

## 内部服务如果相互访问?
![image](http://oss.longmarch.work/k8s-4-1.png)

### 传统内部反向代理
![image](http://oss.longmarch.work/k8s-4-2.png)

### K8s内部反向代理

![image](http://oss.longmarch.work/k8s-4-3.png)

## ClusterIP Service 内部访问MySql Pod 配置
![image](http://oss.longmarch.work/k8s-4-4.png)

![image](http://oss.longmarch.work/k8s-4-5.png)

![image](http://oss.longmarch.work/k8s-4-6.png)

![image](http://oss.longmarch.work/k8s-4-7.png)

    kubectl logs petclinit-s12312c 查看指定pod下的日志

## ClusterIP Service学习小结
- ClusterIP Service 是K8S提供的一种内部反向代理机制
- 复习Service的三种类型
    - ClusterIP 内部服务访问
    - NodePort 对外暴露服务
    - LoadBalancer 对外欂栌服务(公有云)
- Deployment/Pod发布规范中可以添加环境变量,给Pod传递启动参数
- 有状态服务一般只能部署1个Pod实例(如果mysql  但是一般不会把数据库放入K8s中)

# K8S名字空间Namespace和Kube-Dns
## 主要学习内容
- K8S名字空间Namespace原理
- Kube-Dns域名服务原理

## K8s架构部署回顾

![image](http://oss.longmarch.work/k8s-4-8.png)

![image](http://oss.longmarch.work/k8s-4-9.png)


## K8s名字空间抽象Namespace

![image](http://oss.longmarch.work/k8s-4-10.png)
- **Namespace是K8s当中的一种逻辑隔离机制，方便对资源进行逻辑管理，在一个k8s集群当中中可以配置多个Namespace**
- **每个Namespace可以住逻辑上独立的Pods ReplicaSets Services，名字空间的隔离不是物理隔离，实际上不同名字空间的资源根据需要仍然可以进行访问。**

## Namespace相关脚本
    kubectl get ns 可以看到k8s内置支持的namespace 
    kubectl get all -n kube-system  查看kube - system下的所有资源文件
    
## 内部DNS域名服务解析流程

![image](http://oss.longmarch.work/k8s-4-11.png)

```doc
kubectl内部域名解析服务DNS解析流程： petclinic pod 访问mysql service 域名简析过程，当petclinic应用访问mysql service时，
petclinic pod内部操作系统自带的dns client 会自动查询集群内部的dns服务 获得mysql service 对应的clusterIp地址，然后向mysql service 发起调用
```
![image](http://oss.longmarch.work/k8s-4-12.png)

**登录到pod内部 执行里面的sh命令 shell 并且是以终端交互式的方式运行**

![image](http://oss.longmarch.work/k8s-4-13.png)

![image](http://oss.longmarch.work/k8s-4-14.png)

## Namespace和Dns域名解析学习小结
- NameSpace是K8s提供的资源逻辑隔离机制
    - default
    - kube-system
- K8s内部服务名-> ClusterIP转换
    - linux Dns Client + Kube-Dns配合实现
    - Kube-Dns住在Kube-System中
    
# K8s配置抽象ConfigMap
## 主要学习内容
- K8s配置抽象ConfigMap的原理
- Petclinic + ConfigMap应用
- ConfigMap配置更新传播

## K8s配置抽象ConfigMap

![image](http://oss.longmarch.work/k8s-4-15.png)

- k8s平台内置支持微服务配置 ，对应的概念就是ConfigMap，他是K8s平台所支持的一种资源，

- 开发人员将配置填写到configMap当中，发布后K8s支持将ConfigMap当中的配置 以环境变量的方式注入到pod当中，pod中的应用就可以以环境变量的方式来访问配置。

- configMap也支持以存储卷 volume的形式挂载到pod当中，这样pod中的应用就可以以本地配置文件的形式来访问配置

### **微服务间进行共享配置：**

![image](http://oss.longmarch.work/k8s-4-16.png)

![image](http://oss.longmarch.work/k8s-4-17.png)


## 使用ConfigMap后项目架构配置文件变更

![image](http://oss.longmarch.work/k8s-4-18.png)

![image](http://oss.longmarch.work/k8s-4-19.png)

![image](http://oss.longmarch.work/k8s-4-20.png)

![image](http://oss.longmarch.work/k8s-4-21.png)

**通过 “---” 分割 可以将多个发布文件放在一个文件中 一起发布**

![image](http://oss.longmarch.work/k8s-4-22.png)

## ConfigMap相关命令行脚本

```sheel
kebectl get cm 查看configMap的资源

kubectl describe cm petclinic-config 查看指定configMap 下在资源详情(可以看configMap中的配置项)
```

## ConfigMap变更传播

![image](http://oss.longmarch.work/k8s-4-23.png)

只修改configMap 下的配置信息是**不能直接更新petclinic-service中的配置信息**，如果需要同时更新，需要更新configMap metadata下的name 以及petclinic-service下的configMapRef，然后重新发布。

从而进行实时更新configMap属性到petclinic-service，原理是会创建一个新的configMap 删掉老的configMap 

## ConfigMap学习小结
- ConfigMap是K8S提供的一种配置管理抽象,便于在微服务间共享配置。
- ConfigMap可以绑定到Pod的环境变量(Enc)中,配置更新传播:
    - 需重启Pod
    - 建议更新ConfigMap的name和引用
- ConfigMap也可绑定到Pod的持久(Volume),支持配置热更新

# K8s机密配置抽象Secret

## 主要学习内容
- K8s机密配置抽象Secret的原理和演示
- 解释Secret的安全性

## K8s机密配置抽象Secret

![image](http://oss.longmarch.work/k8s-4-24.png)

- k8s提供secret这个抽象概念来支持敏感数据配置。secret可以认为是一种特殊的配置，他提供较安全的存储和访问配置的机制。
- 同样secret也支持俩种常见的方式和pod进行绑定，一种是以环境变量的方式注入到pod当中，另外一种是以存储卷的形式挂载到pod当中。
- secret他的变更传播特性和configMap也一样，如果以环境变量形式和pod进行绑定，那么secret配置变更的时候，需要重启pod才能够获取最新的secret配置信息。
- 如果以存储卷的形式挂载到pod，那么secret可以做到热加载，不必要重启pod。

![image](http://oss.longmarch.work/k8s-4-25.png)

## Secret配置文件

![image](http://oss.longmarch.work/k8s-4-26.png)

![image](http://oss.longmarch.work/k8s-4-27.png)

## Secret配置结果验证

![image](http://oss.longmarch.work/k8s-4-28.png)

![image](http://oss.longmarch.work/k8s-4-29.png)

## Secret相关脚本命令
    kubectl describe secret petclinic-secret 查看petclinic-secret数据详情
    
    kubectl delete secret petclinic-secret 删除

## secret学习小结
- Secret是K8s提供的一种机密配置抽象,便于在微服务间共享机密配置
- Secret仅提供有限安全
    - 协助时防止机密数据泄露
    - 为Secret资源设置单独安全访问策略
- Secret和Pod绑定
    - 环境变量(Env)
    - 持久卷(Volume)
    
