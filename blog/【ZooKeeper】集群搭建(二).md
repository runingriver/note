---
title: 【ZooKeeper】集群搭建(二)
date: 2017/05/13 11:12:22
toc: false
list_number: false
categories:
- zookeeper
tags:
- zookeeper
---

**前言：**在实际搭建zk集群过程中遇到不少困难,简单配置可以google到,但是想深入一点,对zk的配置做更为深入自定义配置,就有点力不从心了,而且很多的配置都没有做深入的研究.
记录此博客,一方面整理记录一下zk常用操作,将网上人云亦云的杂乱中整理成可用的部分,另一方面,根据zk启动脚本的源码,深入zk的配置,做出合理的,想要的配置!

## 1. 基本命令
- `sudo ./zkServer.sh start`  ， `sudo ./zkServer.sh stop` 停止启动。
kill zk进程：`ps -ef|grep 'zookeeper'|grep -v 'grep'|awk '{print $2}'|xargs -n1 kill -9`
- `sudo ./zkServer.sh status` 查看状态。
- `./zkCli.sh -server 127.0.0.1:8212` 如果修改了`clientPort`，则启动要指定端口。

#### 1. 命令行操作
-  `create /nodeName node “value”` 
- `get /nodeName`
- `set /nodeName newValue`
- `delete /nodeName` 
- `ls /`
- `quit`
- `help`

Tip：节点需要一级一级的创建！

#### 2.  监控命令`The Four Letter Words`
`echo stat | nc l-smsmanage1.wap.cn5 8212`或`echo stat | nc 10.88.106.110 8212`

#### 3. 查看zk事务日志
```
java -cp ./zookeeper-3.4.9.jar:./lib/log4j-1.2.16.jar:./lib/slf4j-log4j12-1.6.1.jar:./lib/slf4j-api-1.6.1.jar org.apache.zookeeper.server.LogFormatter ./logs/version-2/log.100000001
```

## 2. 配置
1. 在zk目录新建`zkdata`和`logs`目录：`mkdir -p /opt/zookeeper/{logs,data}`
2. 复制`conf`目录下的`zoo_sample.cfg` ： `sudo cp zoo_sample.cfg zoo.cfg`
3. 在`zkdata`目录下创建`myid`文件，内容对应于`server.A=B:C:D`
4. 配置`zoo.cfg`
```
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/home/q/zookeeper-3.4.9/zkdata
dataLogDir=/home/q/zookeeper-3.4.9/logs
#client连接zk的端口
clientPort=2181
#最大client连接数
#maxClientCnxns=60
#内存snapshot保存个数
#autopurge.snapRetainCount=3
#垃圾清理频率，单位：小时
#autopurge.purgeInterval=1
#让zk成为observer，无投票权
#peerType=observer
server.1=10.88.65.156:2888:3888
server.2=10.88.106.110:2888:3888
server.3=10.88.141.161:2888:3888
```
**参数说明：**
- `tickTime` 是zookeeper服务器之间或客户端与服务器之间维持心跳的时间间隔,也就是说每个tickTime时间就会发送一个心跳。
- `initLimit` 这个配置项是用来配置zookeeper接受客户端（这里所说的客户端不是用户连接zookeeper服务器的客户端,而是zookeeper服务器集群中连接到leader的follower 服务器）初始化连接时最长能忍受多少个心跳时间间隔（tickTime）数。
当已经超过10个心跳的时间（也就是tickTime）长度后 zookeeper 服务器还没有收到客户端的返回信息,那么表明这个客户端连接失败。总的时间长度就是 10*2000=20秒。
- `syncLimit`这个配置项标识leader与follower之间发送消息,请求和应答时间长度,最长不能超过多少个tickTime的时间长度,总的时间长度就是5*2000=10秒。
- `dataDir` 顾名思义就是zookeeper保存数据的目录,即zk运行的顺序日志(WAL),它是内存数据结构的snapshot，便于快速恢复,默认情况下zookeeper将写数据的日志文件也保存在这个目录里；
- `dataLogDir` 可以不配。一般建议把dataDir和dataLogDir分到不同的磁盘上，这样就可以充分利用磁盘顺序写的特性。同时写日志的时候,避免磁头的多次换道!
- `clientPort`这个端口就是客户端连接Zookeeper服务器的端口,Zookeeper会监听这个端口接受客户端的访问请求；
- `server.A=B:C:D`中的A是一个数字,表示这个是第几号服务器,B是这个服务器的IP地址，C第一个端口用来集群成员的信息交换,表示这个服务器与集群中的leader服务器交换信息的端口，D是在leader挂掉时专门用来进行选举leader所用的端口。
- `maxClientCnxns` 每个客户端(每个client ip)链接的限制
- `minSessionTimeout, maxSessionTimeout` 一般，客户端连接zookeeper的时候，都会设置一个session timeout，
 如果超过这个时间client没有与zookeeper server有联系，则这个session会被设置为过期(如果这个session上有临时节点，则会被全部删除，这就是实现集群感知的基础)。
 但是这个时间不是客户端可以无限制设置的，服务器可以设置这两个参数来限制客户端设置的范围(大于最大值用默认最大值，小于最小值用默认最小值)。
 默认值: `minSessionTimeout` (默认值为：`tickTime * 2`) , `maxSessionTimeout` (默认值为 : `tickTime * 20`)

- `autopurge.snapRetainCount，autopurge.purgeInterval`   客户端在与zookeeper交互过程中会产生非常多的日志，而且zookeeper也会将内存中的数据作为snapshot保存下来，
注：这些数据是不会被自动删除的，这样磁盘中这样的数据就会越来越多。不过可以通过这两个参数来设置，让zookeeper自动删除数据。
其中，`autopurge.purgeInterval`就是设置多少小时清理一次。而`autopurge.snapRetainCount`是设置保留多少个`snapshot`，之前的则删除。

注: 如果zk集群有大量的交互操作，此时执行这个日志删除操作，可能会影响zookeeper集群的性能. 
所以，要将日志清理工作放在集群交互低谷时间段执行，但是zk并没有提供指定时间点执行的设置，所以我们可以禁止这个功能，而改在服务器上配置一个crontab，然后手动清理日志。

## 3. zk监控的四字命令（The Four Letter Words）
1. zk对外提供运行状态监控接口，采用的是`tcp`端口连接。
2. zk同样提供HTTP协议的`The Four Letter Words`，不过只在3.5.0之后的版本支持。
3. 使用：`echo srvr | nc 10.88.65.156 8212`

# 4. 分组Group和Weight理解
1. Zookeeper的Weight配置要和Group配置一起使用，用于确定组的稳定性。
2. Weight等于0的节点不参与投票，没有被选举权，大于0表示该节点在组中的权重，判断一个组是否稳定，要判断存活的节点权重之和是否大于所有节点权重和的1/2.
eg：一个Group三个节点权重：`weight.1=3，weight.2=1，weight.3=1`，如果节点2和3down掉，该节点仍然是稳定的！
# 5. 配置日志滚动输出
1. 新建 `sudo touch zookeeper-env.sh`文件`sudo chmod 755 zookeeper-env.sh`，添加如下内容：
```
#!/usr/bin/env bash
#tip:custom configurationfile，do not amend the zkEnv.sh file
#chang the log dir and output of rolling file
ZOO_LOG_DIR="../logs"
ZOO_LOG4J_PROP="INFO,ROLLINGFILE"
```
2. 修改`conf/log4j.propertes`文件
```
zookeeper.root.logger=INFO, ROLLINGFILE

# Max log file size of 10MB
log4j.appender.ROLLINGFILE.MaxFileSize=10MB
# uncomment the next line to limit number of backup files
log4j.appender.ROLLINGFILE.MaxBackupIndex=10
```
将CONSOLE改为ROLLINGFILE，去掉注释`log4j.appender.ROLLINGFILE.MaxBackupIndex=10`

# 6. 配置zookeeper的jvm大小
1. 新建`sudo touch java.env`，`sudo chmod 755 java.env`
2. 添加如下内容：
```
#!/usr/bin/env bash

#config the jvm parameter in a reasonable,note that shell be source so that do not need to use export
#set classpath,here do not set
#export CLASSPATH=""
#set jvm start parameter , also can set JVMFLAGS variable
SERVER_JVMFLAGS="-Xmx1G -Xms1G"
```
有配置加上`export`，其实没必要，父shell中采用`source`导入！从shell源码中可见配置`JVMFLAGS`和`SERVER_JVMFLAGS`都可以！
#### tip: 请不要按照网上博客中那样直接修改`bin`目录下源文件！

# 7. 配置日志清理
这里只配置系统提供的简单日志清理方法，`crontab`和`zkCleanup.sh`不介绍。
1. 周期清理 `zoo.cfg`中加入
```
#memory	snapshot retain	number
autopurge.snapRetainCount=10
#every two days to purge the log
autopurge.purgeInterval=48
```
策略：保留10个内存的snapshot，每48小时检查清理一次。
2. 手动清理：`zkCleanup.sh -n <count>`  
`-n`知道保留文件的数量。eg：`./zkCleanup.sh 5` 保留5个日志文件
3. 定时任务清理
`0 2 * * * /bin/bash /home/q/zookeeper/bin/zkCleanup.sh /home/q/zookeeper/data -n 10 >/dev/null`

# 8. 本地集群搭建
首先，如果测试环境，直接用一个zk即可。简单说下我测试环境zk集群的搭建：
1. 解压`zookeeper-3.4.x.jar`，分别命名`node1，node2，node3`
2. 配置zoo.cfg文件（这里以node3为例）
    ```
    tickTime=2000
    initLimit=10
    syncLimit=5
    dataDir=/home/q/zookeeper/node3/zkdata
    dataLogDir=/home/q/zookeeper/node3/logs
    clientPort=8214
    #memory	snapshot retain	number
    autopurge.snapRetainCount=5
    #every two days to purge the log
    autopurge.purgeInterval=48
    server.1=127.0.0.1:8222:8333
    server.2=127.0.0.1:8223:8334
    server.3=127.0.0.1:8224:8335
    ```
注意几点不同：
 - `clientPort=8214`每个节点的连接端口应该不一样，`node1，node2，node3`分别对应：`8212，8213，8214`
 - zk间的通信和选举接口应该不一样！`8222:8333，8223:8334，8224:8335`
 - host配置成：`127.0.0.1`

3. 在node3目录下新建`logs`和`zkdata`文件夹，并在`zkdata`文件下新建`myid`文件，根据node顺序填写`1,2,3`
完成！
Tip：如果启动不成功，第一排查`dataDir`和`dataLogDir`路径，然后检查端口占用！