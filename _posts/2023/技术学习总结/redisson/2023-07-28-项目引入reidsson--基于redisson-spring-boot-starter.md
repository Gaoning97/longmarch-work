---
layout: post
title: 引入reidsson--基于redisson-spring-boot-starter
tags: redisson学习
categories: redisson学习
---

<a name="o0wWl"></a>

## gradle

```yaml
implementation("org.redisson:redisson-spring-boot-starter:3.20.1")
```

<a name="JyRZK"></a>

## yaml

```yaml
spring:
  redis:
    host: xxx
    port: 6379
    database: 0
    password: xxx
    timeout: 3600
    lettuce:
      pool:
        max-active: 3
        max-wait: 5
        max-idle: 3
        min-idle: 0
    client-name: redisson_demo
    redisson:
#       file: classpath:redisson.yaml
      config: |
         threads: 16
         nettyThreads: 16
         singleServerConfig:
           address: "redis://${spring.redis.host}:${spring.redis.port}"
           password: ${spring.redis.password}
           subscriptionConnectionPoolSize: 50
           connectionPoolSize: 100
           idleConnectionTimeout: 10000
           connectTimeout: 10000
           timeout: 3000
           retryInterval: 1500
           pingConnectionInterval: 1000
           dnsMonitoringInterval: 1000 
```

or

```yaml
spring:
  redis:
    host: xxx
    port: 6379
    database: 0
    password: xxx
    timeout: 3600
    lettuce:
      pool:
        max-active: 3
        max-wait: 5
        max-idle: 3
        min-idle: 0
    client-name: redisson_demo
    redisson:
		  file: classpath:redisson.yaml
```


```yaml
threads: 16
nettyThreads: 16
singleServerConfig:
  address: "redis://${spring.redis.host}:${spring.redis.port}"
  password: ${spring.redis.password}
  subscriptionConnectionPoolSize: 50
  connectionPoolSize: 100
  idleConnectionTimeout: 10000
  connectTimeout: 10000
  timeout: 3000
  retryInterval: 1500
  pingConnectionInterval: 1000
  dnsMonitoringInterval: 1000 
```

<a name="Zts3w"></a>

## 初始化源码：<br />
[redisson-spring-boot -starter初始化过程](https://www.yuque.com/raven-jhxq3/mlcdp1/ev4q7wvxpow8kcv6?view=doc_embed)

<a name="dYAUe"></a>

## 加载配置分析：

[基于redisson学习加载"文本"配置文件](https://www.yuque.com/raven-jhxq3/mlcdp1/sh0lxodfia1s3g6h?view=doc_embed)

