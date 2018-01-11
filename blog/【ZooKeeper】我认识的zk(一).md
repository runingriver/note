---
title: 【ZooKeeper】我认识的zk(一)
date: 2017/05/12 11:12:22
toc: false
list_number: false
categories:
- zookeeper
tags:
- zookeeper
---

## 一. zk使用场景
1. 服务发现： 对集群中节点机器的变化，做出对应的对策！如，dubbo中消费者访问某个服务可以通过zk知道哪些机器在提供服务，从而实现负载均衡策略。
2. 配置管理：集中管理配置文件，配置实时更新。
3. 分布式锁：集群中存在独占资源时，利用zk的顺序节点或节点version特性实现公平锁和乐观锁。
4. leader选举：master-slave架构中常用zk来实现故障转移，master选举。
## 二. zk基本知识
1. zk集群里分三种角色: Leader, Follower和Observer。Leader和Follower参与投票，Observer只监听投票结果，不参与投票。
2. 投票集群里的节点数要求是奇数。
3. 一个集群容忍挂掉的节点数的等式为 N = 2F + 1，N为投票集群节点数，F为能同时容忍失败节点数。比如一个三节点集群，可以挂掉一个节点，5节点集群可以挂掉两个。
4. 一个写操作需要半数以上的节点ack，所以集群节点数越多，整个集群可以抗挂点的节点数越多(越可靠)，但是吞吐量越差。
5. zk里所有节点以及节点的数据都会放在内存里，形成一棵树的数据结构。并且定时的dump snapshot到磁盘。
6. Client（应用程序）与zk servers之间维持的是长连接，并且保持心跳。
Session超时时间：首先zk Servers里配置了Session超时的最小值和最大值，如果client的值在这两个值之间则采用client的，否则采用zk servers的最值。
zk默认最大最小值: `minSessionTimeout` (默认值为：`tickTime * 2`) , `maxSessionTimeout` (默认值为 : `tickTime * 20`)
7. Client（应用程序）可以watch zk那个树形数据结构里的某个节点或数据，当有变化的时候会得到通知。
8. zk会产生三种日志：1.txlog：每个写操作，包括新Session都会记录一条log；2. 内存数据dump的Snapshot；3. 运行的应用产生的日志（`zookeeper.out`）。
9. 3.5.0之前的版本都是采用静态配置，3.5.0之后`group，port，server`都可以动态配置了。
## 三. zk集群搭建
与zk交互较少的集群搭建很简单，这里记录复杂zk集群搭建：

1. 分Group，首先 要确保zk整个集群可靠运行，就是要确保投票zk集群可靠。所以一般将Leader+Follower划为核心Group，核心Group不向外提供服务。再根据不同业务划分Group，如，将服务发现，消息，锁三个业务划分为三个Observe Group。
2. Client（应用程序）只会连接分配给它的Observer Group，不去连接核心Group。这样核心Group就不会给Client提供长连接服务，也不负责长连接的心跳，这大大的减轻了核心Group的压力。
Tip: 因为在实际环境中，一个Zookeeper集群要为上万台机器提供服务，维持长连接和心跳还是要消耗一定的资源的。
Tip: 因为Observer是不参与投票的所以加Observer并不会降低整体的吞吐量，而且Observer挂掉不会影响整个集群的健康。

**分Group的不足：** 所有的写入还是要交给核心Group来处理的，在写入量大的情况下，系统吞吐率会急速下降，这时可以采用集群隔离！
所谓集群隔离就是重新搭建一套zk集群，如将服务发现，消息，锁三个业务划分为三个三个集群！

## zk集群要注意的问题
1. zk所有数据都放在内存中，所以要规划好JVM内存大小，防止Swap！可以在`zookeeper-env.sh`中个性化设置
2. zk需要频繁写txlog并定期将内存dump到磁盘作为一个快照（snapshot），所以要注意定期清理磁盘日志。另外zk默认不会分割日志，而将所有日志保存在`zookeeper.out`日志文件中。
可以在`zookeeper-env.sh`脚本中设置日志轮转。
3. 在服务依赖多的情况下要做好监控。写一个监控程序，定时去操作zk集群中每个节点，记录状态；监控树结构中的节点数和监控节点数据的Watcher数；还可以监控连接zk集群的Client，并监控它们的流量。
4. 尽可能的不要强依赖与zk集群，比如缓存配置文件，缓存应用程序集群节点信息，当zk出现问题时，做服务降级。
5. 不要用zk作数据存储，不做交互频繁的锁操作（细粒度锁）
6. 临时节点不能不能有子节点.

## 四. zk权限控制(acl(访问控制),quota(几点数据控制))
zk中每个节点都可以设置访问权限,并设置节点的数据权限(设置子节点数量,节点数据大小)
### Acl
- `getAcl path`来查看节点的访问权限
#### ZooKeeper提供了如下几种验证模式(scheme)：
1. `digest`：Client端由用户名和密码验证，譬如`user:password`，digest的密码生成方式是Sha1摘要的base64形式
    - `java -cp ./zookeeper-3.4.9.jar:./lib/log4j-1.2.16.jar:./lib/slf4j-log4j12-1.6.1.jar:./lib/slf4j-api-1.6.1.jar org.apache.zookeeper.server.auth.DigestAuthenticationProvider test:test`  获取加密后的密码
    - `setAcl /test1 digest:test:V28q/NynI4JI3Rk54h0r8O5kMug=:crwda`  设置节点`/test1`为digest访问权限
    - `addauth digest test:test`  认证digest权限,这样就可以访问`/test1`节点了.

2. `auth`：不使用任何id，代表任何已确认用户。
3. `ip`：Client端由IP地址验证，如 : `setAcl /test ip:10.194.157.58:crwda`
4. `world`：固定用户为anyone，为所有Client端开放权限
5. `super`：在这种scheme情况下，对应的id拥有超级权限，可以做任何事情(cdrwa）
注:`exists`操作和`getAcl`操作并不受ACL许可控制，因此任何客户端可以查询节点的状态和节点的ACL。

#### 节点的权限（perms）主要有以下几种(`cdrwa`)：
`Create` 允许对子节点Create操作
`Read` 允许对本节点GetChildren和GetData操作
`Write` 允许对本节点SetData操作
`Delete` 允许对子节点Delete操作
`Admin` 允许对本节点setAcl操作
注: 节点的权限不具有继承性,各个节点的acl是独立的.
### quota
设置每个节点的数据大小和子节点个数
- 格式: `setquota -n|-b val path` 其中`-n` 表示子节点个数,`-b`节点数据大小.