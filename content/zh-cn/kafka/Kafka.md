---
title: "kafka简介"
date: 2020-05-02T20:18:42+08:00
draft: false
type: posts
categories: ["linux"]
tags: ['kafka']
---

### Kafka

#### 一、基本概念

1. kafka cluster：kafka集群，一个集群由多个实例（broker）组成，通过zookeeper进行元数据（brokerID、broker 地址等信息，partition的leader、follower信息等）共享与同步
2. broker：kafka集群中的一个kafka服务实例，集群内每个broker都有一个不重复的编号，如broker-0、broker-1
3. producer：消息生产者
4. topic：消息主题，可以理解为消息分类，为逻辑概念。Kafka的消息就保存在topic，每个broker上都可以创建多个topic。
5. partition：topic的分区，每个topic可以有多个分区，分区的作用是做负载，提高Kafka的吞吐量。同一个topic在不同分区的数据是不重复的，partition的表现形式就是一个个文件夹，为物理概念。
6. replication：分区的副本，副本的作用就是提高Kafka的可靠性和可用性。同一个partition有多个replication的情况下，会通过zookeeper选出一个leader，其余的为follower，producer/consumer 读写partition的时候，就是通过leader进行的。当leader不可用时，会选择一个follower成为leader。在Kafka中，默认的最大副本数量是10个，而且副本的数量不能超过broker数量
7. consumer：消息消费者
8. consumer group：消息消费者组。为了提高Kafka的吞吐量。可以将多个consumer组成一个group，同一个partition数据只能被group中的一个consumer消费。同一个group里的consumer可以消费同一个topic的不同分区的数量。这样也可以保证，同一消息，不会被重复消费

#### 二、Kafka核心规则

- 工作流程

  1. producer先从集群中获取分区的leader
  2. producer将消息发送给leader
  3. leader将消息写入本地文件
  4. follower从leader pull 消息
  5. follower将消息写入本地文件后，向leader发送ACK
  6. leader收到所有副本的ACK后，向producer发送ACK

  - 每条消息是追加到分区中，顺序写入磁盘，所以保证同一分区的数据是有序的
  - producer采用push模式将消息发布到broker
  - follower是主动去leader pull消息进行同步的
  - producer如何确定数据该写到那个partition

  1. partition在写入的时候，可以指定需要写入的partition；如果指定，则写入对应的partition
  2. 如果没有指定partition，但是设置了key，则会根据key的值hash到一个partition
  3. 如果既没有partition，也没有key，则会轮询选出一个partition

- 如何保证消息不丢失：producer写入消息、follower同步leader消息的时候，都会返回ACK，通过ACK应答机制，可以确保消息不丢。在生产者向kafka写数据的时候，可以设定参数acks来确定kafka是否确认收到数据，这个值可以设置为0、1、all
  - 0代表producer消息发往cluster不需要等到cluster返回ACK消息，所以设置这个值不能确保消息发送成功，安全性最低，但是效率最高
  - 1代表producer消息发往cluster时，只要leader应答就可以发送下一条，所以这个设置只能确保发送给leader成功
  - all代表producer消息发往cluster时，需要所有的follower都完成从leader同步，才会发送下一条，确保leader发送成功并且所有follower都完成备份。安全性最高，但是效率最低
- 数据保存
  1. partition结构：每个partition文件夹由多个segment文件组成，每组segment文件包含.index、.log文、.timeindex三个文件，其中.index、.log文件为索引文件，log文件就是存储message的地方。Kafka就是利用segment+index的方式解决查询效率低问题
  2. message结构：message主要包含消息体、消息大小、offset、压缩类型等信息
  3. 存储策略：无论message是否被消费，Kafka都会保存所有消息，对于旧数据删除策略有两种：基于时间，默认7天；基于大小，默认1GB。
- consumer group中consumer和partition的关系：
  - 一个consumer可以消费多个partition中的数据
  - 一个partition只能被一个consumer消费

#### 三、Kafka吞吐量大的原因

#### 四、QA

- leader的选举

  kafka不是多数投票选举leader。kafka动态维护一组同步leader数据的副本（ISR in-sync-replication），只有这个组的成员才有资格当选leader，kafka副本写入不被认为是已提交，直到所有的同步副本已经接收才认为。这组ISR保存在zookeeper，正因为如此，在ISR中的任何副本都有资格当选leader，这是kafka的使用模型，有多个分区和确保leader平衡是很重要的一个重要因素。有了这个模型，ISR和f+1副本，kafka的主题可以容忍f失败而不会丢失已提交的消息。

  成为ISR列表中的一员，需要满足两个条件：

  - 副本节点必须能与zookeeper保持会话（心跳机制💗）
  - 副本能够复制leader上的所有写操作，并且不能落后太多（卡住或滞后的副本由replica.lag.time.max.ms配置）

  一旦follower从leader同步数据的延迟超过阀值，leader就会把它从ISR中剔除，放入OSR，新加入的follower也会加入OSR

  AR(assigned replicas) = ISR+OSR (out of-sync-replicas)

- 如何保证已提交的信息不丢失

  replication+leader选举机制

- 

