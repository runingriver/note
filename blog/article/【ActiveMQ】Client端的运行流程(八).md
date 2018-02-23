---
title: 【ActiveMQ】Client端的运行流程(八)
date: 2017/06/19 11:12:22
toc: false
list_number: false
categories:
- ActiveMQ
tags:
- ActiveMQ
---

# 一. Client运行流程
## 1. Client Consumer的工作流程
消息到达socket，触发`TCPTransport.run()` --  `TCPTransport.doRun()` -- ... -- 调用`ActiveMQConnection.onCommand(obj)`-- `ActiveMQSession.dispatch(MessageDispatch messageDispatch)` -- 调用`ActiveMQSessionExecutor.execute()`
**这里分两中情况：**
第一种，系统启动第一条消息进来： `ActiveMQSessionExecutor.wakeup()`(唤醒线程池) -- `taskRunner.wakeup()`创建线程池 -- `PooledTaskRunner.wakeup()` -- 线程池拿一个线程执行`execute()` -- `PooledTaskRunner.runTask()`（每次最多处理1000条消息） -- 从`ActiveMQSessionExecutor.iterate()`方法遍历MessageDispatchChannel容器获取数据。
第二种，线程池已经建立，直接`ActiveMQSessionExecutor.dispatch(MessageDispatch message)`;
两种情况都是将每条消息交给`ActiveMQSessionExecutor.dispatch(MessageDispatch message)`分发 -- 通过ActiveMQSessionExecutor中保存的session获得ActiveMQMessageConsumer -- `ActiveMQMessageConsumer.dispatch(MessageDispatch md))` -- ActiveMQMessageConsumer中保存之前setMessageListener()的MessageListener对象调用其onMessage()方法，最后调用我们自己定义的业务逻辑代码！-- 最后，处理成功回复acknowledge：`ActiveMQMessageConsumer.deliverAcks()` -- 结束。

解释：
`PooledTaskRunner`负责线程池执行逻辑，每次执行最多消费1000条消息，将消费逻辑交给ActiveMQSessionExecutor来处理
`ActiveMQSessionExecutor`中保存了Session对象，消息容器MessageDispatchChannel（TCPTransport线程将消息放到此对象中）对象，PooledTaskRunner对象，作为成员变量。
ActiveMQSessionExecutor扮演者接收和分发（泵）的角色，接收TCPTransport socket发来的消息保存到MessageDispatchChannel中。
将消息轮流分发到各个ActiveMQMessageConsumer（通过session获取该对象）去处理，这多个消息通过线程池PooledTaskRunner中的线程去处理！

**线程池的执行逻辑：**
ActiveMQ Session 线程池，对每个队列（每个consumer）同一时刻只会有一个线程来处理消息，且线程每次先最多消费1000条消息，
如果MessageDispatchChannel队列中还有未消费的消息，再次执行最多1000次的迭代逻辑！迭代次数依赖prefetch size的大小，1000次是默认prefetch size的大小。
这是一次线程池的调用执行过程（run()方法执行一次），如果一条新消息到来，会调用wakeup()，启动第一次的话，会新建线程池，后面会将消息丢到`SimplePriorityMessageDispatchChannel`队列，
然后，唤醒线程池执行消费消息！
下面是摘自`org.apache.activemq.ActiveMQMessageConsumer#dispatch()`方法：
```
if (++dispatchedCount % 1000 == 0) {
    dispatchedCount = 0;
     Thread.yield();
 }
```
没循环1000次就让出cpu！
另外，`org.apache.activemq.thread.PooledTaskRunner#runTask()`方法，是线程池多次循环调用`org.apache.activemq.ActiveMQMessageConsumer#dispatch()`方法的起点，其中有段代码值得看一看：
```
 for (int i = 0; i < maxIterationsPerRun; i++) {
                LOG.trace("Running task iteration {} - {}", i, task);
                if (!task.iterate()) {
                    done = true;
                    break;
                }
            }
```
注：`runTask()`的逻辑是：每次拿到执行权就迭代`maxIterationsPerRun`次（默认也是1000），执行完交出执行cpu，如果`SimplePriorityMessageDispatchChannel`中还有消息没有消费完，则拿到执行权后继续上面的逻辑！
注意：多个消费者可能会共用同一个`ActiveMQ Session Task-x`线程消费，也可能多个消费者对应多个线程，但是同一个消费者只会有一个线程来处理。

**优先级实现逻辑：**
MQ优先级消息队列采用在`SimplePriorityMessageDispatchChannel`中定义`LinkedList<MessageDispatch>[] lists;`实现,如果有10级优先级，则数组大小为10！
一个`ActiveMQMessageConsumer`对象，也即一个队列对应一个`MessageDispatchChannel`对象，`ActiveMQMessageConsumer`对象的所有消息都会按设定的优先级保
存在`MessageDispatchChannel`对象的`LinkedList<MessageDispatch>[] lists;`中。
ActiveMQ Session线程池会不停的迭代`LinkedList<MessageDispatch>[] lists;`中保存的元素，从高优先级的下标开始迭代，直至为空。
Tip：每个queue中的每条消息都有自己的优先级，默认是4，即消息优先级是根据单挑消息设定的，而不是根据队列设定的！

## 2. Client Producer的工作流程
producer的同步发送模式：
1. 调用线程`mainThread`调用`producer.send(msg)`发送一条消息 ==> `ActiveMQSession#send()`这里决定同步或异步发。
2. 同步发送：`syncSendPacket()`,这里会注册一个callback,调用完`TcpTransport.oneway()`后回调阻塞在`response = resp.getResult();`方法.
3. `mainThread`会走到`FailoverTransport.oneway()` ==> 再到`TcpTransport.oneway()`调用socket方法并`flush()`发送一个数据出去.
4. `mainThread`将数据提给socket发送后,会阻塞等待ack的返回!
5. `ActiveMQ Transport`线程在`TcpTransport`中的`run()`方法中死循环(`while (!isStopped()) { doRun(); }`),接收socket返回的数据.
6. `ActiveMQ Transport`线程接收到数据后处理,层层`onCommand()`后,将数据交给`mainThread`线程,最后`mainThread`线程收到ack后返回.
7. 一条同步producer消息发送完成.

`mainThread`线程和`ActiveMQ Transport`线程是怎么配合一发一收的呢?
答: 主要实现在`FutureResponse`方法中,通过`ArrayBlockingQueue<Response> responseSlot = new ArrayBlockingQueue<Response>(1)`实现.
这是一个只包含一个元素的阻塞队列,`mainThread`线程发完数据后`responseSlot.take()`获取数据肯定会阻塞等待数据的到来.
`ActiveMQ Transport`接收到数据后`responseSlot.offer(result)`从而接触`mainThread`的阻塞.

producer的同步发送与异步发送的区别:
`mainThread`线程将Message封装后提交给socket后要等待ack的返回,才算调用完成;而异步调用则是将Message封装后提交给socket后直接返回,并调用完成;

注意: 异步调用如果不设置`ProducerWindowSize`,不管消息是否发送成功,都得不到通知,消息发送不成功主要因为broker的Usage到达极限;
`ProducerWindowSize`的含义:
> The ProducerWindowSize is the maximum number of bytes of data that a producer will transmit to a broker before waiting for acknowledgment messages from the broker that it has accepted the previously sent messages.

即，异步发送模式下，当发送`ProducerWindowSize`大小的数据后调用线程会阻塞住，等待broker的acknowledgment messages（包括ACK和异常等）；
这时如果broker的Usage到达极限，调用线程可以得到通知，从而进行容错处理；
**所以, 我们采用异步调用的时候一定要设置`ProducerWindowSize`的大小,ActiveMq的jar中默认设置该值为`0`。**

Tip：如果broker的Usage到达极限，其还可能阻塞在`producerWindow.waitForSpace();`方法上，等待broker有空间后再发。


## 3. 大量消息（高并发）下，消费者处理策略
解决高并发下load高,消息消费慢的问题
首先,如果不使用线程池,消息消费非常慢,系统资源没有充分利用,broker大量消息pending!
如果,使用线程池,需要解决消息异常,restart等情况消息丢失,大量消息不及时消pending在内存中费导致OOM等问题.
另一方面,HBase没有pool,每次写入都要new对象等,导致load较高,消费较慢,加入对象缓存,加快写入hbase!
特殊情况处理,如一次机房迁移,ZK的master被关闭,大量Pending在broker的消息,导致MQ不能正常选举,最终导致MQ不能对外提供服务!
首先,尽量不让消息pending到broker,及时恢复MQ策略,将消息丢失降到最低,以及ZK和MQ集群机器异常的预处理策略!
测试,解决bug!

高并发消息解决方案：
问题：使用MQ client提供的消费消息线程池，在处理大量消息时会非常慢。
解决方案：
1. 最简单的方案，多线程每次主动去broker拉取消息！实际效率并不高！
2. 采用新建线程池来接手Session线程池消息的具体业务逻辑！实际效率较高，但是较复杂（要处理消息丢失，异常等策略）！


# 二. Client运行异常的处理

### 1. Consumer相关异常
**结论：**如果直接消费消息，也即使用Session线程消费消息，如果逻辑中抛出异常（一般是unchecked异常），在`ActiveMQMessageConsumer.dispatch()`方法中catch住，`isAutoAcknowledgeEach()`为true，执行rollback()，broker会将该消息放入`ActiveMQ.DLQ`队列！这种异常情况导致消息消息处理非常慢，如果大量异常，大量消息会pending到broker！
**注意：**如果将消息交给线程池处理，那么消费消息异常只会导致该线程销毁，不会被Session线程捕获，并重新交给broker处理！原理如下：
```

    @Test
    public void testRunableAndCallable() {
        final ExecutorService service1 = Executors.newSingleThreadExecutor();
        ExecutorService service2 = Executors.newSingleThreadExecutor();
        service2.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    service1.execute(new Runnable() {
                        @Override
                        public void run() {
                            logger.info("enter runnable.");
                            throw new RuntimeException("mock unchecked exception.");
                        }
                    });
                    logger.info("service2 finish.");
                } catch (Exception e) {
                    logger.error("error:{}", e);
                }
            }
        });
        logger.info("finish test.");
    }
```
上面这个单元测试，最后会报告成功，日志中会有异常信息，是由于service1的一个线程异常被片抛到栈顶！并没有抛给Test主线程，也没有抛给service2线程，
原理和Session线程池将任务交给另一个线程池执行原理相同！
**值得注意的是：** 如果使用新建线程池执行Session线程池提交的任务(onMessage()通过传递方法执行)，新建线程池中采用的是`LinkedBlockingQueue<Runnable>()`，会发生什么情况？
如果是`LinkedBlockingQueue<Runnable>(1)`或者是`SynchronousQueue<Runnable>()`，会发生什么情况？
直接给出我的答案：
如果是`LinkedBlockingQueue<Runnable>()`，当有大量消息到来时，消息会保存到链表中导致OOM或者长时间Full GC（导致各种连接超时，重试，重连），而且
如果此时服务重启（或kill线程等），这些保存在LinkedBlockingQueue中的消息会被无声无息的丢弃（放到LinkedBlockingQueue后，session线程就认为消息消费成功，发送ACK了）。
如果是`SynchronousQueue<Runnable>()`或`LinkedBlockingQueue<Runnable>(1)`，当不限定最大线程池时，Session来条消息如果没有空闲线程，则会新建一个线程，大量消息同时到来时，
会创建大量线程，很容易导致OOM；如果限定线程最大大小，当线程池达到最大线程个数，再有消息到来没有可用线程处理时，会采取拒绝策略，默认是抛异常！

**还得在说一句：**在用线程池处理消息的时候，注意要catch住所有异常，否则，在大量消息失败时，异常会抛到栈顶，导致线程死亡，这样就会发生线程池频繁创建线程的现象！

**拓展：**疑问继续
使用线程池A接收Session线程池的消息，使Session线程池只起一个消息转送角色，本地线程池A来处理真正的具体业务逻辑。但是当大量消息到来线程池A中缓存队列已满，该如何处理？
方案一：最简单有效方式，是使用`CallerRunsPolicy`策略，即多的任务反交给Session线程池去处理，这样有效地解决了速度控制的问题！
方案二：使用Semaphore，当线程池A到达极限，则让Seesion线程池等一下。也可以拓展现有线程池，实现成阻塞线程池！
方案三：使用流量控制，比如使用guava的RateLimiter，使用固定速率往线程池A中push消息！

### 2. Connection和Producer相关异常
首先，一个问题，broker宕机或网络断掉，producer持续不断的在发消息，会发生什么情况，发送的消息会丢失吗？
分两种情况：`tcp://ip:61616`类型连接和`failover:(tcp://primary:61616)`类型连接
开发中我们通常会设置两种监听器：`ExceptionListener`和`TransportListener`
设置`TransportListener`：`connectionFactory.setTransportListener(transportListener)`设置监听连接异常!
设置`ExceptionListener`：
`connectionFactory.setExceptionListener(new MQExceptionListener("consumer connectionFactory"));`
`queueConnection.setExceptionListener(new MQExceptionListener("consumer queueConnection"));`
注：这两个都是设置的同一个listener,且后面的会覆盖前面的!
注：consumer一般不设置该监听器，因为每条消息过来都会触发一次onCommand()的回调!

1. `tcp://ip:61616`类型连接
当broker宕机或网络断掉，连接断开(broker down掉)，无论consumer或producer，如果注册了`ExceptionListener`和`TransportListener`，则两者都会监听到异常!都是在`onException`方法中抛异常!
在Producer中，如果生产者还在不停地生产消息，每生成一条消息，session会在创建一条消息前（`queueSession.createTextMessage(message)`），
检查session是否close（`ActiveMQSession.checkClosed() `），此时连接断开，`checkClosed()`中会抛`IllegalStateException("The Session is closed")`异常到调用线程！
Broker和网络恢复，consumer和producer不能自恢复！
注：如果连接是failover，连接会自动恢复，否则需要重启!


2. `failover:(tcp://primary:61616)`类型连接
`failover:(tcp://primary:61616)?timeout=3000`
如果连接断开`ActiveMQ Task-`线程会不停地尝试每次尝试10次，对于每条消息会等待3000ms，若扔不能发送出去则抛异常，直到连接重新建立，恢复正常，但是这期间异常消息会被丢弃，不会重新发送!
`failover:(tcp://primary:61616)`
不加timeout，连接断开，监听的TransportListener中抛transportInterupted异常，`ActiveMQ Task-`线程不停地尝试每次重连10次，发送消息线程在该消息处阻塞住，直到连接重新建立，恢复正常，触发TransportListener.transportResumed()，这期间没有消息发送，不会出现消息丢失的情况!
注：failover连接下，consumer端，如果连接broker down掉或连接中断，不会抛任何异常(没有注册`TransportListener`)，直到failover连接重新建立，重新开始消费消息!
注：failover连接下的producer端，注册了`ExceptionListener`，连接断开不会抛`ExceptionListener.onException()`异常!


