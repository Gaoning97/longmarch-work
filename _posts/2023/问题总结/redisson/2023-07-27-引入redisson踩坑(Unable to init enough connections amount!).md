---
layout: post
title: 引入redisson踩坑(Unable to init enough connections amount!)
tags: redisson踩坑
categories: redisson踩坑
---
## 业务背景:

项目中有业务需要用到分布式锁的功能,而redisson对于分布式锁有比较好的封装,于是遍着手开干接入redisson。

## 版本依赖:

项目中用到的redis版本为(spring-boot-starter-data-redis:2.3.8.RELEASE)，redis是**买的阿里云的云数据库**， 引入的redisson版本为(redisson-spring-boot-starter:3.13.6), 没有做其他自定义配置，而是直接采用redisson-starter的默认配置，所以引入redisson是非常简单的，接入jar包依赖即可。

## 问题初见端倪

**项目**接入redisson后，在进行单元测试的时候，经常性的会出现
RedisConnectionException：Unable to init enough connections amount! Only 15 of 24 were initialized的错误，即使是没有做任何事情的测试用例，也会经常性的出现问题，详细的代码以及错误信息如下：

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = RedissonDemoApplication.class)
@Slf4j
public class RedissonDemoApplicationTests {


@Resource
private RedissonClient redissonClient;

@Test
public void contextLoads() throws IOException {
System.out.println("Test");
}

}
```

![image.png](https://cdn.nlark.com/yuque/0/2023/png/27220920/1690436766051-7c9b4188-3c95-40ed-812d-79834cbb286e.png#averageHue=%23dcd9d9&clientId=u16d0ae05-eec6-4&from=paste&height=518&id=u14aab16d&originHeight=1036&originWidth=2712&originalType=binary&ratio=2&rotation=0&showTitle=false&size=706767&status=done&style=none&taskId=u4c44a64c-d38b-44f3-9c62-c5fe4d22bb7&title=&width=1356)
根据错误信息提示开始排查尝试解决问题。

## 解决过程

### Round1:怀疑redisson默认配置是否不合理

根据报错信息 [google一下](https://www.google.com/search?q=RedisConnectionException%EF%BC%9AUnable+to+init+enough+connections+amount!+Only+15+of+24+were+initialized&oq=RedisConnectionException%EF%BC%9AUnable+to+init+enough+connections+amount!+Only+15+of+24+were+initialized&aqs=chrome..69i57.808j0j7&sourceid=chrome&ie=UTF-8),搜索后发现,有蛮多的人有类似的报错,多数解决方案提到修改默认的配置信息,譬如超时时间改大一点,初始化连接调小,空闲时间调小,等等...

![image.png](https://cdn.nlark.com/yuque/0/2023/png/27220920/1690270136649-d2bfe876-50dc-4fc1-ad90-21a477b93fba.png#averageHue=%23d6d6d6&clientId=uda8fc9e0-4e6a-4&from=paste&height=452&id=KnMBV&originHeight=904&originWidth=1948&originalType=binary&ratio=2&rotation=0&showTitle=false&size=143193&status=done&style=none&taskId=u7f16b5da-0c56-4398-ad5e-39026d25677&title=&width=974)

如下：

```java
@Configuration
public class RedissonAutoConfiguration {


@Value("${redisson.address}")
private String addressUrl;
@Value("${redisson.password}")
private String password;

@Bean
public RedissonClient getRedisson() {
Config config = new Config();
config.useSingleServer()
.setAddress(addressUrl)
.setPassword(password)
.setRetryInterval(5000)
.setTimeout(10000)
.setConnectTimeout(10000)
// 设置连接数为1
.setConnectionMinimumIdleSize(1);
return Redisson.create(config);
}
}
```


然而，并没什么X用，即使是设置连接数设置成1，还是一样的报错。

G~

### Round2:怀疑是redisson的问题

继续排查(google)~
因为即使连接池的个数设置为1，redisson还是会出现报错，不能申请到足够的连接数。于是便怀疑的redisson的问题。
果然，在redisson的github中，查到有人有类似的问题，而redisson官方给出的回应为，这是一个bug，他们已经更新了redisson的版本，只要吧redisson版本更新到指定版本就可以了。
[Unable to init enough connections amount! Only 0 of 1 were initialized. · Issue #4902 · redisson/redisson](https://github.com/redisson/redisson/issues/4902)
如下：

![image.png](https://cdn.nlark.com/yuque/0/2023/png/27220920/1690270075290-c98dc914-1c4d-4cac-833a-6d7a3c476110.png#averageHue=%23c9c9c9&clientId=uda8fc9e0-4e6a-4&from=paste&height=235&id=uc32282d4&originHeight=470&originWidth=2694&originalType=binary&ratio=2&rotation=0&showTitle=false&size=119213&status=done&style=none&taskId=u37caacae-6d3b-4140-bf58-30e1e2e4053&title=&width=1347)

![image.png](https://cdn.nlark.com/yuque/0/2023/png/27220920/1690270067382-fccff2d4-9558-4b99-9998-eaa023a4d439.png#averageHue=%23c7c7c7&clientId=uda8fc9e0-4e6a-4&from=paste&height=258&id=u12a751a7&originHeight=516&originWidth=1354&originalType=binary&ratio=2&rotation=0&showTitle=false&size=56951&status=done&style=none&taskId=u9ba97edf-7122-4788-aa69-c91689f4907&title=&width=677)
跟新版本后再试一下，然而，依旧没什么X用。
GG~

### Round3:怀疑阿里云redis配置设置是否有误

Redisson在启动时会根据配置申请足够数量的连接放入连接池。连接池是一组预先建立的与Redis服务器建立连接的连接对象，这些连接对象在Redisson启动时被创建，并且可以根据配置的最大连接数来决定创建多少个连接。
启动时redisson默认会多次访问redis获取连接，放入连接池中：

**因此怀疑是否在启动时刚好未能申请到连接,导致连接超时?**

官方文档中描述 redisson初始化时默认会申请32个连接(老版本为24个)

### ![image.png](https://cdn.nlark.com/yuque/0/2023/png/27220920/1690279261255-e31cacb4-0620-4c94-9ab0-994d91ae4db7.png#averageHue=%23eaeaea&clientId=uda8fc9e0-4e6a-4&from=paste&height=776&id=c9P8X&originHeight=1552&originWidth=2490&originalType=binary&ratio=2&rotation=0&showTitle=false&size=390849&status=done&style=none&taskId=u57fd8c09-dccf-47dc-8831-7de5fd147e8&title=&width=1245)

查看阿里云redis配置信息
![image.png](https://cdn.nlark.com/yuque/0/2023/png/27220920/1690279587800-daf59271-68c6-4923-bdcb-76988cf18de2.png#averageHue=%23ecedee&clientId=uda8fc9e0-4e6a-4&from=paste&height=489&id=TUNW1&originHeight=978&originWidth=1612&originalType=binary&ratio=2&rotation=0&showTitle=false&size=221911&status=done&style=none&taskId=ua09ec8b0-270a-4498-9aa7-dc9bccc4b79&title=&width=806)

最大客户端连接数为10000,远远大于申请所需要的32个请求.

GGG了~

因为有蛮多的业务要进行开发，所以中场休息。。
---------------------------------------------------------------

中场：耻辱结束游戏，注释掉redisson相关代码。改用redis实现分布式锁
---------------------------------------------------------------

有时间了！再战！
---------------------------------------------------------------

再战！
---------------------------------------------------------------

### Round4:怀疑是否是版本依赖或者环境有问题

因为之前没有使用过redisson ，所以怀疑是否是和版本依赖or环境有关系，导致出现问题

##### 最开始的环境搭配

1.org spring-boot-starter + 公司阿里云 redis❎
2.org redisson + 公司阿里云 redis❎

- 由springboot starter的redisson改为org 的redissonr(害怕springboot自动装配时做了什么额外的配置，导致出现问题)

然而依旧是不尽人意，翻了一下俩个版本的源码，发现只是创建redissonClient的入口不同，其他的代码都是一样的，（ 简单的源码分析：[redisson-spring-boot -starter初始化过程](https://www.yuque.com/raven-jhxq3/mlcdp1/ev4q7wvxpow8kcv6?view=doc_embed))

##### 更改环境的尝试

3.org spring-boot-starter + 自己服务器搭建的redis ✅
4.org redisson + 自己服务器搭建的redis ✅

- 改为使用自己服务器搭建的redis 结果发现无论是使用starter 还是 org原本的redisson 都不会出现问题。

终于！！！但是...
好消息：问题找到了，是因为阿里云redis的问题。
坏消息：不能因为这个小问题把阿里云redis给换了。

GGGG~

### Round5:解决阿里云的问题。

##### 阿里云没问题？

将阿里云reids的配置信息和我自己服务器的redis配置信息导出进行对比后发现，俩者并没有什么区别。
因为对于配置信息不是很明白，所以SOS 求救运维大哥，运维大家帮忙调整了一波配置信息，结果依旧没什么用，还是报错。

此时此刻的我仿佛就像小丑🤡一样，觉得是阿里云的问题，但是找不出问题所在，考虑要不然就梅开二度，再次放弃使用redisson。

中午吃饭的时候运维大哥问我问题解决的怎么样了，我说感觉就是阿里云的问题，但是找不到问题出在哪。运维大哥建议我如果觉得就是阿里云的问题，**不如自己申请一个阿里云reids再试一下**。如果还是找不到问题，那只能给阿里云提工单或者放弃了。

##### 阿里云有问题！

下午的时候说干就干，用自己的号申请了一个阿里云redis，跑了一下demo发现问题消失了！！！

于是便开始对比俩者redis的配置信息，结果发现并没有什么不同。甚至都是用的默认的配置，没什么改动...

直到去一项项对比redis服务器信息时才发现，虽然俩个redis服务器都是redis5.0，但我的是小版本是不同的
![image.png](https://cdn.nlark.com/yuque/0/2023/png/27220920/1690444058082-629599cc-485f-4d48-9a61-21314ca4f432.png#averageHue=%23bababa&clientId=u4ca328d9-c2ed-4&from=paste&height=202&id=u80088ba3&originHeight=404&originWidth=1196&originalType=binary&ratio=2&rotation=0&showTitle=false&size=79833&status=done&style=none&taskId=u40e177a3-7dbc-49a2-97ff-e3e26380b3d&title=&width=598)
很明显我的版本要新的多，于是变查看了阿里云reids的版本更新日志，发现他们在之前的版本中有过一定的更新和优化，公司是在20年的时候就选购了 后面也没有做过升级，自然也没有这些功能。

![image.png](https://cdn.nlark.com/yuque/0/2023/png/27220920/1690361517910-36e165d0-9f20-4e17-b1bb-654f52255409.png#averageHue=%23b7b7b7&clientId=uae592cab-e0fe-4&from=paste&height=154&id=IXLEF&originHeight=308&originWidth=1960&originalType=binary&ratio=2&rotation=0&showTitle=false&size=54769&status=done&style=none&taskId=u38729c79-e904-40d1-b626-bb998fb1987&title=&width=980)

升级！！升级！！

![image.png](https://cdn.nlark.com/yuque/0/2023/png/27220920/1690444164339-d7992e61-24f8-4ea3-9027-40fa5fdf226b.png#averageHue=%23858282&clientId=u4ca328d9-c2ed-4&from=paste&height=102&id=u9612497f&originHeight=204&originWidth=498&originalType=binary&ratio=2&rotation=0&showTitle=false&size=19782&status=done&style=none&taskId=ud7769966-d53a-489c-8c38-f0ba0acffba&title=&width=249)
喊运维大哥帮忙进行了升级，升级后又试下一下，果然，问题解决！！！

## 总结

这个问题出现的主要原因就是因为频繁重启的时候会进行多次获取连接池连接等等，会进行特别多次的请求。
低版本的阿里云redis对应并发情况下请求处理能力不足，导致超时。

