前言：

## zk+levelDB的集群搭建

## 权限控制

## 配置


<simpleAuthenticationPlugin anonymousAccessAllowed="true">
 <users>
 <authenticationUser username="admin" password="admin" groups="users,admins" />
 <authenticationUser username="user" password="user" groups="users" />
 <authenticationUser username="guest" password="guest" groups="guests" />
 </users>
 </simpleAuthenticationPlugin>

 <authorizationPlugin>
 <map>
 <authorizationMap>
 <authorizationEntries>
 <authorizationEntry queue=">" write="producers" read="consumers" admin="admins" />
 </authorizationEntries>
 </authorizationMap>
 </map>
 </authorizationPlugin>


http://activemq.apache.org/slow-consumer-handling.html
http://activemq.apache.org/security.html

## 一. 启动前配置

1. 配置jvm大小: 修改`./bin/env`，设置内存大小。

2. 配置磁盘使用： `./conf/activemq.xml` ，设置`systemUsage`

3. 配置日志格式: 如将日志(`conf/log4j.properties`)大小由1024KB改为10240KB









### 一. activemq.xml 配置



### 二. 日志配置



### 三. 权限配置

`jmx.password`和`jmx.access`是配置jmx管理接口的权限，这里我们一般不开启jmx！

`credentials.properties` 是配置Client连接Broker的权限控制。

```

activemq.username=admin

activemq.password=xxx

```

`jetty-realm.properties`是登录ActiveMQ管理界面的用户名密码：

`admin: xxx, admin` 注：前面是密码，后面是用户名！






# hawtio集成使用
针对ActiveMQ 集成hawtio监控,大多说的不够详细,而且会遇到很多问题!

## 下载





## 解压

`sudo unzip hawtio.war -d ./hawtio` 把`.war`包解压到当前目录的`hawtio/`目录下

或者 `cd /hawtio; jar -xvf hawtio.war` 解压 `jar`命令会解压到当前目录!



## 配置

1. 修改`webapps/hawtio/WEB-INF/web.xml` 中属性`<env-entry-name>hawtio/authenticationEnabled</env-entry-name>` 为 `true`.

2. 修改登录密码

 修改 文件 `./conf/users.properties` 密码格式: `admin=your-password` ,其中,默认是`admin=admin`

 **注: 与`activemq`自带的console不同的是`activemq`登录密码是在`jetty-ream.properties`文件中修改!**















## 2. 基本使用

1. 下载activemq(http://activemq.apache.org/download.html), 官方入门(http://activemq.apache.org/getting-started.html)

2. 解压, 进入`/bin`目录, 启动: `sudo ./activemq start`后台执行，  `sudo ./activemq console`前台执行(日志输出到控制台)。

    注: 如果命令行窗口关闭mq进程会退出.

    - `sudo nohup ./activemq start` 日志默认放在`data/activemq.log`

    - `sudo nohup ./activemq start /home/q/log/mqlog` 指定日志输出路径.

3. 监控界面: (http://127.0.0.1:8161/admin/), 用户名密码默认都是:`admin`

4. 关闭 `sudo ./activemq stop`

5. 诊断系统各项参数`sudo ./activemq-diag`

### 其他命令

1. 查看mq是否启动`netstat -an | grep 61616`



### 配置链接

1. consumer配置 (http://activemq.apache.org/destination-options.html)



## 二. 特殊情况

### 1. master宕机切回master策略:

方法一: 

1. 首先shutdown slave broker.

2. 将slave broker的data目录拷贝到master的data目录.

3. 启动master,再启动slave.

Tip：该方法是官方文档提供的解决方案，该方案优点在于，当pending在broker的数据量很大的情况下，集群能正确恢复。

方法二: 

1. 如有Master A，Slaver B，Slaver C三台机器，分配的权重分别是30,20,10，机器A的性能最好，故一般用来做Master，特殊原因A down掉。

2. 首先，重启master A。此时master A会变成slaver A，Slaver B会被选举成Master B

2. 停掉B （现在是master B），重新选举，变成slave的master A会被选举为master，通过配置的weight选项确定.

3. 启动B B变成slaver。

Tip：该方案优点在于，当pending在broker的数据较小时，且有producer在往broker push数据，集群能够接收producer的数据，并自恢复！



### 2. 基于LevelDB的HA架构下，ZooKeeper的master挂断，MQ集群会重新选举，可能会导致MQ集群不能正常服务

首先，当Broker中pending的消息较少时，模拟ZK master down掉，MQ集群进行选举，能够自恢复！

当，Broker中pending的消息较多时，模拟ZK master down掉，MQ集群进行选举，不能自恢复，会不停的转移Master，重复选举。

不能自恢复的原因主要由于Producer在不停地往集群push消息，或者不可控制因素，导致LevelDB中的数据不能正确被处理！

解决：

1. 不能自恢复有三个可能的主要因素：pending broker的数据太大，producer不见得往集群push小，其他未知原因！

2. 当出现未知情况MQ集群不能提供服务，第一时间Stop所有节点，然后逐个start，不能逐个直接restart！

3. 当节点数据不一致时，选择一个节点数据作为恢复数据，将其他节点数据删除，并将恢复数据复制到其他节点，逐个start！

4. 期间，如果producer没做特殊情况处理，可能producer push的数据会丢失！

Tip：ZK的Slaver和MQ的Slaver其中任何一台down掉都不会影响整个MQ集群的服务！


Tip：mq现在架构2台是可以运行的，但是2者不能down任何其一，3台可以保证down了一台，系统依旧可以运行！



踩坑：

1. MQ broker谨慎操作，一次打印jstack，导致OOM，服务down了。
如果MQ出故障不能提供服务，最快的方式，是先stop三台服务，备份一台机器上面的data文件夹（防止重启失败，数据混乱），然后逐一重启！
2. zk挂了。mq不能自恢复，需要重启，如果是restart多个mq之间可能会不停地切换master！
