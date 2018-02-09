---
title: 【ActiveMQ】Client开发遇到的问题(九)
date: 2017/06/12 11:12:22
toc: false
list_number: false
categories:
- ActiveMQ
tags:
- ActiveMQ
---

## 1. 开发的一次OOM
1. OOM程序逻辑
程序注册`MessageListener`进行消息的消费，采用一个`newFixedThreadPool(10)`的线程池处理消息。
2. Broker的架构
AMQ采用HA架构，一台Master，两条Slaver，消息持久化是LevelDB，通过注册中心Zookeeper，实现Master选举。
3. 问题
首先，运行一段时间，发现线程池的线程名在变化，加上一个计数，变化到6000多，也就是说新建和销毁6000多次线程！然后，运行一段时间后程序OOM！
### 解决
1. OOM第一时间dump内存，保留现场！
2. 分析程序逻辑：程序默认Consumer默认有一个缓存区保存1000条消息，这些消息丢给线程池执行，而消息较多时线程池会先将消息放入到`LinkedBlockingQueue`阻塞队列。
表象上看，消息消费非常快，实则是保存在了线程池的队列中！
3. 果不其然，MAT分析，`LinkedBlockingQueue`占了所有可用的内存，GC日志发现程序不停的Full GC。
Full GC会导致数据库连接失败，HBase连接失败，Zookeeper连接等一系列的网络IO超时失败！
对于Activemq连接超时后，会执行重连，一段时间后会自己关闭连接（`ThreadPoolUtils.awaitTermination`）将线程池关闭！
```
INFO o.a.activemq.util.ThreadPoolUtils.doShutdown:152]-Shutdown of ExecutorService: java.util.concurrent.ThreadPoolExecutor@4bb34ae[Shutting down, pool size = 2, active threads = 2, queued tasks = 0, completed tasks = 7] is shutdown: true and terminated: false took: 10 minutes.
```
而查看Broker端会有一个Inactive Monitor线程，检测到当前连接不可用会关闭当前连接。
```
2017-07-29 21:23:11,755 | WARN  | Transport Connection to: tcp://xx.xxx.106.24:48737 failed: org.apache.activemq.transport.InactivityIOException: Channel was inactive for too (>30000) long: tcp://xx.xx.106.24:48737 | org.apache.activemq.broker.TransportConnection.Transport | ActiveMQ InactivityMonitor Worker
2017-07-29 21:28:51,837 | WARN  | Ignoring ack received before dispatch; result of failover with an outstanding ack. Acked messages will be replayed if present on this broker. Ignored ack: MessageAck {commandId = 2653540, responseRequired = false, ackType = 2, consumerId = ID:xxx-xx.xxx.xxx-59892-1501294728224-3:1:1:1, firstMessageId = ID:l-xxx.xxx.xx-58793-1501158447806-3:1:1:1:975174, lastMessageId = ID:l-XX.XX.XX-58793-1501158447806-3:1:1:1:975174, destination = queue://xxxxx, transactionId = null, messageCount = 1, poisonCause = null} | org.apache.activemq.broker.region.PrefetchSubscription | ActiveMQ Transport: tcp:///xx.xx.106.23:43678@56161
```

## 2. 采用线程池消费消息，线程池频繁的创建线程
现象：程序运行一段时间后，发现日志中消费消息的线程名，变的很大，也就是说线程在不停的创建和销毁。
首先看下面测试：
```
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    @Test
    public void testThreadCreate() throws InterruptedException {
        final ExecutorService service1 = Executors.newSingleThreadExecutor(new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                logger.info("create thread,current thread:{},poolNumber:{}", Thread.currentThread().getName(), poolNumber.get());
                return new Thread(r, "MQConsumer-Thread-" + poolNumber.getAndIncrement());
            }
        });

        for (int i = 0; i < 2; i++) {
            service1.execute(new Runnable() {
                @Override
                public void run() {
                    logger.info("enter service1 runnable.");
                    throw new RuntimeException("exception");
                }
            });
        }

        TimeUnit.SECONDS.sleep(100);
    }
```
Tip：如果采用线程池的时候不捕捉异常，会导致异常被抛到栈顶，JVM会干掉这个线程，看日志输出：
```
[08-04 10:17:11.143 [main] INFO com.qunar.sms.mq.MessageProcessorTest.newThread:291]-create thread,current thread:main,poolNumber:1
[08-04 10:17:11.156 [MQConsumer-Thread-1] INFO com.xxx.MessageProcessorTest.run:300]-enter service1 runnable.
[08-04 10:17:11.156 [MQConsumer-Thread-1] INFO com.xxx.MessageProcessorTest.newThread:291]-create thread,current thread:MQConsumer-Thread-1,poolNumber:2
[08-04 10:17:11.157 [MQConsumer-Thread-2] INFO com.xxx.MessageProcessorTest.run:300]-enter service1 runnable.
[08-04 10:17:11.157 [MQConsumer-Thread-2] INFO com.xxx.MessageProcessorTest.newThread:291]-create thread,current thread:MQConsumer-Thread-2,poolNumber:3
Exception in thread "MQConsumer-Thread-1" java.lang.RuntimeException: exception
	at com.qunar.sms.mq.MessageProcessorTest$7.run(MessageProcessorTest.java:301)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
	at java.lang.Thread.run(Thread.java:745)
Exception in thread "MQConsumer-Thread-2" java.lang.RuntimeException: exception
	at com.qunar.sms.mq.MessageProcessorTest$7.run(MessageProcessorTest.java:301)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
	at java.lang.Thread.run(Thread.java:745)
```
有意思的是，第一次main线程new了一个`MQConsumer-Thread-1`线程，执行中异常，抛到到栈顶，`MQConsumer-Thread-1`销毁，销毁之前创建了线程`MQConsumer-Thread-2`。
那么，想想如果线程池是`newScheduledThreadPool()`，定时执行一次任务异常后，下一次定时任务会执行吗？

结果是，肯定会执行，偶然的机会解决了我一直放在心中，并且没有想通的问题，有一次开启一个定时任务，其中一次发送了OOM，第二天查看日志，发现这个线程池还在正常的工作！
当时，没想通，为什么线程挂了，后面的任务还能继续执行呢？现在看来原因就是这么简单！
另外，在测试`newFixedThreadPool(5)`，线程池中包含多个线程池的情况下，一个异常抛到线程栈顶，该线程会先创建一个线程（而不是调用线程去创建，调用线程只会创建初始的5个线程），然后再被销毁！

下面是模拟，线上线程频繁创建的原因的代码：
```
    @Test
    public void testExceptionMessage() throws InterruptedException, JMSException {
        //首先生产N条消息
        pendingMessageToBroker("test.process.test4", PENDING_NUMBER);
        //确保所有消息已经推送到broker
        TimeUnit.SECONDS.sleep(1);

        //建立消费者
        Consumer consumer = Consumer.createDefault("test.process.test4", DEV_BETA_URL);
        final CountDownLatch latch = new CountDownLatch(1);
        consumer.receiveMessage(new MessageProcessor(false) {
            @Override
            public void processMessage(Message message) {
                if (message instanceof TextMessage) {
                    String text = getStringFromTextMessage(message);
                    if (StringUtils.equals("shutdown", text)) {
                        latch.countDown();
                    } else {
                        //模拟消费异常
                        throw new RuntimeException("mock unchecked exception.");
                    }
                }
            }
        });
        latch.await();
    }
```
由于在消费消息过程中，由于某些原因导致异常，没有catch住，线程被销毁，然后重新创建了线程！
如果不采用线程池，消费消息会由名为`ActiveMQ Seesion Task-x`的线程池来执行，它会catch住unchecked异常，并将异常信息和未能正确消费的消息返还给broker处理，
一般broker会redeliver到其他消费者，也可能直接丢弃到`ActiveMQ.DLQ`队列中，这个过程非常缓慢，会拖慢消费者，导致大量消息pending在broker！ 


