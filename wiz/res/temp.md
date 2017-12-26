前言：这不是关于ActiveMq的一个全面介绍，这里整体记录使用ActiveMq过程中的一些知识点，总结起来。

### 1. 基本使用知识点

### 默认端口
管理页面端口：8161
LevelDB主从复制端口：62626
client连接broker端口：`openwire：61616`；`amqp：5672`；`stomp：61613`；`mqtt：1883`；`ws：61614`
zk的sever与client：2181; master选举端口:3888; master-slave通信:2888;

### activemq相关jar包介绍
1. `activemq-all`所有activemq相关jar的集合,不建议在工程中直接引入, 处出现很多冲突!
2. `activemq-core`是`activemq-broker`和`activemq-client`的结合. 等于将两个包柔和成一个包.
3. `activemq-broker`是中间代理的实现,即部署在服务器的逻辑实现. 当需要管理某个broker时(比如:BrokerService生命周期),可以引入此jar包.
4. `activemq-client`是客户端的实现,实现工程项目和服务器部署的broker的通信, 通常我们工程中需要引入的jar包.
5. `activemq-pool`客户端和服务器之间的连接是一个耗时的过程,我们可以把客户端和服务器的连接管理交给pool,每次从pool中获取连接.可以说是对`activemq-jms-pool`的一个封装.
6. `activemq-jms-pool`具体的连接池的实现,也是依赖`commons-pool`的实现.

tip: 一般我们项目只要引入`activemq-client`即可, 若消息比较多需要连接池可引入`activemq-pool`,如果想结合spring等,还需要`spring-jms`等.
注: 很多说工程项目中需要引入`activemq-core`和`activemq-pool`, 其实是没有问题的,看源码对比`activemq-client`实现,基本都是一样的实现. 

### conf目录中文件的作用
1. `activemq.xml` broker的最主要的配置文件！配置权限，协议，队列，broker架构等等。
2. `credentials.properties`配置，可简单理解为**client连接broker队列时的权限**，配合`simpleAuthenticationPlugin`使用，`simpleAuthenticationPlugin`无法将权限控制到具体的queue或topic，`credentials-enc.properties`作用同`credentials.properties`，但是密码是加密后的密码，具体参见：`http://activemq.apache.org/encrypted-passwords.html`
3. `groups.properties`和`users.properties`定义**topic和queue的权限管理**，是JAAS鉴权方法，可以精确到某个destination，`groups.properties`里面定义组，`users.properties`定义用户名密码。与`login.config`配合使用！
4. `login.config`配置JAAS，参见：`http://activemq.apache.org/security.html`
5. `jetty-realm.properties`用于设置86161端口的web管理界面的登录权限。
6. `jmx.access`和`jmx.password`是启用了jmx后的权限管理。
7. `log4j.properties`和`logging.properties`日志配置，`logging.properties`少用，`log4j.properties`用于配置broker的运行日志，和audit的登录日志。

### bin目录中文件的作用
1. `activemq` broker启动停止，统计等一系列操作。`sudo ./activemq --help`查看使用方法。
2. `activemq-diag`系统诊断`sudo ./activemq-diag --help`查看使用方法！
3. `env`启动配置参数，使用前推荐阅读一遍`env`脚本！

- - -

## 2. 开发和配置MQ相关知识点
1. `SimpleMessageListenerContainer` 默认情况下批量拿到了1000条消息，但是会平均分配给各个Session。
2. `DefaultMessageListenerContainer` 则支持更多的特性，例如事务。
3. `ActiveMQ Jolokia REST API` 用于通过http来操作队列！
比如：`curl -u admin:admin -d "body=message" http://localhost:8161/api/message/TEST?type=queue` 给队列发送一个消息。
> JMX是一个J2EE监控/管理协议，采用MBean进行网络传输，但是其他非java平台无法理解，Jolokia则是解决这个问题的一个方法，它可以把JMX的MBean Restful JSON 化。
4. 连接broker的URI中不允许有空格.
5. `client`的配置可以通过`链接URL`和`ActiveMQConnectionFactory`对象进行配置。（http://activemq.apache.org/connection-configuration-uri.html）
如配置消息的异步发送：
`cf = new ActiveMQConnectionFactory("tcp://locahost:61616?jms.useAsyncSend=true");`
`((ActiveMQConnectionFactory)connectionFactory).setUseAsyncSend(true);`
二者等价。
6. 默认情况下AMQ采用异步发送模式, 如果是`非事务的持久化消息`则默认采用的是同步发送(不允许消息丢失), 如果能够容忍少量消息丢失,则可以在这种模式下启动异步发送!
7. URL中的`randomize=false`表示master/slave模式下, clients的连接不随机连接到slave上! 也即,slave仅仅只同步master消息!
8. `connectionFactory.setOptimizeAcknowledge(true)` 设置批量确认, 以提高效率和吞吐量!
9. broker添加gc log日志. 
文件`bin/env` 
字段 `ACTIVEMQ_OPTS_MEMORY ` 
内容 `-Xms4G -Xmx8G -verbose:gc -XX:+PrintGCDateStamps -XX:+PrintGCDetails -Xloggc:$ACTIVEMQ_DATA/gc.log`
10. queue和Topic消息的顺序,只能在activemq.xml中设置`strictOrderDispatch`.
Queue: 
`<policyEntry queue=">" strictOrderDispatch="false" />`
默认不设置的情况下,单消费者时,消息按照顺序发送, 当Topic: 
```
<destinationPolicy>
  <policyMap>
    <policyEntries>
      <policyEntry topic=">">
        <dispatchPolicy>
          <strictOrderDispatchPolicy/>
        </dispatchPolicy>
      </policyEntry>
    </policyEntries>
  </policyMap>
</destinationPolicy>
```
11. 让mq支持热配置

```
<broker xmlns="http://activemq.apache.org/schema/core" start="false" ... >
    <plugins>
      <runtimeConfigurationPlugin checkPeriod="30000" />
      ...
    </plugins>
    ...
</broker>
```
注：不要漏掉`start="false" `，`checkPeriod="30000"`是检查xml配置修改间隔(这里配置的是30s)，默认0，即不检查xml配置修改！

可以热配置的节点：
1. `<networkConnectors>`
2. `<destinationPolicy><policyMap><policyEntries>`
3. `<plugins><authorizationPlugin><map><authorizationMap><authorizationEntries>`
4. `<destinationInterceptors><virtualDestinationInterceptor><virtualDestinations>`
12. 事务的理解：表示多条消息发送到broker过程中,如果没有问题,则producer commit确认ok, 这多条消息才发送到consumer消费.
 当发送多条消息后发现异常(比如,数据错误),这时producer rollback确认事务失败, 则这写消息都不会发送到消费者!


- - -

## 3. 概念

### 1. 基本原理
1. 5.6之前的版本,mq为每一个消息队列分配一个线程处理消息的分发!5.6之后, 默认采用线程池来处理所有队列的消息分发!
2. P2P（point to point）中，没有得到确认的消息会被转发到其他的消费者。
3. producer默认使用同步方式发送消息（持久化消息的Producer send一个消息后block，直到broker将消息持久化到磁盘并发送acknowledgement。）

### 2. 消息预取限制
1. 设置预取消息的意义: 
**borker将消息push(推)到consumer的一个缓冲中, 由于push的速度肯定大于consumer消费消息的速度, 数据量大的情况, consumer的缓冲会满.所以要设置broker push消息到consumer的数量.  queue也是如此**, queue默认是1000, topic默认100, 通常在消费端初始化时设置.
    - `tcp://localhost:61616?jms.prefetchPolicy.all=50`
    - `tcp://localhost:61616?jms.prefetchPolicy.queuePrefetch=1`
    - 代码控制.       
       ` queue = new ActiveMQQueue("TEST.QUEUE?consumer.prefetchSize=10");`
       ` consumer = session.createConsumer(queue);`

2. **消息消费与预取相关过程**: 
 1. broker第一次push最大限制的消息到consumer.
 2. 直到consumer回复的acknowledge）达到50%的预取消息后,broker才再发送50%总预取限制数的消息到consumer,从而将consumer的缓冲填满.

3. 消费者速度控制
如果消息量较大的时候推荐设置较大的预取值；消息数量小,但是消费处理时间长的情况,建议设置预取值为1,确保消费者一次只处理一条消息. 
如果预取值设为0,表示consumer自己通过轮询的方式去broker中拉取消息，每次调用receive(timeout)方法时去拉取，这种方式效率较低，会明显增加每个消息的消费延迟！
**Slow Consumer**定义: 有他设置的预取值两倍的消息出去等待push状态!

4. 当pool consumer时，prefetch可能会出现的问题
push到prefetch缓冲区中的消息，如果没有消费，broker会一直认为该消息在消费中，所以只有consumer关闭的时候，prefetch的Unconsumer消息，才会被返回给broker！
所以当pool consumer后，consumer不会被销毁（close），所以那些Unconsumer消息，会等到这个consumer重用的时候才会被消费！
Tip：Spring的`CachingConnectionFactory`对象会缓存consumer，所以，要么关闭consumer缓存，要么设置`prefetch=0`。
> 参见：http://activemq.apache.org/what-is-the-prefetch-limit-for.html

### 3. 消息丢弃(discard)策略`ConstantPendingMessageLimitStrategy`

由于非持久化消息都会被保存在内存中，所以当消费者非常缓慢或者down掉,会导致RAM被填满,从而影响其他正常的消息。所以，将当一条新的消息到来且`prefetch buffer`被填满时，队列中旧的消息就会被丢弃。
`ConstantPendingMessageLimitStrategy` 表示保存在内存中的消息最大数量值（一般应该大于prefetchSize）。`-1`表示不丢弃消息，`0`表示在RAM中只保留prefetchSize大小的消息！
`<prefetchRatePendingMessageLimitStrategy multiplier="2.5"/>` 另一种计算方法，`2.5`表示保留在内存中的消息数量值为prefetchSize的2.5倍。
参见：`http://activemq.apache.org/slow-consumer-handling.html`

### 4. InactivityMonitor
连接监控(InactivityMonitor, 翻译: 不正常连接监控),参考`AbstractInactivityMonitor readCheck()和writeCheck()`源码
`writeCheck()`: 监测是否有正常的消息写入, 如果没有就用该连接发送一个`KeepAliveInfo`消息.
`readCheck()` : 如果有接收者在工作,则不监测, 否则监测消息是否在maxInactivityDuration内送达, 未送达抛`InactivityIOException`异常,给`TransportListener`!
不正常连接监视进程, 会检查每个连接是否是正常可用的,简言之就是如果一个连接在maxInactivityDuration时间内消息没有被接收(read),则会判定为不正常,且关闭这个连接.

连接监控的相关URI配置参数：
1. `wireFormat.maxInactivityDuration`    默认:30000毫秒, 消息被read能够容忍的最大时长. 可以理解成,一条消息write和read能够容忍的最长时长!
2. `wireFormat.maxInactivityDurationInitalDelay` 默认:10000毫秒 , 旨在解决当broker压力较大时创建一个连接需要较长时间.连接建立后,多长时间后开启`InactivityMonitor`任务.
3. `transport.useInactivityMonitor`  默认:true. 是否启动InactivityMonitor,监测不活动连接.
4. `transport.useKeepAlive`  默认: true. 当连接处于空闲状态时, 发送一条keep alive消息,用于InactivityMonitor线程检测是否在maxInactivityDuration时间内被read. 只有在没有正常消息发送时,才会发送一条keep alive来检测连接的正常性.有正常消息交互时InactivityMonitor会根据正常的消息的处理情况,检测连接的正常性.

InactivityMonitor连接监控理解：
InactivityMonitor采用两个daemon线程,`READ_CHECK_TIMER`读 ,`WRITE_CHECK_TIMER`写
注意与TCP连接url中的`keepAlive=true`的区别: 简言之`保证tcp层面的连接不被关闭!`. socket一段时间没有交互,系统会回收socket. 所以`keepAlive=true`定时发送一条0字节的消息,避免系统回收网络IO.
Tip: 网络IO资源回收两种方式: 1. 系统监测socket很长时间没有交互,则认为该连接dead,回收该IO. 2. 应用程序可以指定关闭IO资源, InactivityMonitor作用就是监测连接出现异常并关闭IO.
Tip:不使用InactivityMonitor两种方式: `transport.useInactivityMonitor=false` 和 `Configuring wireFormat.maxInactivityDuration=0` 。都是在URI中配置。

### 5. 可能用到的配置
- `<policyEntry queue=">" strictOrderDispatch="false" />` 配置queue是否严格按照顺序分派.
- 消费者优先级配置
`queue = new ActiveMQQueue("TEST.QUEUE?consumer.priority=10");    consumer = session.createConsumer(queue);`
优先级`0-127`,默认为`0`, 优先级最高为`127`, broker发送消息根据优先级高低, 首先发送消息到优先级最高的consumer,当这个consumer的 `prefetch buffer`满后, broker再给下一个优先级的consumer发送消息.


### 5. Advisory 消息
Advisory 消息是broker监听生产者和消费者的Topic类型消息,主要作用有:
1. 监听消费者生产者连接的建立和销毁.
2. 监听临时节点的创建和销毁.
3. 对于Topic和Queue的设置超时消息,发送通知.
4. 监听所有连接的建立和销毁.
5. 监听是否将消息发送到没有消费者的队列.
没建立一个Destination都会随之建立一个Advisory的Destination监听该Destination.

## 2. 禁用Advisory 消息
系统默认开启Advisory 消息,下面介绍如何禁用:
1. 关闭所有Advisory类型消息: broker的`activemq.xml`中配置 `<broker advisorySupport="false">`
2. 通过连接配置禁用: `tcp://localhost:61616?jms.watchTopicAdvisories=false`
3. 建立连接的时候配置:
`ActiveMQConnectionFactory factory = new ActiveMQConnectionFactory();`
`factory.setWatchTopicAdvisories(false);`
使用示例: `http://activemq.apache.org/nms/activemq-enumerate-destination-using-advisory-messages.html`

- - -

## 4. 协议
broker与client之间数据传输的协议大致有：openwire（默认），amqp，stomp，mqtt，amqp等。 
producer和broker以及consumer和broker之间的传输协议可以不同，可以针对业务场景分别指定。

1. openwire（配置：http://activemq.apache.org/configuring-wire-formats.html）
OpenWire是一种非常快的二进制协议，旨在实现最大化的性能和功能。ActiveMQ默认协议。
AMQP与OpenWire非常相似，因为他们都是被设计来通过二进制格式（比文本要高效），支持高性能的消息传递。

`activemq.xml`中配置:
`<transportConnector name="openwire" uri="tcp://0.0.0.0:61616?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>`
其中`maxFrameSize`表示每条消息的最大大小。
client链接配置：
`failover:(tcp://localhost:61616?wireFormat.maxInactivityDuration=0,tcp://10.100.8.5:61616?wireFormat.maxInactivityDuration=0)?randomize=false&amp;jms.useAsyncSend=true`
注意：maxInactivityDuration=0表示不关闭inactive连接，使用failover时，jms.*类型的参数应写在括号外面
`randomize=false` 表示,只有当当前连接的broker不可用,才连接到第二个broker上! 默认`randomize=true`,即随机选择一个可用broker连接,以提供负载均衡!其实,在HA架构下设不设置都可以,因为只有一个broker提供服务!
默认failover中第一个URI会被优先选择,如果想多个同时是优先选择的可以这样设置:
`failover:(tcp://local1:61616,tcp://local2:61616,tcp://remote:61616)?randomize=false&priorityBackup=true&priorityURIs=tcp://local1:61616,tcp://local2:61616`
即, local1和local2有相同的优先级,优先选择其中之一.

2. Stomp
Stomp,是一种简单方便的基于文本的协议，作为基于文本的格式，STOMP的实现非常简单，性能也比较低。

3. AMQP
AMQP：ActiveMQ 5.8新增加的传输连接。用于支持AMQP（高级消息队列协议）。
因为AMQP是消息队列的标准协议，而且已经越来越被广泛使用，所以ActiveMQ也支持了此协议。
AMQP协议可以搭配NIO或SSL协议使用，amqp+nio用于提升系统的延展性和性能。amqp+ssl可以创建安全连接。
**amqp配置：**`<transportConnector name="amqp" uri="amqp://localhost:5672"/>  `
**amqp+nio配置：**`<transportConnector name="amqp+nio" uri="amqp+nio://localhost:5672"/>  `
**amqp+ssl配置：**`<transportConnector name="amqp+ssl" uri="amqp+ssl://localhost:5672"/>  `


4. MQTT
MQTT：ActiveMQ 5.8新增加的传输连接。是一个轻量级的消息订阅/发布协议。和AMQP一样，同样支持搭配NIO或SSL使用。
**mqtt配置：**`<transportConnector name="mqtt" uri="mqtt://localhost:1883"/>  `
**mqtt+nio配置： **`<transportConnector name="mqtt+nio" uri="mqtt+nio://localhost:1883"/>  `
**mqtt+ssl配置： **`<transportConnector name="mqtt+ssl" uri="mqtt+ssl://localhost:1883"/>  `
5. http，udp，ssl，tcp，nio
 - http协议配置：`<transportConnector name="http" uri="http://localhost:8080"/>`
 - https协议配置：`<transportConnector name="https" uri="https://localhost:8080"/>`
 - udp协议配置：`<transportConnector name="udp" uri="udp://localhost:8123"/>`
 - ssl协议配置：`<transportConnector name="ssl" uri="ssl://localhost:8123"/>  `
SSL：需要一个安全连接的时候可以考虑使用SSL，适用于client和broker在公网的情况，如使用aws云平台等。

5. tcp配置： `<transportConnector name="tcp" uri="tcp://localhost:61616"/>  `
**TCP：ActiveMQ默认的传输连接，也是最常用的使用方式。长连接，每个客户端实例都会与服务器维持一个连接。每个连接一个线程。**
TCP的优点是：
性能高：**ActiveMQ使用默认协议OpenWire序列化和反序列化消息。OpenWire是一个性能很高的序列化协议。**
可用性高：TCP是使用最广泛的技术，几乎所有的开发语言都支持TCP协议。
可靠性高：TCP协议确保消息不会在网络传说的过程中丢失。

6. NIO配置：`<transportConnector name="nio" uri="nio://localhost:61616"/>`
NIO：使用Java的NIO方式对连接进行改进，因为NIO使用线程池，可以复用线程，所以可以用更少的线程维持更多的连接。
如果有大量的客户端，或者性能瓶颈在网络传输上，可以考虑使用NIO的连接方式。也可以根据不同的场景选择不用的传输连接.
比如：Producer有很多，但是Consumer很少，可以Producer用NIO协议，Consumer用TCP协议。
