---
title: 【ActiveMQ】集群搭建(二)
date: 2017/06/13 11:12:22
toc: false
list_number: false
categories:
- ActiveMQ
tags:
- ActiveMQ
---

前言：使用ActiveMq前一般都要读一遍它的官方文档，但是ActiveMq的文档写的相当凌乱（个人观点），然后在搭建ActiveMq高可用集群过程遇到很多疑惑和坑，这里记录下来，以作前车之鉴吧。

# 基于Master-Slaver+LevelDB的HA集群搭建
下面搭建一个Master-Slaver-Slaver的集群，采用`activemq 5.14.4`版本，每个节点目录采用`activemq-node-%d`方式命名。

## 1. 配置zk
进入：`conf`目录，编辑`activemq.xml`文件，在`broker`节点下，删除默认的持久化方式：
```
<persistenceAdapter>
    <kahaDB directory="${activemq.data}/kahadb"/>
</persistenceAdapter>
```
然后，改为下面的配置：
```
<replicatedLevelDB
        directory="${activemq.data}/leveldb"
        replicas="3"
        bind="tcp://0.0.0.0:56262"
        zkAddress="127.0.0.1:8212,127.0.0.1:8213,127.0.0.1:8214"
        zkPath="/activemq/leveldb-stores"
        zkSessionTimeout="30s"
        weight="20"
/>
</persistenceAdapter>
```
其中，`directory="${activemq.data}/leveldb"`表示持久化目录位置，`replicas="3"`表示有三个节点， `bind="tcp://0.0.0.0:62626"`设置LevelDB主从复制端口，`zkAddress="127.0.0.1:8212,127.0.0.1:8213,127.0.0.1:8214"`表示zk集群地址，`zkPath="/activemq/leveldb-stores"`设置activemq在zk集群中的节点路径，`zkSessionTimeout="30s"`表示activemq与zk建立连接后session超时时间，**强烈建议设大点，因为如果太短当各种原因（比如节点load高或zk回复较慢）下，可能会导致session超时从而触发failover！**，`weight="20"`表示每个节点的权重，failover时安装这个权重进行选举Master，可以把机器性能较好的设置一个较大权重。



## 2. 配置权限

