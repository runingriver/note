---
title: 【ZooKeeper】集群配置（三）
date: 2017/05/14 11:12:22
toc: false
list_number: false
categories:
- zookeeper
tags:
- zookeeper
---

# 配置 
本节是在`【ZooKeeper】集群搭建（二）`的基础之上，记录zk集群搭建的常用配置。配置具体意义和更详细内容请看，配置与Google的结果有部分出入，可以根据需求进行配置。 
Tip：所有配置文件都在`./conf`目录下! 该节配置与一些博客写法有部分出入,通过阅读启动脚本,根据脚本里面的逻辑,以自认为较合理的方式,设置jvm参数,日志输出方式!

## 1. 基本配置
修改`zoo.cfg`文件
```
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/home/q/zookeeper-node2/zkdata
dataLogDir=/home/q/zookeeper-node2/logs
clientPort=8212
#memory	snapshot retain	number
autopurge.snapRetainCount=5
#every two days to purge the log
autopurge.purgeInterval=48
server.1=10.xx.65.156:8222:8333
server.2=10.xx.106.110:8222:8333
server.3=10.xx.141.161:8222:8333
```
这里给出另一种配置, 方便机器迁移的搭建方式:
```
# /etc/hosts中指定
192.168.1.20 zk1
192.168.1.21 zk2
192.168.1.22 zk3
# zoo.cfg 中配置对应的名称, 这样如果机器迁移,就只需改每台机器的host即可.
server.1=zk1:2081:3801
server.2=zk2:2801:3801
server.3=zk3:2801:3801
```

## 2. 配置运行zk的jvm
`./conf`目录下新建`java.env`文件,修改到`sudo chmod 755 java.env`权限,主要用于`GC log`,`RAM`等的配置.
```
#!/usr/bin/env bash
#config the jvm parameter in a reasonable
#note that the shell is source in so that do not need to use export 
#set java  classpath 
#CLASSPATH="" 
#set jvm start parameter , also can set JVMFLAGS variable
SERVER_JVMFLAGS="-Xmx1024m"
```



## 3. 配置zk日志的滚动输入
默认zk日志输出到一个文件,且不会自动清理,所以,一段时间后zk日志会非常大!
这里配置zk日志滚动输出,且每个文件10M限制,最多保留10个文件.

1. `zookeeper-env.sh`
`./conf`目录下新建`zookeeper-env.sh`文件,修改到`sudo chmod 755 zookeeper-env.sh`权限
```
#!/usr/bin/env bash
#tip:custom configurationfile，do not amend the zkEnv.sh file
#chang the log dir and output of rolling file
ZOO_LOG_DIR="../logs"
ZOO_LOG4J_PROP="INFO,ROLLINGFILE"
```

2. log4j.properties 修改日志的输入形式
```
# Define some default values that can be overridden by system properties
zookeeper.root.logger=INFO, ROLLINGFILE
# ......
# Max log file size of 10MB
log4j.appender.ROLLINGFILE.MaxFileSize=10MB
# uncomment the next line to limit number of backup files
log4j.appender.ROLLINGFILE.MaxBackupIndex=10
# ........
```

## 4. 使用zooInspector

1. `github clone zooInspector` 地址（https://github.com/zzhang5/zooinspector）也可以用官方的地址。
2. `cd zooinspector/`  `mvn clean package` 编译
3. 修改`target/bin`目录下的`.sh`权限。运行。集群多台机器，任意一台即可监控zk节点。
本机：`/home/xxx/github/zooinspector/target/zooinspector-pkg/bin/zooinspector.sh`
