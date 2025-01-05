---
layout: post
title: 'MQ 技术与 RocketMQ 集群：秒杀系统的优化与全链路消息可靠性解析'
date: 2024-09-15
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: MQ



---

> 文章主要介绍了 MQ 技术与 RocketMQ 集群在秒杀系统中的应用及相关问题。包括 MQ 的优点、秒杀系统的难点与优化方案、RocketMQ 的架构和组件、消息丢失与积压的解决办法、消费者消息处理等，还探讨了多种相关技术原理和机制。

引入MQ，将同步的时间驱动转为异步的消息驱动，完成服务解耦。

什么是消息驱动？什么是MQ？有什么用？

提到秒杀相关--立即使用mq

提到mq想到：

# 1. MQ的三个优点

- 异步
  - 同步的事件驱动改为异步的消息驱动
  - 快递直接送到家--><font style="color:#E8323C;">菜鸟驿站</font>-->客户自己去取
- 解耦
  - 不同技术、不同语言系统对接
  - Thinking in java--><font style="color:#E8323C;">编辑社</font>-->中文版、法语版、韩文版
- 削峰（最重要的场景，秒杀场景）
  - 用稳定的系统资源处理突发的流量冲击
  - 长江水涨水落--><font style="color:#E8323C;">三峡大坝</font>-->蓄水



# 2. 秒杀有哪些难点？如何优化一个秒杀系统？

1. 秒杀前，页面访问压力大
   1. <font style="color:#117CEE;">解决方案：页面静态化，CDN+Redis+Nginx多级缓存</font>
2. 秒杀时，下单过于集中，作弊软件刷单
   1. <font style="color:#117CEE;">解决方案：前端页面增加答题环节</font>
3. 秒杀时，下单请求对系统冲击大，影响其他正常功能
   1. <font style="color:#117CEE;">解决方案：为秒杀独立一套订单系统</font>
4. 秒杀时，快速精准扣减库存。
   1. <font style="color:#117CEE;">解决方案：基于缓存如Redis实现快速精准扣减库存</font>
5. 秒杀后，快速过滤未抢到的下单请求
   1. <font style="color:#117CEE;">解决方案：库存扣减完后，快速通知Nginx，过滤下单请求</font>
6. <font style="color:#DF2A3F;">秒杀后，下单模块压力大。</font>
   1. <font style="color:#117CEE;">解决方案：下单请求写入MQ，后端下单模块慢慢下单。下单后，也通过MQ通知下游服务，完成下单。</font>

MQ的以下问题如何解决？

- <font style="color:#E8323C;">如何保证消息不丢失？</font>
- <font style="color:#E8323C;">消息积压严重怎么办？</font>
- <font style="color:#E8323C;">如何保证消息不重复消费？</font>
- <font style="color:#E8323C;">如何保证消息消费顺序？</font>
- <font style="color:#E8323C;">RocketMQ如何优化底层数据读写？</font>

![]({{ '/assets/img/RocketMQ/1.png' | prepend: '' }})
![](.\img\RocketMQ\1.png)



# 3. 常用的MQ技术

![]({{ '/assets/img/RocketMQ/2.png' | prepend: '' }})
![](.\img\RocketMQ\2.png)

# 4. RocketMQ的集群架构

![]({{ '/assets/img/RocketMQ/3.png' | prepend: '' }})
![](.\img\RocketMQ\3.png)

RocketMQ架构上主要分为四部分：

1. Producer 消息生产者
2. Consumer 消息消费者
3. NameServer 路由注册中心
4. Broker 服务调度节点

<font style="color:#DF2A3F;">问题：为什么RocketMQ要自己做一个NameServer，而不使用线程的Zookeeper、Nacos、Eureka？</font>

答：NameServer非常轻量级（节点之间不存在数据通信），每个节点上保存全量的broker信息，不需要进行交互（例如选举等操作）。轻量带来的问题：broker有可能在NS1上注册成功，但是在NS2上注册失败，这就会导致两台NameServer上数据不一致，牺牲了数据一致性。基于AP，牺牲了CP。

一谈到微服务就要考虑CAP，根据自己业务定制。

# 5.RocketMQ如何保证全链路消息不丢失？

![]({{ '/assets/img/RocketMQ/4.png' | prepend: '' }})
![](.\img\RocketMQ\4.png)

所有MQ产品消息丢失的元凶：<font style="color:#DF2A3F;">网络+缓存</font>

1. 生产者发送消息到MQ有可能丢失消息
2. MQ收到消息后，写入硬盘时有可能丢失消息
3. 消息写入硬盘后，硬盘坏了，也有可能丢失消息
4. 消费者消费MQ消息，如果进行异步消费，也有可能丢失消息

## 5.1 路由中心挂了怎么办？

![]({{ '/assets/img/RocketMQ/5.png' | prepend: '' }})
![](.\img\RocketMQ\5.png)

问题：

1. <font style="color:#DF2A3F;">NameServer的路由发现与路由剔除机制是什么样的？</font>
2. <font style="color:#DF2A3F;">从CAP理论的角度分析，NameServer保证的是CP还是AP？为什么要这样设计？</font>
3. <font style="color:#DF2A3F;">NameServer全部挂了，客户端还能不能正常工作？</font>
   1. <font style="color:#117CEE;">答：短时间可以。Producer和Consumer本地都有一个本地缓存（缓存Broker信息），所以在短时间内是可以正常工作的（比如Producer一下子发10条消息，发到第5条的时候NS挂了，剩余5条还是可以继续发送的，但是后续还想重新发消息就不能发了；对于Consumer基本上就立即不能用了，Producer和Consumer会不断的向NS发送心跳请求询问是否更新缓存）</font>

## <font style="color:#000000;">5.2 生产者发送消息到MQ消息丢失</font>

方案一：同步发送+多次重试。最通用的方案

方案二：RocketMQ提供的事务消息机制。

从具体的业务场景理解事务消息机制的作用。

![]({{ '/assets/img/RocketMQ/6.png' | prepend: '' }})
![](.\img\RocketMQ\6.png)



问题：

1. <font style="color:#DF2A3F;">理解分布式事务问题</font>
2. <font style="color:#DF2A3F;">half消息如何保证不向下游服务推送？</font>
3. <font style="color:#DF2A3F;">如何控制RocketMQ进行消息状态回查的次数和频率？</font>
   1. <font style="color:#117CEE;">回查15次（transactionCheckMax），可修改</font>
   2. <font style="color:#117CEE;">频率（回查间隔，transactionCheckInterval）。60s，可修改</font>
4. <font style="color:#DF2A3F;">事务消息机制真的只跟生产者端有关吗？</font>

一个订单系统mq的设计（可以作为面试的一个示例--内外网的webservice调用）

1. 使用mq的事务send方法
2. 新增订单作为本地事务的执行，返回mq给unknown 状态
3. 等待mq进行回查，同步订单信息给第三方，并且协会第三方id作为回查的逻辑
4. 如果第三步成功了，就都成功了，如果有一个失败就不发送消息了，把本地事务也回滚即可

## 5.3 消息传到Broker了就真的安全了吗

![]({{ '/assets/img/RocketMQ/7.png' | prepend: '' }})
![](.\img\RocketMQ\7.png)

问题：

1. <font style="color:#E8323C;">PageCache是什么？什么叫刷盘？缓存不安全，我不用缓存不行吗？</font>
2. <font style="color:#E8323C;">同步刷盘和异步刷盘有什么区别？如何进行配置？</font>
3. <font style="color:#E8323C;">同步刷盘就是傻傻等待着写完磁盘吗？那写入消息不就会变得很慢？</font>

Broker磁盘坏了怎么办？--主从备份

普通集群：同步同步VS异步同步？（没有选举）

Dledger集群：有选举

Broker主节点挂了，从节点不会升为主节点，只有等到主节点启动之后才会响应

## 5.4 深入理解Dledger高可用集群

1. 基于Raft协议定期选举主节点

基于Raft协议定期进行主节点选举。主节点负责响应客户端请求。协调从节点完成请求处理逻辑。

2. 接管CommitLog文件写入

接管CommitLog消息写入过程。增加两阶段文件写入。

![]({{ '/assets/img/RocketMQ/8.png' | prepend: '' }})
![](.\img\RocketMQ\8.png)

大厂作死题：

1. <font style="color:#DF2A3F;">Dledger集群选举的过程是什么样的？如何保证高可用？如何防止脑裂问题？</font>
2. <font style="color:#DF2A3F;">Dledger集群如何接管CommitLog文件写入？如何兼容普通集群的客户端消息读写机制？</font>

## 5.5 消费者消息零丢失

<font style="color:#DF2A3F;">消费者端先处理本地事务还是先提交Offset？</font>

正常来说，消费者会不断发请求请求消息，所以不存在消息丢失。

但是有一种情况可能会存在消息丢失，比如以下代码：接收消息时会开一个线程去接收消息，主线程直接返回接收成功，相当于一个异步的操作，这样会提高吞吐量，但是会有消息丢失的风险。

![]({{ '/assets/img/RocketMQ/9.png' | prepend: '' }})
![](.\img\RocketMQ\9.png)

消费者端由于有消息重试机制，通常不会丢失消息。更多的是要考虑消息幂等的问题。

## 5.6 要是整个MQ服务挂了呢？

互联网大厂才会考虑的作死问题：（有个概念就行，不必深究）

<font style="color:#DF2A3F;">如果整个MQ服务挂了，怎么保证消息零丢失？</font>

![]({{ '/assets/img/RocketMQ/10.png' | prepend: '' }})
![](.\img\RocketMQ\10.png)

## 5.7 RocketMQ全链路消息零丢失整体方案

![]({{ '/assets/img/RocketMQ/11.png' | prepend: '' }})
![](.\img\RocketMQ\11.png)

## 5.8 RocketMQ消费者消息零丢失方案总结

1. <font style="color:#DF2A3F;">生产者端如何保证发送消息零丢失？</font>
   1. 方案一：同步发送+多次尝试。 --降低吞吐量
   2. 方案二：事务消息机制。 --多次网络请求
2. <font style="color:#DF2A3F;">MQ收到消息后如何保证零丢失？</font>
   1. 同步刷盘。 --消息写入变慢
   2. Dledger主从架构。 --频繁网络传输
3. <font style="color:#DF2A3F;">消费者消息如何保证零丢失？</font>
   1. 先处理本地事务，再提交Offset。 --不能用异步提升吞吐量
4. <font style="color:#DF2A3F;">如果整过MQ服务挂了怎么保证消息零丢失？</font>
   1. MQ服务不可用，发送消息时增加降级缓存。

## 5.9 RocketMQ消息积压严重怎么办？

![]({{ '/assets/img/RocketMQ/12.png' | prepend: '' }})
![](.\img\RocketMQ\12.png)

1. RocketMQ的客户端负载均衡机制
2. 消费者节点个数不能超过Topic的队列数。如果还是不够怎么办？
   1. 队列搬运。新建topic，多几个队列，将原积压消息的topic消息搬运到新topic，再增加消费者。

## 5.10 源码级理解RocketMQ的延迟队列机制

![]({{ '/assets/img/RocketMQ/13.png' | prepend: '' }})
![](.\img\RocketMQ\13.png)

其实延迟队列机制的本质就是队列搬运，将延迟队列的信息set到系统新建的延迟队列中进行消费。

half消息也是队列搬运。













<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>