---
title: 【ActiveMQ】消息唯一性的保证(十)
date: 2017/06/12 11:12:22
toc: false
list_number: false
categories:
- ActiveMQ
tags:
- ActiveMQ
---

前言：一次broker的failover，导致DLQ队列中多出了很多消息，这些消息都是由client端报duplicate message异常提交到DLQ队列中的。由此必须得深入client端的源码了解一下其机制。
1. 消息是如何保证不被重复消费？
2. 上述现象在client端是如何产生的？
## 一. 起因
### 1. 场景描述
现MQ的架构是`Master+ Slaver + Slaver`，底层持久化采用LevelDB，由于业务原因需要将其中一台Slaver服务器转移，具体操作很简单：1. stop old slaver mq；2. start new slaver mq。
在stop slaver后，奇怪的master发生了failover，原master权重30，竟然failover到权重为10的另外一台Slaver上了（权重根据机器性能设置的），好奇怪mq使用zk的选举策略是怎么做的。
先不探究这个failover，因为经验证数据并没有丢失，但是failover过程中DLQ队列中多出了2000多条消息，这两千多条消息都是记录的下面异常：
```
java.lang.Throwable: Suppressing duplicate delivery on connection, consumer ID:l-xxxx.xxx.xxx-41868-1510630408156-11:1:1:1
```
服务器也报了如下异常：
```
[2017-11-23 12:03:20.521 [ActiveMQ Session Task-2691] WARN o.a.activemq.ActiveMQMessageConsumer.dispatch:1457]-
ID:l-xxx.xx.xxx-41868-1510630408156-11:1:1:1 suppressing duplicate delivery on connection, 
poison acking: MessageDispatch {commandId = 0, responseRequired = false, 
consumerId = ID:l-xxx.xxx.xxx-41868-1510630408156-11:1:1:1, destination = queue://xxx.xxx.xxx.update, 
message = ActiveMQTextMessage {commandId = 46551652, responseRequired = true, 
messageId = ID:l-xxx.xx.xxx-39490-1509524021580-1:1:1:1:46551648, originalDestination = null, 
originalTransactionId = null, producerId = ID:l-xxx.xx.xx-39490-1509524021580-1:1:1:1, 
destination = queue://xxx.xxx.xxxx.update, transactionId = null, expiration = 0, 
timestamp = 1511409744482, arrival = 0, brokerInTime = 1511409744482, brokerOutTime = 1511409795129, 
correlationId = null, replyTo = null, persistent = true, type = null, priority = 4, groupID = null, 
groupSequence = 0, targetConsumerId = null, compressed = false, userID = null, 
content = org.apache.activemq.util.ByteSequence@7bcee2f8, marshalledProperties = null, 
dataStructure = null, redeliveryCounter = 0, size = 0, properties = null, readOnlyProperties = true, 
readOnlyBody = true, droppable = false, jmsXGroupFirstForConsumer = false, 
text = {"condition":{"xxx":"56201711231200538...:"xxx"}}}, redeliveryCounter = 0}
```
### 2. 结论
很简单：网络连接断开，broker进行failover或网络异常等情况发送后，连接恢复，此时broker会将没有ACK回来的消息重新发送给原来接受该消息的client，
此时client如果检测到这个是重复消息，那么会在接收这条消息时抛异常，并将其返回给broker放入DLQ队列中去.
No,No,No，没有结束，检测到是重复的消息抛异常？No,真实处理是：检测到重复消息后给broker发一个特殊ack（`posionAck`），
ack中携带消息`MessageDispatch`实体和封装的一个异常对象返回给服务器，服务器收到该ack后，才将这条消息丢到了DLQ中。
具体，且看下面的源码解析：
## 二. 过程分析
首先，所有的分析都不考虑Transaction的情况！
### 1. 重复消息处理流程
这个现象的所有过程都在`org.apache.activemq.ActiveMQMessageConsumer#dispatch()`方法中，此方法会被`ActiveMQ Session Task-x`线程循环重复调用去消费消息。
逻辑简化后如下：
```
synchronized (unconsumedMessages.getMutex() ) {
	if ( !unconsumedMessages.isClosed() )
	{
		if ( this.info.isBrowser() || !session.connection.isDuplicate( this, md.getMessage() ) )
		{
			if ( listener != null && unconsumedMessages.isRunning() )
			{
				ActiveMQMessage message = createActiveMQMessage( md );
				beforeMessageIsConsumed( md );
				try {
					boolean expired = isConsumerExpiryCheckEnabled() && message.isExpired();
					if ( !expired )
					{
						listener.onMessage( message );
					}
					afterMessageIsConsumed( md, expired );
				} else {
					/* deal with duplicate delivery */
					LOG.warn( "{} suppressing duplicate delivery on connection, poison acking: {}", getConsumerId(), md );
					posionAck( md, "Suppressing duplicate delivery on connection, consumer " + getConsumerId() );
				}
			}
		}
```
首先获取消息执行权，然后判断是否是重复调用的消息，如果不是则正常逻辑处理，如果是重复消息，则记录日志并给broker发送一个`posionAck`。
过程很简单，这里我是如何判断是重复消息的呢？
### 2. 重复消息判断逻辑
`org.apache.activemq.ConnectionAudit#isDuplicate()`方法是封装重复消息检查的逻辑：
```
    class ConnectionAudit {
        private LinkedHashMap<ActiveMQDestination, ActiveMQMessageAudit> destinations = new LRUCache<>(1000);
        synchronized boolean isDuplicate(ActiveMQDispatcher dispatcher, Message message) {
            if (checkForDuplicates && message != null) {
                ActiveMQDestination destination = message.getDestination();
                if (destination != null) {
                    ActiveMQMessageAudit audit = destinations.get(destination);
                    if (audit == null) {
                        audit = new ActiveMQMessageAudit(xx, xx);
                        destinations.put(destination, audit);
                    }
                    boolean result = audit.isDuplicate(message);
                    return result;
                }
            }
            return false;
        }
    }
```
没有什么比代码解释的更清楚，其中`LRUCache`是基于`LinkedHashMap`的简单缓存实现，这里将`ActiveMQDestination`和`ActiveMQMessageAudit`缓存起来。
`ActiveMQDestination`是MQ中destination对象，`ActiveMQMessageAudit`是具体的check duplicate对象。
下一步，研究`ActiveMQMessageAudit`的`isDuplicate`方法的实现：
```
    public boolean isDuplicate(final MessageId id) {
        boolean answer = false;
        if (id != null) {
            ProducerId pid = id.getProducerId();
            if (pid != null) {
                BitArrayBin bab = map.get(pid.toString());
                if (bab == null) {
                    bab = new BitArrayBin(64);
                    map.put(pid.toString(), bab);
                    modified = true;
                }
                answer = bab.setBit(id.getProducerSequenceId(), true);
            }
        }
        return answer;
    }
```
这里map定义：`new LRUCache<String, BitArrayBin>(0, 2048, 0.75f, true)` 也是一个LRU缓存，根据`ProducerId`作为key，`BitArrayBin`作为`value`。
重点：`BitArrayBin`，它是一个精心设计保存`producerSequenceId`的容器，这里`producerSequenceId`是每条消息的唯一序列号，只保存它是因为，前面有两层LRU缓存。
分别保存`destination`和`producerId`这样就唯一确定了一条消息，`producerId`采用bitmap实现判断消息是否已经消费过，即`BitArrayBin`实现。
`BitArrayBin`采用`LinkedList<BitArray>`保存`BitArray`，`BitArray`是一个数组，处理`producerSequenceId`后作为其数组的下标，boolean为value，true表示该位上有值（已经消费）！
Tip：两层LRU并不是二级缓存的意思，destinations存储的是每一个队列，map存储的是每一个队列中的所有Producer，真正判断是否重复发送的是`BitArrayBin`！
## 三. ACK的原理
ACK类型：autoACK，DupACK，ClientACK，IndividualACK。下面简单从源码的角度看看实现，这里只看consumer端client的情况（Producer端是一样的原理）：
`ActiveMQMessageConsumer`中通过`protected final LinkedList<MessageDispatch> deliveredMessages = new LinkedList<>();`保存已经发送的消息。
每条消息处理前后都会调用：`beforeMessageIsConsumed(md);`和`afterMessageIsConsumed(md, expired);`这两个方法。

`beforeMessageIsConsumed(md);`源码：
```
 private void beforeMessageIsConsumed(MessageDispatch md) throws JMSException {
        md.setDeliverySequenceId(session.getNextDeliveryId());
        lastDeliveredSequenceId = md.getMessage().getMessageId().getBrokerSequenceId();
        if (!isAutoAcknowledgeBatch()) {
            synchronized(deliveredMessages) {
                deliveredMessages.addFirst(md);
            }
            if (session.getTransacted()) {
                if (transactedIndividualAck) {
                    immediateIndividualTransactedAck(md);
                } else {
                    ackLater(md, MessageAck.DELIVERED_ACK_TYPE);
                }
            }
        }
    }
```
基本逻辑就是，除开`Topic`模式下`DupACK`情况，所有消息处理前先放入`deliveredMessages`链表的头中(而不是尾部)。如果是事务消息则xxx。

`afterMessageIsConsumed(md, expired);`源码：
```
    private void afterMessageIsConsumed(MessageDispatch md, boolean messageExpired) throws JMSException {
        //先判断unconsumedMessages有没有关闭，若关闭直接返回。
        if (messageExpired) {
            //如果该消息是过期消息，则返回该消息的ack（EXPIRED_ACK_TYPE）
        } else {
            if (session.getTransacted()) {
                // 事务消息，不做任务处理
            } else if (isAutoAcknowledgeEach()) {
                //如果是autoAck，走下面流程
                if (optimizeAcknowledge) {
                    //如果启用优化ack，则这里批量确认ACK逻辑
                } else {
                    MessageAck ack = makeAckForAllDeliveredMessages(MessageAck.STANDARD_ACK_TYPE);
                    if (ack != null) {
                        deliveredMessages.clear();
                        session.sendAck(ack);
                    }
                }
            } else if (isAutoAcknowledgeBatch()) {
                ackLater(md, MessageAck.STANDARD_ACK_TYPE);
            } else if (session.isClientAcknowledge() || session.isIndividualAcknowledge()) {
                boolean messageUnackedByConsumer = false;
                synchronized (deliveredMessages) {
                    messageUnackedByConsumer = deliveredMessages.contains(md);
                }
                if (messageUnackedByConsumer) {
                    ackLater(md, MessageAck.DELIVERED_ACK_TYPE);
                }
            } else {
                throw new IllegalStateException("Invalid session state.");
            }
        }
    }
```
如果是autoAck则`makeAckForAllDeliveredMessages`回复brokerack，`makeAckForAllDeliveredMessages`是将`deliveredMessages`链表中首尾的两个`MessageId`对象附带ACK返回。
- 因为这里是autoAck，所以正常情况下`deliveredMessages`中只有一条消息，注意，这里如果autoAck未能及时回复ack（比如网络异常，未获取到`deliveryingAcknowledgements`），`deliveredMessages`中也可能会有多条需要ack的消息，这时client会返回一条ack以确认全部未回复的ack！
- 如果是如果是批量ack的情况，则走`ackLater`逻辑，`ackLater`逻辑是如果未ack的message数量大于等于`PrefetchSize`一半时，就将这些消息以一条`MessageAck`的方式返回给broker，`MessageAck`中保存了首尾message的messageId。
- 至于clientAck和IndividualAck逻辑就不赘述了。

结论：所有待回复broker的ACK都保存在一个链表中，autoAck一般情况下每条消息回复一条ack，当网络异常或消息并发很高(`deliveryingAcknowledgements.compareAndSet(false, true))`失败)时，会多条消息一起ack！DupACK是当未ack消息到达prefetchSize的一半时以一条ack的方式，进行全部确认！


