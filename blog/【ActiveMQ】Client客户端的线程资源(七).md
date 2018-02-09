---
title: 【ActiveMQ】Client客户端的线程资源(七)
date: 2017/06/18 11:12:22
toc: false
list_number: false
categories:
- ActiveMQ
tags:
- ActiveMQ
---

**前言: **基于`activemq-client 5.14`版的开发，如果简单需求，可以参照网上的教程。但是面对较复杂或者想弄清楚发送和接收的过程，阅读源码的基础上再开发，十分有必要，这里提供了一份对`activemq-client`进行二次封装的实现，详见github`activemq-simple-client`。

本文从以下几个方面叙述：
1. 从线程的角度来阅读其源码，帮助快速了解整个逻辑。
2. 从流程出发，叙述producer和consumer源码的执行流程。
3. 从异常的角度，说明特殊情况的问题和可能的处理方式。
## 一. Client资源占用
### 1. client启动
创建一个生产者或消费者的基本流程: `new ActiveMQConnectionFactory(url)` --> `connectionFactory.createQueueConnection()`(其中会`new ActiveMQConnection`) --> `queueConnection.createQueueSession(transacted, acknowledge)` --> session创建`createQueue`或`createProducer` --> session创建`createConsumer`或`createProducer` --> ok!
- `createQueueConnection()`会创建一个客户端和Broker的连接,

### 2. client启动启用的线程
#### (1) `ActiveMQConnection`创建一个线程池用于异步处理一些杂碎的事情:
```
executor = new ThreadPoolExecutor(1, 1, 5, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>(), new ThreadFactory() {
            public Thread newThread(Runnable r) {
                Thread thread = new Thread(r, "ActiveMQ Connection Executor: " + transport);
                return thread;
            }
        });
```
杂碎的事情包括：
1. 异常情况，由于某些异常并不影响整个连接的正常逻辑，所以抛异常进行了异步处理；
2. 生产者的流控制，如果broker的Usage到达limit，则异步通知生产者broker的Usage，通知producer上的监听器（如果设置了usage的监听），以及处理Usage Full和Not Full的一些回调。

#### (2)`AbstractInactivityMonitor`创建两个Timer和一个线程池：
```
READ_CHECK_TIMER = new Timer("ActiveMQ InactivityMonitor ReadCheckTimer", true);
WRITE_CHECK_TIMER = new Timer("ActiveMQ InactivityMonitor WriteCheckTimer", true);

READ_CHECK_TIMER.schedule(readCheckerTask, initialDelayTime, readCheckTime);
WRITE_CHECK_TIMER.schedule(writeCheckerTask, initialDelayTime, writeCheckTime);

```

```
ASYNC_TASKS = createExecutor();
private final ThreadFactory factory = new ThreadFactory() {
        @Override
        public Thread newThread(Runnable runnable) {
            Thread thread = new Thread(runnable, "ActiveMQ InactivityMonitor Worker");
            thread.setDaemon(true);
            return thread;
        }
    };
private ThreadPoolExecutor createExecutor() {
        ThreadPoolExecutor exec = new ThreadPoolExecutor(0, Integer.MAX_VALUE, getDefaultKeepAliveTime(), TimeUnit.SECONDS, new SynchronousQueue<Runnable>(), factory);
        exec.allowCoreThreadTimeOut(true);
        return exec;
    }
```
Timer定时执行`readCheck`和`writeCheck`，真正的执行是丢给`ASYNC_TASKS`这个线程池去执行！

#### (3)`TransportThreadSupport`创建一个Tcp连接线程
该线程将socket收到的消息enqueue到MessageDispatchChannel（SimplePriorityMessageDispatchChannel具体保存每条消息的容器）中，并唤醒（新建）线程池（如果线程池没有实例化）。
```
protected void doStart() throws Exception {
        runner = new Thread(null, this, "ActiveMQ Transport: " + toString(), stackSize);
        runner.setDaemon(daemon);
        runner.start();
    }
```
线程名Like：`ActiveMQ Transport: tcp:///xxx.87:56161@48695`
这个线程是在`ActiveMQConnectionFactory.createActiveMQConnection()`中创建连接并开启IO连接相关线程（包括InactivityMonitor）：
```
Transport transport = createTransport(); //创建一个Tcp连接
connection = createActiveMQConnection(transport, factoryStats);
//....
transport.start(); //开启连接线程
```
`createTransport()`创建线程过程：`TcpTransportFactory -- createTransport -- new TcpTransport`
其中，`TcpTransport`对象两个参数`socketBufferSize = 64 * 1024;`和`ioBufferSize = 8 * 1024;`前者是Socket的buffer，后者是JVM中`InputStream`的buffer！
Tip：每个Consumer和Producer实例都对应这样一个`ActiveMQ Transport`线程，该线程用于接收socket的发送回来的包（eg：ack包数据）。
Tip：producer中同步调用方式中，调用线程直接将数据`marshal`后交给给socket，这个`ActiveMQ Transport`线程则负责接收socket回来的ACK包，然后写入阻塞队列`responseSlot = new ArrayBlockingQueue<Response>(1)`，此之前调用线程会阻塞在`responseSlot.take();`方法上。直到`ActiveMQ Transport`线程收到ACK。详见：`FutureResponse`

#### (3.1) Failover连接线程池（用于管理网络连接，比如断开重连）
以上，是基于`tcp://ip:61616`类型连接下情况，当使用`failover:(tcp://primary:61616)`类型连接时，会额外创建一个命名为`ActiveMQ Task-`的线程池。
该线程池和session的consumer消费线程池一样，`TaskRunnerFactory.createDefaultExecutor()`中创建，`corePoolSize=0，maxPoolSize=Integer.MAX，timeout=30s，SynchronousQueue`。
该线程池会在系统系统由调用线程实例化，`FailoverTransportFactory`中调用TaskRunnerFactory创建PooledTaskRunner线程池的封装对象，它用于创建failover类型的连接，随后30s，该线程池线程会被回收，所以查询系统线程很难发现它。
当broker 宕掉或连接断开，此时由`ActiveMQ Transport:xxx`线程调用，又会创建一个`ActiveMQ Task-`线程来进行重连接，不停地尝试连接每次尝试10次，重连中如果不成功会抛下面异常：
```
[08-10 17:25:33.031 [ActiveMQ Task-3] WARN o.a.a.transport.failover.FailoverTransport.doReconnect:1100]-Failed to connect to [tcp://xx:56161] after: 10 attempt(s) continuing to retry.
[08-10 17:30:03.781 [ActiveMQ Task-3] WARN o.a.a.transport.failover.FailoverTransport.doReconnect:1100]-Failed to connect to [tcp://xx:56161] after: 20 attempt(s) continuing to retry.
```
Tip：该线程池用完就会被回收，且仅用于failover类型连接的建立和重连，producer发送消息于该线程池无关。

#### (4)重头戏Session相关的线程：
`ActiveMQSession`是从建立连接到消费或生产消息，这整个逻辑的核心，保存着所有的信息(对象)，以组合的方式将`ActiveMQConnection`连接引入，解决消费和生产消息链路！
Session处理收发消息是`ActiveMQSessionExecutor`对象，其中涉及几个重要的成员变量：`MessageDispatchChannel`保存消息，`TaskRunner`线程池处理消息。
Session创建过程：`ActiveMQConnection.createSession()`首先连接创建完毕并start，通过ActiveMQConnection对象创建ActiveMQSession对象并把自己传给ActiveMQSession。
创建ActiveMQSession中同时创建`ActiveMQSessionExecutor`收发消息，`SimplePriorityMessageDispatchChannel`保存消息，`PooledTaskRunner`线程池处理消息等对象。
另外，`ActiveMQMessageConsumer`和`ActiveMQMessageProducer`是消费消息和生产消息的实现，在Session和Connection基础上进一步的封装。
将`ActiveMQConnection`以组合方式封装到`ActiveMQMessageConsumer`和`ActiveMQMessageProducer`中。
同时，一个Session可以对应多个Consumer和Producer，所以Session中保存了Consumer和Producer的实例引用：
```
    protected final CopyOnWriteArrayList<ActiveMQMessageConsumer> consumers = new CopyOnWriteArrayList<ActiveMQMessageConsumer>();
    protected final CopyOnWriteArrayList<ActiveMQMessageProducer> producers = new CopyOnWriteArrayList<ActiveMQMessageProducer>();
```
创建一个Session线程池流程：`ActiveMQConnection`对象中保存`TaskRunnerFactory sessionTaskRunner`创建线程池（`TaskRunner`）的引用，对外提供`getSessionTaskRunner()`方法，该方法是一个`TaskRunner`创建工厂！
`ActiveMQSessionExecutor`中当消息进入队列(MessageDispatchChannel)会触发`wakeup()`方法，`wakeup()`会创建`PooledTaskRunner`对象并创建线程池！

```
this.taskRunner = session.connection.getSessionTaskRunner().createTaskRunner(this,"ActiveMQ Session: " + session.getSessionId());
//....
taskRunner.wakeup();
```
Tip：该线程池也会在`ActiveMQMessageConsumer`实例化的时候，且connection已经启动后start！
`TaskRunnerFactory`用于创建线程池，最大1000个线程，keepAliveTime 30s，SynchronousQueue阻塞消息！
```
protected ExecutorService createDefaultExecutor() {
        ThreadPoolExecutor rc = new ThreadPoolExecutor(0, getMaxThreadPoolSize(), getDefaultKeepAliveTime(), TimeUnit.SECONDS, new SynchronousQueue<Runnable>(), new ThreadFactory() {
            @Override
            public Thread newThread(Runnable runnable) {
                String threadName = name + "-" + id.incrementAndGet();
                Thread thread = new Thread(runnable, threadName);
                thread.setDaemon(daemon);
                thread.setPriority(priority);

                LOG.trace("Created thread[{}]: {}", threadName, thread);
                return thread;
            }
        });
        if (rejectedTaskHandler != null) {
            rc.setRejectedExecutionHandler(rejectedTaskHandler);
        }
        return rc;
    }
```


