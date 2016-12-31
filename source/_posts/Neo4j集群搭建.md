---
title: Neo4j集群搭建
date: 2016-12-09 23:36:19
tags: 
	- Neo4j
categories: Neo4j
---

## Neo4j集群的功能

Neo4j可以在集群模式下配置，以适应不同的负载，容错和可用硬件的要求。Neo4j集群由一个Matser实例和零个或多个Slave实例组成。 集群中的所有实例都在其本地数据库文件中具有数据的完整副本。

一个具有三个实例的Neo4j集群如图所示：

![](/images/cluster.PNG)

上图中的绿色箭头代表的是，每个实例包含所需的逻辑，以便与集群中的其他成员协调以进行数据复制和选举管理。上图中的蓝色箭头代表的是，Slave实例（不包括atbiter实例）和Master实例之间的相互通信，以确保每个每个数据库中的数据都是最新的。

## Neo4j集群的具体搭建过程

### 安装环境
* 虚拟机系统：UbuntuServer 16.04
* 虚拟机配置：内存2G， CPU2P
* Jdk: 1.8
* Neo4j: 3.0.7企业版

### 安装步骤
**添加IP和主机名称**

在每个虚拟机上打开hosts文件
``` shell
vi /etc/hosts

```
分别在每个虚拟机的配置文件中添加三个虚拟机的IP和名称
``` shell
10.108.217.243 Neo4j-01
10.108.219.44  Neo4j-02
10.108.217.230 Neo4j-03
```
**关闭防火墙(没有的话则不必管)**
``` shell
iptables off 
```

**为每个虚拟机安装Neo4j**

我使用root用户安装的，如果用其他用户安装也可以。

创建Neo4j的目录
```  shell
mkdir /usr/neo4j
```
将下载的neo4j-enterprise-3.0.7-unix.tar.gz文件放到刚才的目录中


解压安装包
``` shell
tar -xf /usr/neo4j/neo4j-enterprise-3.0.7-unix.tar.gz
rm /usr/neo4j/neo4j-enterprise-3.0.7-unix.tar.gz
```
用完解压命令之后解压出的文件夹用户组是nogroup，使用一下命令更改所有者和用户组
``` shell
chown -R root:root /usr/neo4j/neo4j-enterprise3.0.7
```

### 配置

修改每个虚拟机上的neo4j配置文件
``` shell
vi /usr/neo4j/neo4j-enterprise-3.0.7/conf/neo4j.conf

```
![](/images/conf.png)

为个虚拟机的Neo4j配置文件修改图中三个部分：

``` shell
Neo4j-01：

ha.mode=HA
ha.server_id=1
ha.initial_hosts=IP1:5001,IP2:5001,IP3:5001

Neo4j-02：

ha.mode=HA
ha.server_id=2
ha.initial_hosts=IP1:5001,IP2:5001,IP3:5001

Neo4j-03：

ha.mode=HA
ha.server_id=3
ha.initial_hosts=IP1:5001,IP2:5001,IP3:5001

```

### 每个虚拟机启动Neo4j

``` shell
/usr/neo4j/neo4j-enterprise-3.0.7/bin/neo4j start
```

至此一个具有一个主机两个从机的Neo4j集群就搭建完成了。

## Neo4j集群详细配置

* ***dbms.ha:*** 数据库的运行模式（**HA/ARBITER**）；
* ***ha.server_id:*** 集群中每个实例的唯一标识。必须是一个唯一的正整数；
* ***ha.host.coordination:*** 集群中的每个实例用来监听集群通信的端口，默认5001；
* ***ha.initial_hosts:*** 当每个实例启动时，帮助他们寻找和加入集群的端口。当集群刚启动时，在所有实例都加入到集群并且可以互相通信之前，这段时间数据库不允许被访问；
* ***ha.host.data:*** 用来监听集群中从Master提交过来的事物操作，默认为6001。不能与***ha.host.coordination***相同；
* ***仲裁实例:*** 仲裁实例不同于Master实例和Slave实例，它工作在仲裁模式，虽然参与集群的通信，但是不复制集群中的数据存储，主要作用是打破Matser选举过程的死结，开启仲裁实例***dbms.mode=ARBITER***

## Master选举

### 选举规则

1. 开启集群时第一个启动的实例就是Master实例。
2. 如果Master实例因为某种原因宕机，或者是集群冷启动，则Slave实例中提交事务最多的一个将会被选举为Master。
3. 如果有两个Slave实例提交的事务一样多，那么host_server_id靠前的一个将赢得选举。

### 查看当前实例状态

输入以下命令，如果返回true则是Master实例
``` shell
curl http://<your ip>:7474/db/manage/server/ha/master
```

输入以下命令，如果返回true则是Slave实例
``` shell
curl http://<your ip>:7474/db/manage/server/ha/slave
```

输入以下命令，如果返回master则是Master实例，返回slave则是Slave实例
``` shell
curl http://<your ip>:7474/db/manage/server/ha/available
```
详细情况如下图所示：
![](/images/cluster_status.png)

本文结束。


