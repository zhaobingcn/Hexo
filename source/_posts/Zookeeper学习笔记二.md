---
title: Zookeeper学习笔记二
date: 2016-07-27 14:36:07
tags: Zookeeper
categories: Zookeeper
---
Zookeeper官网下载下载最新的安装包，目前最新的稳定版是Release 3.4.9(stable)，本课题所有示例都将基于这个版本，整体环境如下：

* 电脑环境：Ubuntu 16.04
* JDK：1.8
* Zookeeper：Release 3.4.9(stable)

## 安装Zookeeper

## 1、创建目录，安装zookeeper
手上只有一台测试机器，所以采用伪集群的方式运行zookeeper，下面创建三个文件夹server1、server2、server3分别用于安装zookeeper。
```
cd /opt
sudo mkdir zookeeper-3.4.9
sudo chmod -R 777 zookeeper-3.4.9
cd zookeeper-3.4.9
cp ~/Downloads/zookeeper-3.4.9.tar.gz ./
mkdir server1 && tar -zxvf zookeeper-3.4.9.tar.gz -C ./server1 --strip-components 1
mkdir server2 && tar -zxvf zookeeper-3.4.9.tar.gz -C ./server2 --strip-components 1
mkdir server3 && tar -zxvf zookeeper-3.4.9.tar.gz -C ./server3 --strip-components 1
rm zookeeper-3.4.9.tar.gz
```

## 2、修改配置文件
```
#server1
mkdir ./server1/data
cp server1/conf/zoo_sample.cfg server1/conf/zoo.cfg
vim server1/conf/zoo.cfg
    #需要修改的属性如下
    dataDir=/opt/zookeeper-3.4.9/server1/data
    clientPort=2181
    server.1=localhost:2888:3888
    server.2=localhost:2889:3889
    server.3=localhost:2890:3890
#创建与server对应的myid
echo "1" > server1/data/myid
#server2
mkdir server2/data
cp server2/conf/zoo_sample.cfg server2/conf/zoo.cfg
vim server2/conf/zoo.cfg
    #需要修改的属性如下
    dataDir=/opt/zookeeper-3.4.9/server2/data
    clientPort=2182
    server.1=localhost:2888:3888
    server.2=localhost:2889:3889
    server.3=localhost:2890:3890
#创建与server对应的myid
echo "2" > server2/data/myid
#server3
mkdir server3/data
cp server3/conf/zoo_sample.cfg server3/conf/zoo.cfg
vim server3/conf/zoo.cfg
    #需要修改的属性如下
    dataDir=/opt/zookeeper-3.4.9/server3/data
    clientPort=2183
    server.1=localhost:2888:3888
    server.2=localhost:2889:3889
    server.3=localhost:2890:3890
#创建与server对应的myid
echo "3" > server3/data/myid
```

## 3、启动服务测试
下图显示信息表示服务启动正常：

![](/images/zookeeper_2.png)

## 客户端脚本

客户端脚本的测试基于上面安装的Zookeeper服务，之后的所有测试示例同样基于上述服务。

## 创建节点

使用create命令可以创建一个节点：create /first-node zhaobing，表示在Zookeeper的根节点下创建了一个/first-node的节点，并且节点的数据内容是”bboyjing”。

## 读取节点

使用ls命令，可以列出指定节点下的所有子节点：ls /，表示查看根节点下的所有子节点。
输出：[first-node, zookeeper]，first-node是之前创建的节点，zookeeper节点是自带的保留节点。
使用get命令，可以获取指定节点的数据内容和属性信息：get /first-node

## 更新节点

使用set命令，可以更新指定节点的数据内容：set /first-node hello-bboyjing

## 删除节点

使用delete命令，可以删除指定节点：delete /first-node，要注意的是指定节点没有子节点才可以被删除

## 注意

上述的命令只是简单的演示，每个命令还会有其他参数，比如创建节点时可以选择永久节点还是临时节点，更新的时候可以基于某个version之类的，这里就不展开了。