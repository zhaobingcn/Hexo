---
title: Kafka基本原理
date: 2017-04-23 20:23:36
tags:
    - Kafka
    - 消息系统
    - 大数据
categories: Kafka
---
# Kafka简介
## 简介
 Kafka提供了类似JMS（Java Message Service）的特性，但是在设计实现上完全不同，此外，它并不是JMS规范的实现。Kafka对消息保存时根据Topic分类，发送消息的是Producer接收消息的是Consumer，Kafka集群由多个Kafka实例组成，每个实例被称作Broker。无论是Kafka集群，还是Producder和Consumer都依赖于Zookeeper保证系统的可用性和保存系统的Meta数据。
 
## Kafka中的常见名词

> **1.producer** 
消息的生产者，发送消息到Kafka集群或者终端的服务 
> **2.broker** 
集群中包含的服务器
> **3.topic**
每条发送到kafka中消息的类别，即kafka的消息是面向topic的 
> **4.partition**
是物理上的概念，每个topic包含一个或者多个partition。kafka的分配单位是partition 
> **5.consumer**
从kafka集群中消费消息的终端或者服务 
> **6.consumer group**
每个consumer都属于一个consumer group，每条消息只能被consumer group中的一个consumer消费，但是可以被多个consumer group消费
> **7.replica**
partition中的副本，用来保证这部分数据的高可用性 
> **8.leader**
replica中的一个角色，负责消息的读和写，producer和consumer只和leader交互 
> **9.follower**
replica中的一个角色，从leader复制数据，为了保证高可用性，leader故障时可以成为leader 
> **10.controller**
kafka集群中的一个服务器，用来进行leader election和这种failover 
> **11.zookeeper**
协调kafka中的producer，consumer以及保存kafka中的meta数据

拓扑结构如下图：
![](/images/kafka_topology.png)

## 基本性问题介绍
一个Topic可以认为是一类消息，每个Topic将会被分为多个partition（分区），每个partition在存储层面是append log。任何发布到此partition的消息都会被追加到log文件的尾部，每条消息在文件中的位置称为offset（偏移量），offset是一个long型数字，他唯一标记一条信息的位置。kafka并没有提供其他的索引机制来存储offset，因此在Kafka中几乎不允许对消息进行"随机读写"。
![](/images/kafka_partition.png)
Kafka和JMS的实现（ActiveMQ）不同的是：即使消息被消费，消息仍然不会被立即删除。日志文件将会根据broker中的配置要求，保留一段时间以后删除，一种是保留一段时间后删除，比如说保留两天，另一种是当消息队列超过一定大小时，而不管消息是否被消费过。Kafka通过这种简单的手段来释放磁盘空间，以及减少消息消费之后文件改动的磁盘IO开销。

对于consumer而言，它需要保存的是消息的offset。当消息正常消费时，offset会线性的向前驱动，即消息将依次被消费。事实上consumer可以使用任意顺序消费消息，只需要他将offset设置为任意值。（offset将会被保存在zookeeper中）
![](/images/kafka_zookeeper.png)

kafka集群几乎不需要维护任何producer和consumer状态信息，这些信息由zookeeper保存。因此producer和consumer的客户端实现非常轻量级，他们可以随意的离开而不会对集群造成影响。

partition的设计目的有多个，其中最主要的原因是kafka通过文件存储，通过分区可以将日志文件分散到多个server上，避免文件尺寸达到单机存储的上限，每个partitio都会被当前服务器保存。follower只是复制leader的状态，作为leader的服务器承载了全部的请求压力，因此从集群的整体考虑，有多少个partitions就有多少个leader，kafka可以将leader均衡的分散在不同的节点上分担负载压力。

## producer发布消息
### 写入方式
producer 采用 push 模式将消息发布到 broker，每条消息都被 append 到 patition 中，属于顺序写磁盘（顺序写磁盘效率比随机写内存要高，保障 kafka 吞吐率）。

### 消息路由
producer产生消息发送到broker时，会根据一定的算法选择将其存储到哪个partition。其路由机制为：
```
指定了 patition，则直接使用；
未指定 patition 但指定 key，通过对 key 的 value 进行hash 选出一个 patition
patition 和 key 都未指定，使用轮询选出一个 patition。
```
### 写入流程
producer 写入消息序列图如下所示：
![](/images/kafka_writemessage.png)
写入具体流程说明
```
 producer 先从 zookeeper 的 "/brokers/.../state" 节点找到该 partition 的 leader
 producer 将消息发送给该 leader
 leader 将消息写入本地 log
 followers 从 leader pull 消息，写入本地 log 后 leader 发送 ACK
 leader 收到所有 ISR 中的 replica 的 ACK 后，增加 HW（high watermark，最后 commit 的 offset） 并向 producer 发送 ACK

```

### 消息传输保证（producer delivery guarantee）
一般存在下面三种情况
```
 At most once 消息可能会丢，但绝不会重复传输
 At least one 消息绝不会丢，但可能会重复传输
 Exactly once 每条消息肯定会被传输一次且仅传输一次
```
当 producer 向 broker 发送消息时，一旦这条消息被 commit，由于 replication 的存在，它就不会丢。但是如果 producer 发送数据给 broker 后，遇到网络问题而造成通信中断，那 Producer 就无法判断该条消息是否已经 commit。虽然 Kafka 无法确定网络故障期间发生了什么，但是 producer 可以生成一种类似于主键的东西，发生故障时幂等性的重试多次，这样就做到了 Exactly once，但目前还并未实现。所以目前默认情况下一条消息从 producer 到 broker 是确保了 At least once，可通过设置 producer 异步发送实现At most once。

## broker保存消息

### 存储方式
物理上把topic分成一个或者多个partition，每个partition物理上对应一个文件夹（该文件夹存储该partition的所有索引和消息文件），如下：
![](/images/kafka_topicsave.png)
### 存储策略
无论消息是否被消费，kafka 都会保留所有消息。有两种策略可以删除旧数据：
```
基于时间：log.retention.hours=168
基于大小：log.retention.bytes=1073741824
```
需要注意的是，因为Kafka读取特定消息的时间复杂度为O(1)，即与文件大小无关，所以这里删除过期文件与提高 Kafka 性能无关。

### topic的创建与删除

#### 创建topic
创建topic的序列图如下所示：
![](/images/kafka_createtopic.png)
流程说明：
```
1. controller 在 ZooKeeper 的 /brokers/topics 节点上注册 watcher，当 topic 被创建，则 controller 会通过 watch 得到该 topic 的 partition/replica 分配。
2. controller从 /brokers/ids 读取当前所有可用的 broker 列表，对于 set_p 中的每一个 partition：
	2.1 从分配给该 partition 的所有 replica（称为AR）中任选一个可用的 broker 作为新的 leader，并将AR设置为新的 ISR
	2.2 将新的 leader 和 ISR 写入 /brokers/topics/[topic]/partitions/[partition]/state
3. controller 通过 RPC 向相关的 broker 发送 LeaderAndISRRequest。
```
#### 删除topic
删除topic的序列图如下所示：
![](/images/kafka_deletetopic.png)
```
controller 在 zooKeeper 的 /brokers/topics 节点上注册 watcher，当 topic 被删除，则 controller 会通过 watch 得到该 topic 的 partition/replica 分配。
若 delete.topic.enable=false，结束；否则 controller 注册在 /admin/delete_topics 上的 watch 被 fire，controller 通过回调向对应的 broker 发送 StopReplicaRequest。
```

## 高可用

### replication
一个Topic可以有多个partitions，被分布在kafka集群中的多个server上。kafka可以配置partition需要备份的个数（replicas），每个partition会被备份到多台机器上，以提高可用性。当一台机器宕机时，其他机器上的副本仍然可以提供服务。

每个partition都会有多个备份，其中一个是leader，其他的都是follower。leader执行所有的读写操作，follower只是跟进leader的状态。

kafka分配replica的算法如下
```
将所有的broker(假设总共有n个broker）和待分配的partition排序
将第i个partition分配到第（i mod n）个broker上
将第i个partition的第j个replica分配到第（（i+j）mod n）个broker上
```
### leader failover
当partition对应的leader宕机时，需要从follower中选举出新leader。在选举新leader时，一个基本的原则是，新的leader必须拥有旧的 leader commit 过的所有消息。

kafka在zookeeper中（/brokers/.../state）动态维护了一个ISR（in-sync replicas），由3.3节的写入流程可知 ISR 里面的所有 replica 都跟上了 leader，只有 ISR 里面的成员才能选为 leader。对于 f+1 个 replica，一个 partition 可以在容忍 f 个 replica 失效的情况下保证消息不丢失。

当所有 replica 都不工作时，有两种可行的方案：

1. 等待 ISR 中的任一个 replica 活过来，并选它作为 leader。可保障数据不丢失，但时间可能相对较长。
2. 选择第一个活过来的 replica（不一定是 ISR 成员）作为 leader。无法保障数据不丢失，但相对不可用时间较短。
kafka 0.8.* 使用第二种方式。

kafka 通过 Controller 来选举 leader，流程请参考5.3节。

### broker failover
kafka broker failover 序列图如下所示：
![](/images/kafka_brokerfailer.png)
流程说明： 
```
1. controller 在 zookeeper 的 /brokers/ids/[brokerId] 节点注册 Watcher，当 broker 宕机时 zookeeper 会 fire watch
2. controller 从 /brokers/ids 节点读取可用broker
3. controller决定set_p，该集合包含宕机 broker 上的所有 partition
4. 对 set_p 中的每一个 partition
    4.1 从/brokers/topics/[topic]/partitions/[partition]/state 节点读取 ISR
    4.2 决定新 leader（如4.3节所描述）
    4.3 将新 leader、ISR、controller_epoch 和 leader_epoch 等信息写入 state 节点
5. 通过 RPC 向相关 broker 发送 leaderAndISRRequest 命令
```
### controller failover
当 controller 宕机时会触发 controller failover。每个 broker 都会在 zookeeper 的 "/controller" 节点注册 watcher，当 controller 宕机时 zookeeper 中的临时节点消失，所有存活的 broker 收到 fire 的通知，每个 broker 都尝试创建新的 controller path，只有一个竞选成功并当选为 controller。

当新的 controller 当选时，会触发 KafkaController.onControllerFailover 方法，在该方法中完成如下操作：

```
读取并增加 Controller Epoch。
在 reassignedPartitions Patch(/admin/reassign_partitions) 上注册 watcher。
在 preferredReplicaElection Path(/admin/preferred_replica_election) 上注册 watcher。
通过 partitionStateMachine 在 broker Topics Patch(/brokers/topics) 上注册 watcher。
若 delete.topic.enable=true（默认值是 false），则 partitionStateMachine 在 Delete Topic Patch(/admin/delete_topics) 上注册 watcher。
通过 replicaStateMachine在 Broker Ids Patch(/brokers/ids)上注册Watch。
初始化 ControllerContext 对象，设置当前所有 topic，“活”着的 broker 列表，所有 partition 的 leader 及 ISR等。
启动 replicaStateMachine 和 partitionStateMachine。
将 brokerState 状态设置为 RunningAsController。
将每个 partition 的 Leadership 信息发送给所有“活”着的 broker。
若 auto.leader.rebalance.enable=true（默认值是true），则启动 partition-rebalance 线程。
若 delete.topic.enable=true 且Delete Topic Patch(/admin/delete_topics)中有值，则删除相应的Topic。
```