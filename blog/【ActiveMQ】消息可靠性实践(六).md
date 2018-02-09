---
title: 【ActiveMQ】消息可靠性实践(六)
date: 2017/06/17 11:12:22
toc: false
list_number: false
categories:
- ActiveMQ
tags:
- ActiveMQ
---

## 一. Broker
1. **HA架构**
ActiveMQ提供Master-Slaver模式的主从架构模式，简单Master-Slaver只能保证消息的可恢复，Master-Slaver-Slaver模式能保证系统的高可用，其中一台宕掉，仍能正常提供服务。
2. **持久化**
Broker支持基于Kahadb和LevelDB的消息持久化，另一方面支持消息统一的数据库和共享存储模式，消息的持久化是消息可靠性的最重要环节，如果系统重启 ，宕机都能保证消息可恢复。
3. **ACK**
网络环节上，ActiveMQ采用ACK保证消息正确到达，提供三种模式的ACK：
**AUTO_ACKNOWLEDGE**：Producer每发一条Broker持久化到磁盘成功都会回复一条成功消息；Consumer每消费一条都会回复Broker一条的ACK消息，如果消费异常也会发一条携带异常的消息给Broker，Broker会将该消息放入DLQ队列。自动确认并不一定是每成功一次就往Broker发送一条ACK消息，当开启optimizeACK后，会在某个时间点批量发ACK确认消息！
**CLIENT_ACKNOWLEDGE**：Producer每发一条消息，要等到Consumer成功消费，并发送ACK到Broker，Broker才会返回一条ACK给Producer。`message.acknowledge()`方法可以批量确认一批消息，也可以消费一条确认一条。
**DUPS_OK_ACKNOWLEDGE**：消息批量确认，Broker不每次成功持久化一条消息都发一条ACK给Producer，Producer也不用等待上一条ACK回来才再发下一条消息，他们之间采用批量ACK的方式！Consumer则由于Prefetch的存在，Broker无需等待Consumer的每一个确认ACK，而是直接填充Prefetch缓冲区，但Consumer消费消息回复ACK是采用批量回复的方式。
Tip：还有**SESSION_TRANSACTED**事务类型消息确认和**INDIVIDUAL_ACKNOWLEDGE**单条消息确认机制，这里不详述了。(http://shift-alt-ctrl.iteye.com/category/304508)讲的比较清楚！
4. **Networke Monitor**
网络环节上，ActiveMQ为保证链路的通畅，会创建readCheck和writeCheck定时监测线程，实时的监测链路的可用性！
5. **Failover**
网络环节上，我们可以配置failover模式（单机也可以配置），ActiveMQ为保证链路故障后的可自动恢复，会开启`ActiveMQ Task-x`的线程池，来连接和重连，保证连接断开后能再次连上。

上面列出了我运维开发中理解较深刻的五类消息可靠性的策略，当然，ActiveMQ在保证消息可靠性上远不止这些，保证消息的可靠性是一个MQ最基本也是最重要的部分，而且整个中间件的设计都是在围绕可靠性基础上的。

## 二. Client
核心部分，上面的是ActiveMQ提供的可靠性，在实际开发中，保证消息的可靠性，我们也有许多要做的，先问几个问题：
1. 系统重启或网络故障，consumer端如何保证消息不丢失？
2. 系统重启或网络故障，producer端如何保证消息不丢失？
3. 原生提供的消费策略是线程池单线程消费模式，当采用线程池多线程消费消息时，太多的消息会导致Reject策略，consumer端如何保证消息不丢失？
4. 当Broker短暂不可用，原生client策略是阻塞或异常抛弃，阻塞是调用线程的阻塞，这就会导致很多问题，那么如何保证期间producer发送的消息不会丢失？
带着以上几个问题，我这里给出我解决策略：
### Consumer
`activemq-client.jar`（后面用client替代）提供的消费策略是，开启一个线程池，每次拿一个线程去消费Broker push过来的消息，如果异常，则抛给Broker保存到DLQ队列中。
这种策略当系统重启或网络故障，未收到ACK的消息Broker会再次发送给其他的消费者。
当需要消费大量消息时，client策略肯定不行，理所当然使用多线程来解决，如果使用`SynchronousQueue`或指定长度的`LinkedBlockingQueue`太多消息过来会导致Reject，如果`LinkedBlockingQueue`设置太长，很容器导致OOM（亲身体会），不啰嗦了给出我的策略。
1. 设置线程池如下：
```
service = new ThreadPoolExecutor(nThread, nThread, 0L, TimeUnit.SECONDS,
                new LinkedBlockingQueue<Runnable>(queueSize),
                new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                        logger.info("create thread,current thread:{},poolNumber:{}", Thread.currentThread().getName(), poolNumber.get());
                        return new Thread(r, "MQConsumer-Thread-" + poolNumber.incrementAndGet());
                    }
                }, new ThreadPoolExecutor.CallerRunsPolicy());

registerShutdownHook();
```
线程个数和阻塞队列长度可根据情况设置，我这里是10个线程，1000的阻塞队列长度，再来消息，由`ActiveMQ Session`线程自己处理，这样有效控制了消费速度！
随之而来的问题，就是，保存1000条消息的阻塞队列，`ActiveMQ Session`线程将消息递给线程池就认为消费成功，当系统重启，这些消息都有可能丢失，所以这里采用钩子线程！
```
public void registerShutdownHook() {
        shutdownHook = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    shutdownSemaphore.acquire();
                    if (service.isTerminating() || service.isShutdown() || service.isTerminated()) {
                        return;
                    }
                    logger.info("shutdown hook,{} task retain in the queue", service.getQueue().size());
                    closeThreadPool();
                    logger.info("close pool finish,isTerminated:{}", service.isTerminated());
                } catch (InterruptedException e) {
                    logger.error("shutdown semaphore acquire exception.", e);
                } finally {
                    shutdownSemaphore.release();
                }
            }
        }, "MessageProcessor-shutdown-hook-thread");

        Runtime.getRuntime().addShutdownHook(shutdownHook);
        shutdownSemaphore = new Semaphore(1);
    }
private void closeThreadPool() {
        if (service != null && !service.isShutdown()) {
            service.shutdown();
            try {
                service.awaitTermination(1, TimeUnit.HOURS);
            } catch (InterruptedException e) {
                logger.error("await pooled task execute finish error.", e);
            }
        }
    }
```
如果，消费消息依赖Spring的bean，那么很可能，在hook线程执行时，Spring已经销毁了，所以在spring的销毁`ContextClosedEvent`事件上调用一个close方法：
```
public void close() {
        if (isUsedThreadPool) {
            try {
                shutdownSemaphore.acquire();
                if (service.isTerminating() || service.isShutdown() || service.isTerminated()) {
                    return;
                }
                Runtime.getRuntime().removeShutdownHook(shutdownHook);
                logger.info("close MessageProcessor,{} task retain in the queue", service.getQueue().size());
                closeThreadPool();
                logger.info("close MessageProcessor finish,isTerminated:{}", service.isTerminated());
            } catch (InterruptedException e) {
                logger.error("shutdown semaphore acquire exception.", e);
            } finally {
                shutdownSemaphore.release();
            }
        }
    }
```
`shutdownSemaphore`信号量保证，不管是hook还是close，只被调用一次，且保证了spring要等到线程池关闭后才能销毁（这里用锁也可以实现）！
经过实践和测试，上面保证了大量消息的及时快速消费，也保证了每次发布系统重启，消息的不丢失，具体代码可参见我的github！
Tip：如果使用线程池的其他reject策略，还需需要注意`isTerminating()=true`时，新任务到来的情景。

### Producer
Producer的client端，系统重启或网络故障在Client文中有详细讲，这种情况下，client是不能保证消息不丢失的，或者说是不稳定的。
不啰嗦，很晚了- -，给出我的策略：
采用failover当连接断开时，阻塞在该条消息发送处，一直等待连接重新建立完成，设置一个缓冲区，将新来的消息缓存在其中！
代码：
```
public void sendImportantMessage(final String message) {
        if (message == null || message.isEmpty()) {
            logger.error("parameter of send message is empty.");
            return;
        }
        sendMessagePool.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    TextMessage textMessage = queueSession.createTextMessage(message);
                    producer.send(textMessage);
                } catch (Exception e) {
                    logger.error("send message error.message:{}", message, e);
                }
            }
        });
    }

    private static final AtomicInteger poolNumber = new AtomicInteger(0);
    private static final int CACHE_MESSAGE_SIZE = 10000;
    private ThreadPoolExecutor sendMessagePool;

    public void initSendMessagePool() {
        //注意并发问题
        if (sendMessagePool != null) {
            return;
        }
        sendMessagePool = new ThreadPoolExecutor(1, 1, 0L, TimeUnit.SECONDS,
                new LinkedBlockingQueue<Runnable>(CACHE_MESSAGE_SIZE),
                new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                        return new Thread(r, "MQSendMessage-Thread-" + poolNumber.incrementAndGet());
                    }
                }, new ThreadPoolExecutor.DiscardOldestPolicy());

        Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
            @Override
            public void run() {
                closeSendMessagePool();
            }
        }, "Producer-shutdown-hook-thread"));
    }
```
每条消息过来，都直接交给线程池处理，线程池中LinkedBlockingQueue默认是10000，即可缓存1w条消息，当连接断开，会阻塞在` producer.send(textMessage);`
但是仍可以接收外面传过来的消息，当连接恢复，这些消息再被push到broker。
同样，要设置一个hook，系统重启上，spring是否销毁不会影响后续发送：
```
private synchronized void closeSendMessagePool() {
        try {
            if (sendMessagePool == null || sendMessagePool.isShutdown() || sendMessagePool.isTerminated()) {
                return;
            }

            logger.info("Producer shutdown,queue size:{}", sendMessagePool.getQueue().size());
            sendMessagePool.shutdown();
            sendMessagePool.awaitTermination(1, TimeUnit.HOURS);
            logger.info("Producer close pool finish,isTerminated:{}", sendMessagePool.isTerminated());
        } catch (Exception e) {
            logger.error("close producer pool exception.", e);
        }
    }
```
但是要保证在producer关闭前发送完这些消息，所以要下面的close：
```
 public void close() {
        if (sendMessagePool != null && !sendMessagePool.isShutdown()) {
            closeSendMessagePool();
        }

        try {
            producer.close();
        } catch (JMSException e) {
            logger.error("close producer exception", e);
        }

        try {
            queueSession.close();
        } catch (JMSException e) {
            logger.error("close queueSession exception", e);
        }

        try {
            queueConnection.close();
        } catch (JMSException e) {
            logger.error("close queueConnection exception", e);
        }
    }
```
当使用者在关闭应用时关闭连接，此时，hook和close，其中之一会执行剩余消息的发送逻辑，如果是hook发送，close会阻塞在 `closeSendMessagePool();`方法上，保证了连接在消费完消息后再关闭。
此时，可能会有个疑问，为什么不使用client提供的`producerWindowSize+useAsyncSend`方式提供的异步发送模式呢？
答：当然，这种方式是可以的，但是有几个问题，这种方式并没有一个线程来负责将消息放入`producerWindowSize`的`buffer`中，如果failover不加timeout，会阻塞调用线程，从而阻塞主逻辑。
当加timeout，同样阻塞主逻辑，等待timeout超时，主逻辑才能执行下一条。但是我的这个逻辑加上`producerWindowSize+useAsyncSend`这个逻辑也是可以，没问题的，但是得看场景吧。

拓展阅读: 
Kafka消息可靠性保证: http://blog.csdn.net/u013256816/article/details/71091774




