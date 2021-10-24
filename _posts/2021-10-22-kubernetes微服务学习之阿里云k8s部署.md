layout: post
title: kubernetes微服务学习之阿里云k8s部署
tags: K8s
categories: K8s
---
# 学习内容
- 将Petclinic单体解耦拆分为微服务架构
- 在阿里云部署发布微服务

# 将Petclinic单体解耦拆分为微服务架构
## petclinic微服务架构
![image](http://oss.longmarch.work/k8s-5-1.png)

![image](http://oss.longmarch.work/k8s-5-2.png)

![image](http://oss.longmarch.work/k8s-5-3.png)

## DeckerFile文件描述
```sheel
FROM openjdk：8-
使用openJdk

ARG AARTIFACT_NAME  EXPOSED_PORT
传递参数 artifact_name exposed_port

ADD ${ARTIFACT_NAME}.jar /usr/share/app.jar 
把springboot构建出的jar 文件添加到镜像的 /usr/share/app.jar  

EXPOSE ${EXPOSED_PORT}
文档的作用

ENTRPOINT 
指定镜像要启动哪一个命令
```
## springcloud gateway 微服务yaml
springcloud gateway  主要作用是反向路由、转发 找到后台的微服务做调用。

![image](http://oss.longmarch.work/k8s-5-4.png)

## K8s部署配置文件

![image](http://oss.longmarch.work/k8s-5-5.png)

![image](http://oss.longmarch.work/k8s-5-6.png)

![image](http://oss.longmarch.work/k8s-5-7.png)

![image](http://oss.longmarch.work/k8s-5-8.png)

![image](http://oss.longmarch.work/k8s-5-9.png)

![image](http://oss.longmarch.work/k8s-5-10.png)

## 进入dashboard页面
![image](http://oss.longmarch.work/k8s-5-11.png)

![image](http://oss.longmarch.work/k8s-5-12.png)

```sheel
kubectl get ns （查看所有的namespace）
kubectl get secret -n kubernetes-dashboard (查看指定namespace下所有的secret)
kubectl describe secret kubernetes-dashboard-token-qsbwf -n kubernetes-dashboard (查看指定namespace下指定secret的执行详情)
kubectl proxy (启动k8s 代理)
```

# Petclinic微服务的阿里云K8S发布

## 部署架构
![image](http://oss.longmarch.work/k8s-5-13.png)

```doc
VPC: Virtual Private Cloud 虚拟专有云 阿里云K8S集群要求部署在VPC中。

CIDR: IP地址空间

SLB：Server LoadBalancer 在管理端Master节点一般由多个节点组成高可用的集群，为了让管理员能够通过kubectl 命令行工具 以高可用的方式访问APIServer 因为ApiServer肯定是有多台的，
所以前置一般都会有 SLB 负载均衡设备，通过SLB就可以高可用和负载均衡的方式访问ApiServer

NAT: 为了让外部的网络和VPC能够互通，一般需要部署 NAT(网络地址转换设备)，
通过NAT管理员可以SSH到Master节点或者Worker节点上，做一些运维管理的操作，
另外通过NAT VPC内部的节点也可以访问外部的网络，比如拉取docker镜像。

应用端SLB： 在应用端，当应用被部署到K8S集群，一般也需要部署前置的SLB负载均衡设备，
这样外部的流量才可以访问到K8S内部的应用，同时实现高可用和负载均衡调用，应用端SLB一般可以通过发布文件动态创建，发布时，serviceType是LoadBalancer。
```


![image](http://oss.longmarch.work/k8s-5-14.png)

## Petclinic微服务配置

![image](http://oss.longmarch.work/k8s-5-15.png)

![image](http://oss.longmarch.work/k8s-5-16.png)

![image](http://oss.longmarch.work/k8s-5-17.png)

![image](http://oss.longmarch.work/k8s-5-18.png)

![image](http://oss.longmarch.work/k8s-5-19.png)

![image](http://oss.longmarch.work/k8s-5-20.png)

## 阿里云环境配置

![image](http://oss.longmarch.work/k8s-5-21.png)

![image](http://oss.longmarch.work/k8s-5-22.png)

![image](http://oss.longmarch.work/k8s-5-23.png)

![image](http://oss.longmarch.work/k8s-5-24.png)

![image](http://oss.longmarch.work/k8s-5-25.png)

![image](http://oss.longmarch.work/k8s-5-26.png)

![image](http://oss.longmarch.work/k8s-5-27.png)

## 配置本地kubectl 连接k8s集群
![image](http://oss.longmarch.work/k8s-5-28.png)

## 部署configMap

![image](http://oss.longmarch.work/k8s-5-29.png)

![image](http://oss.longmarch.work/k8s-5-30.png)

***扩容缩容：
修改发布文件，达到扩容缩容
kubectl edit deployment 修改配置文件***

