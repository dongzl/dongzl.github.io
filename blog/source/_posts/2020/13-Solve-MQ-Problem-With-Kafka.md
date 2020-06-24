---
title: Kafka 是如何解决使用 MQ 中容易出现的一些问题
date: 2020-03-16 20:58:08
cover: https://gitee.com/dongzl/article-images/raw/master/cover/kafka_study.png
# author information, multiple authors are set to array
# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨在总结分析对于使用 MQ 中容易出现的问题，Kafka是如何解决的。

categories: 
  - 架构设计
tags: 
  - Kafka
---

## 背景描述

目前，很多的业务系统都会或多或少的使用各种 `MQ` 消息队列框架，例如：`RabbitMQ`、`Kafka`、`RocketMQ` 等等，使用 `MQ` 消息队列，可以满足业务系统 `解耦`、`异步`、`削峰填谷` 场景的需要，但是使用 `MQ` 中也不得不面临一些问题，这些问题总结如下：

> 1、如何保证消息队列的高可用？
> 2、如何保证消息不被重复消费？（如何保证消息消费的幂等性）
> 3、如何保证消息的可靠传输？（如何处理消息丢失的问题）
> 4、如何保证消息的顺序性？

其实这些问题也是在面试中 `MQ` 经常被问到的问题，下面我们就总结分析一下，`Kafka` 中是如何解决上述问题的。

## Kafka 如何保证高可用

`Kafka` 的基本架构组成是：由多个 `broker` 组成一个集群，每个 `broker` 是一个节点；当创建一个 `topic` 时，这个 `topic` 会被划分为多个 `partition`，每个 `partition` 可以存在于不同的 `broker` 上，每个 `partition` 只存放一部分数据。

这就是**天然的分布式消息队列**，就是说一个 `topic` 的数据，是**分散放在多个机器上的，每个机器就放一部分数据**。

在 `Kafka 0.8` 版本之前，是没有 `HA` 机制的，当任何一个 `broker` 所在节点宕机了，这个 `broker` 上的 `partition` 就无法提供读写服务，所以这个版本之前，`Kafka` 没有什么高可用性可言。

在 `Kafka 0.8` 以后，提供了 `HA` 机制，就是 `replica` 副本机制。每个 `partition` 上的数据都会同步到其它机器，形成自己的多个 `replica` 副本。所有 `replica` 会选举一个 `leader` 出来，消息的生产者和消费者都跟这个 `leader` 打交道，其他 `replica` 作为 `follower`。写的时候，`leader` 会负责把数据同步到所有 `follower` 上去，读的时候就直接读 `leader` 上的数据即可。`Kafka` 负责均匀的将一个 `partition` 的所有 `replica` 分布在不同的机器上，这样才可以提高容错性。

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/13-Solve-MQ-Problem-With-Kafka/Solve-MQ-Problem-With-Kafka-01.png" width="800px">

拥有了 `replica` 副本机制，如果某个 `broker` 宕机了，这个 `broker` 上的 `partition` 在其他机器上还存在副本。如果这个宕机的 `broker` 上面有某个 `partition` 的 `leader`，那么此时会从其 `follower` 中重新选举一个新的 `leader` 出来，这个新的 `leader` 会继续提供读写服务，这就有达到了所谓的高可用性。

写数据的时候，生产者只将数据写入 `leader` 节点，`leader` 会将数据写入本地磁盘，接着其他 `follower` 会主动从 `leader` 来拉取数据，`follower` 同步好数据了，就会发送 `ack` 给 `leader`，`leader` 收到所有 `follower` 的 `ack` 之后，就会返回写成功的消息给生产者。

消费数据的时候，消费者只会从 `leader` 节点去读取消息，但是只有当一个消息已经被所有 `follower` 都同步成功返回 `ack` 的时候，这个消息才会被消费者读到。

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/13-Solve-MQ-Problem-With-Kafka/Solve-MQ-Problem-With-Kafka-02.png" width="800px">

## 使用 Kafka，如何保证消息不被重复消费

### Kafka 在什么场景下会导致消费者消费到重复数据

`Kafka` 实际上有个 `offset` 的概念，就是每个消息写到 `partition` 里，都会有一个 `offset`，代表消息的序号，在 `consumer` 消费了数据之后，**每隔一段时间（定时定期）**，`consumer` 会把自己消费过的消息的 `offset` 提交一下，表示**这些消息已经消费过了，如果发生异常，下次我将从上次消费到的 offset 来继续消费**。

但是这个过程还是会有意外出现，比如 `consumer` 机器突然宕机，这会导致 `consumer` 有些消息已经处理了，但是没来得及提交 `offset`，这个时候重启之后，就会有少数宕机之前会来得及提交 `offset` 的消息会被再消费一次。

### 如何解决消费到重复数据问题

对于重复消费的问题，我理解对于任何 `MQ` 消息队列都是无法避免的，所以就需要我们在业务系统进行**重复消费的幂等性**验证，当然也要考虑业务场景需要，比如一个日志采集系统，每天上千万的数据，某条重复数据的影响几乎可以忽略不计，这个时候也是不需要考虑；如果是其他场景，需要保证幂等性，可以从如下几方面考虑：

- 如果是写数据库，可以将消息中的某个字段做为数据库唯一约束，在数据处理之前，先根据唯一约束查询一下，判断是否已经消费过；
- 如果是写入 `Redis`，可以直接调用 `set` 方法，天然具备幂等性；
- 在生产者发送每消息的时候，在消息里面加一个全局唯一的 `id`，类似订单 `id` 之类的数据，当消费到这个数据之后，先根据这个全局 `id` 去比如 `Redis` 里查一下，如果不存在，说明没有消费过，就继续处理，然后这个 `id` 写 `Redis`；如果存在，说明已经消费过，消息直接丢弃。

## 使用 Kafka，如何保证消息的可靠传输

对于保证消息的可靠传输，其实就是如何解决消息丢失的问题；那么我们首先需要弄明白消息为什么会丢失，对于一个消息队列，会有 `生产者`、`MQ`、`消费者` 这三个角色，在这三个角色数据处理和传输过程中，都有可能会出现消息丢失，

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/13-Solve-MQ-Problem-With-Kafka/Solve-MQ-Problem-With-Kafka-03.png" width="800px">

下面我们就结合 `Kafka` 来分析一下消息丢失的原因以及解决办法：

### 消费者异常导致的消息丢失

消费者可能导致数据丢失的情况是：消费者获取到了这条消息后，还未处理，`Kafka` 就自动提交了 `offset`，这时 `Kafka` 就认为消费者已经处理完这条消息，其实消费者才刚准备处理这条消息，这时如果消费者宕机，那这条消息就丢失了。

消费者引起消息丢失的主要原因就是消息还未处理完 `Kafka` 会自动提交了 `offset`，那么只要关闭自动提交 `offset`，消费者在处理完之后手动提交 `offset`，就可以保证消息不会丢失。但是此时需要注意重复消费问题，比如消费者刚处理完，还没提交 `offset`，这时自己宕机了，此时这条消息肯定会被重复消费一次，这就需要消费者根据实际情况保证幂等性。

### Kafka 导致的消息丢失

`Kafka` 导致的数据丢失一个常见的场景就是 `Kafka` 某个 `broker` 宕机，，而这个节点正好是某个 `partition` 的 `leader` 节点，这时需要重新重新选举该 `partition` 的 `leader`。如果该 `partition` 的 `leader` 在宕机时刚好还有些数据没有同步到 `follower`，此时 `leader` 挂了，在选举某个 `follower` 成 `leader` 之后，就会丢失一部分数据。

对于这个问题，`Kafka` 可以设置如下 4 个参数，来尽量避免消息丢失：

- 给 `topic` 设置 `replication.factor` 参数：这个值必须大于 `1`，要求每个 `partition` 必须有至少 `2` 个副本；
- 在 `Kafka` 服务端设置 `min.insync.replicas` 参数：这个值必须大于 `1`，这个参数的含义是一个 `leader` 至少感知到有至少一个 `follower` 还跟自己保持联系，没掉队，这样才能确保 `leader` 挂了还有一个 `follower` 节点。
- 在 `producer` 端设置 `acks=all`，这个是要求每条数据，必须是写入所有 `replica` 之后，才能认为是写成功了；
- 在 `producer` 端设置 `retries=MAX`（很大很大很大的一个值，无限次重试的意思）：这个参数的含义是一旦写入失败，就无限重试，卡在这里了。

### 生产者数据传输导致的消息丢失

对于生产者数据传输导致的数据丢失主常见情况是生产者发送消息给 `Kafka`，由于网络等原因导致消息丢失，对于这种情况也是通过在 **producer** 端设置 **acks=all** 来处理，这个参数是要求 `leader` 接收到消息后，需要等到所有的 `follower` 都同步到了消息之后，才认为本次写成功了。如果没满足这个条件，生产者会自动不断的重试。

[面试官问我如何保证Kafka不丢失消息?我哭了！](https://mp.weixin.qq.com/s/qttczGROYoqSulzi8FLXww)

## Kafka 如何保证消息的顺序性

在某些业务场景下，我们需要保证对于有逻辑关联的多条MQ消息被按顺序处理，比如对于某一条数据，正常处理顺序是`新增-更新-删除`，最终结果是数据被删除；如果消息没有按序消费，处理顺序可能是`删除-新增-更新`，最终数据没有被删掉，可能会产生一些逻辑错误。对于如何保证消息的顺序性，主要需要考虑如下两点：

- 如何保证消息在 `MQ` 中顺序性；
- 如何保证消费者处理消费的顺序性。

### 如何保证消息在 MQ 中顺序性

对于 `Kafka`，如果我们创建了一个 `topic`，默认有三个 `partition`。生产者在写数据的时候，可以指定一个 `key`，比如在订单 `topic` 中我们可以指定订单 `id` 作为 `key`，那么相同订单 `id` 的数据，一定会被分发到同一个 `partition` 中去，而且这个 `partition` 中的数据一定是有顺序的。消费者从 `partition` 中取出来数据的时候，也一定是有顺序的。通过制定 `key` 的方式首先可以保证在 `kafka` 内部消息是有序的。

### 如何保证消费者处理消费的顺序性

对于某个 `topic` 的一个 `partition`，只能被同组内部的一个 `consumer` 消费，如果这个 `consumer` 内部还是单线程处理，那么其实只要保证消息在 `MQ` 内部是有顺序的就可以保证消费也是有顺序的。但是单线程吞吐量太低，在处理大量 `MQ` 消息时，我们一般会开启多线程消费机制，那么如何保证消息在多个线程之间是被顺序处理的呢？对于多线程消费我们可以预先设置 `N` 个内存 `Queue`，具有相同 `key` 的数据都放到同一个内存 `Queue` 中；然后开启 `N` 个线程，每个线程分别消费一个内存 `Queue` 的数据即可，这样就能保证顺序性。当然，消息放到内存 `Queue` 中，有可能还未被处理，`consumer` 发生宕机，内存 `Queue` 中的数据会全部丢失，这就转变为上面提到的**如何保证消息的可靠传输**的问题了。

## 参考资料

- [Kafka学习之路 （三）Kafka的高可用](https://www.cnblogs.com/qingyunzong/p/9004703.html)
- [Kafka 消息丢失与消费精确一次性](https://mp.weixin.qq.com/s/JhK8fHJRJRh32QcZ8SPmyg)