---
layout:     post
title:      HDFS简介 
discription: 本文通过一个具体的例子来解释什么是HDFS。
date:       2018-04-23 10:27:00
categories: Hadoop
tags:       Hadoop HDFS
---

* 文章目录
{:toc}

> HDFS(Hadoop Distributed Filesystem)，又称DFS，是Hadoop平台下的分布式文件存储系统

## HDFS架构
![HDFS架构](http://oc26wuqdw.bkt.clouddn.com/blog/2018/5/hdfs/hdfsarchitecture.png)




## HDFS概念
### 数据块
每个磁盘都有默认的数据块大小，数据块是磁盘进行数据读写的最小单位。HDFS同样有块（block）的概念，默认是64MB。HDFS上的文件存储同样被划分为块大小的多个分块（chunk）作为独立的存储单元。使用块存储还非常适合于数据备份而提高数据可用性和提供数据容错能力。
HDFS数据块默认64MB是为了均衡数据块传输时间和磁盘寻址时间的比例和MapReduce任务中的任务数量。块过小，寻址次数就多，寻址时间开销就大了；块过大，单一任务拆分的任务数就少了，体现不出分布式处理的优势

### namenode和datanode
> HDFS集群有两类节点以管理者（namenode）-工作者（datanode）模式运行。

namenode管理文件系统命名空间，维护着文件系统树及整棵树内所有的文件和目录（命名空间镜像文件和编辑日志文件），并记录着每个文件中各个块所在的数据节点信息（非永久保存，系统启动时重建）。
datanode是工作节点，根据需要存储并检索数据块（受客户端或namenode调度），并定期向namenode发送存储的块列表。
所以namenode的容错非常重要，hadoop提供两种机制

- 通过网络文件系统备份元数据
- 辅助namenode作为镜像

### 联邦HDFS
namenode在内存中保存文件系统中每个文件和每个数据块的引用关系，这意味着对于一个拥有大量文件的超大集群来说，内存将成为横向扩展的瓶颈。Hadoop在2.X中引入联邦HDFS允许添加namenode实现扩展，其中每个namenode管理文件系统命名空间中的一部分（如/user目录下的所有文件）

### HDFS的高可用
即使通过数据备份防止数据丢失，依然无法实现文件系统的高可用，Hadoop在2.x中对HDFS增加了高可用（HA）的支持。通过配置活动-备用（active-standby）namenode，当活动namenode失效时，备用namenode就开始接管客户端的请求，这一实现需要在架构上做如下修改

- namenode之间通过高可用的共享存储实现编辑日志的共享
- datanode需要同时向两个namenode发送数据块处理报告（因为数据块映射信息存储在namenode的内存中）
- 客户端需要使用特定的机制来处理namenode时效的问题

> 一个称为故障转移控制器（failover_controller）的系统管理着将活动namenode转移为备用namenode的过程
  
## 数据流
### HDFS读流程
![HDFS读流程](http://oc26wuqdw.bkt.clouddn.com/blog/2018/5/hdfs/read.png)

### HDFS写流程
![HDFS写流程](http://oc26wuqdw.bkt.clouddn.com/blog/2018/5/hdfs/write.png)

## 导入数据到HDFS
### Flume
Apache Flume是一个将大规模流数据导入HDFS的工具，通过管道的方式将本地文件写入Flume中，Flume节点允许以任何拓扑方式进行组织，典型配置是在每台源机器上运行一个Flume节点，通过多层级的聚合，最后将数据存入HDFS

### Sqoop
Apache Sqoop是为了将数据从结构化存储设备批量导入HDFS设计的，例如关系型数据库