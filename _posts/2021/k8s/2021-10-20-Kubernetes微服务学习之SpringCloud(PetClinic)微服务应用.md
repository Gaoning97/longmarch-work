---
layout: post
title: Kubernetes微服务学习之SpringCloud(PetClinic)微服务应用
tags: K8s
categories: K8s
---
# spring社区版PetClinic微服务项目技术栈及架构简介
![image](http://oss.longmarch.work/k8s-1-1.png)
![image](http://oss.longmarch.work/k8s-1-2.png)

# PetClinic微服务Docker Compose 部署文件简析
**Docker-Compose:**
是可以一键运行/关闭Docker容器(只要在开发测试使用)，规范服务镜像的依赖关系的管理Docker容器的工具。

    docker-compose up 一键启动

    docker-compose down 一键关闭
```yaml
version: '2'

services:
## 服务名称
  config-server:
  ## 镜像名称
    image: springcommunity/spring-petclinic-config-server
  ## 容器名称  
    container_name: config-server
  ## 内容限制    
    mem_limit: 512M
  ## 端口号映射关系    
    ports:
     - 8888:8888

  discovery-server:
    image: springcommunity/spring-petclinic-discovery-server
    container_name: discovery-server
    mem_limit: 512M
    ## 规范容器动动顺序  config-server容器启动后再启动
    depends_on:
      - config-server
      ## 规范容器内部应用启动顺序
      ## 等待config-server服务启动后再启动该服务 
    entrypoint: ["./dockerize","-wait=tcp://config-server:8888","-timeout=60s","--","java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
    ports:
     - 8761:8761

  customers-service:
    image: springcommunity/spring-petclinic-customers-service
    container_name: customers-service
    mem_limit: 512M
    depends_on:
     - config-server
     - discovery-server
    entrypoint: ["./dockerize","-wait=tcp://discovery-server:8761","-timeout=60s","--","java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
    ports:
    - 8081:8081

  visits-service:
    image: springcommunity/spring-petclinic-visits-service
    container_name: visits-service
    mem_limit: 512M
    depends_on:
     - config-server
     - discovery-server
    entrypoint: ["./dockerize","-wait=tcp://discovery-server:8761","-timeout=60s","--","java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
    ports:
     - 8082:8082

  vets-service:
    image: springcommunity/spring-petclinic-vets-service
    container_name: vets-service
    mem_limit: 512M
    depends_on:
     - config-server
     - discovery-server
    entrypoint: ["./dockerize","-wait=tcp://discovery-server:8761","-timeout=60s","--","java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
    ports:
     - 8083:8083

  api-gateway:
    image: springcommunity/spring-petclinic-api-gateway
    container_name: api-gateway
    mem_limit: 512M
    depends_on:
     - config-server
     - discovery-server
    entrypoint: ["./dockerize","-wait=tcp://discovery-server:8761","-timeout=60s","--","java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
    ports:
     - 8080:8080

  tracing-server:
    image: openzipkin/zipkin
    container_name: tracing-server
    mem_limit: 512M
    environment:
    - JAVA_OPTS=-XX:+UnlockExperimentalVMOptions -Djava.security.egd=file:/dev/./urandom
    ports:
     - 9411:9411

  admin-server:
    image: springcommunity/spring-petclinic-admin-server
    container_name: admin-server
    mem_limit: 512M
    depends_on:
     - config-server
     - discovery-server
    entrypoint: ["./dockerize","-wait=tcp://discovery-server:8761","-timeout=60s","--","java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
    ports:
     - 9090:9090

  ## Grafana / Prometheus

  grafana-server:
    build: ./docker/grafana
    container_name: grafana-server
    mem_limit: 256M
    ports:
    - 3000:3000

  prometheus-server:
    build: ./docker/prometheus
    container_name: prometheus-server
    mem_limit: 256M
    ports:
    - 9091:9090

```

**简析doker-compose.yml注意点:**

- dopends_on 定义容器依赖和启动次序
- dockerrize命令规范容器内应用依赖和启动次序
- 通过mem_limit限制容器可以内存,配合JVM UseCGroupMemoryLimitForHeap参数完成限制
- Docker-compose会内键一个网络,服务之间可以通过服务名相互访问

# PetClinic微服务的基本功能描述
- Angularjs 主界面
- Eureka Server UI 服务注册中心
- Admin Server UI 监控/配置服务实例内部状态
- zipkin Tracing Server UI 调用链和依赖监控
- Promethues Metircs监控作图
- Grafana Dashboard + JMeter Metrics 展示 + 压测


# Springcloud版PetClinic微服务 与 K8s版PetClinic微服务架构和技术栈差异

## 微服务公共关注点
![image](http://oss.longmarch.work/k8s-1-3.png)

## 系统架构对比

![image](http://oss.longmarch.work/k8s-1-4.png)

![image](http://oss.longmarch.work/k8s-1-7.png)

## 技术栈差异

![image](http://oss.longmarch.work/k8s-1-6.png)

## Springcloud版PetClinic微服务 VS K8s版PetClinic微服务
- SC未解决服务自动化部署问题,K8s是容器化微服务调度发布平台
- K8s内置支持发现 + loadbalance
- k8s平台对具体服务架构无关
- SC：组件框架式架构 K8s：平台型架构
- K8S是面向容器和微服务的云计算平台




