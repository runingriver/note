---
title: 【ActiveMQ】PolicyEntry整理(三)
date: 2017/06/12 11:12:22
toc: false
list_number: false
categories:
- ActiveMQ
tags:
- ActiveMQ
---

**前言: **本篇是根据相关资料和MQ运维过程中编辑整理出来的，主要在broker端配置topic和queue的相关策略，二者有很多的相似之处。
个人不建议在broker端配置topic和queue的分发策略，只有在特殊情况下或保护broker稳定运行前提下，进行配置，通常使用默认配置即可。

**activemq是我见过的最烂的官方文档有没有！**

# 一. 相关概念
## Message group，durable subscriber/no-durable subscriber
对应ActiveMq的Topic而言，有两点：1.Message group;2.durable subscriber/no-durable subscriber；
- Message Group是指在发送端对某类消息进行标识分组，消费者可以针对Group获取消息。
Tip：注意区别Kafka的Group，kafka Group是将消费者分组，实现消息的负责均衡或类似queue的功能，分到同一组的消费者共同消费所有消息，即一条消息只会发给Group中的一个消费者。

- durable subscriber表示client消费者与broker断开后重连后能接收到，断开期间publisher的所有消息，但是第一次与broker建立连接告诉broker自己是durable subscriber时，broker并不会把之前的所有消息发给这个subscriber。相比no-durable subscriber只会收到建立连接后publisher发给broker的消息！
Tip：注意与持久化和非持久化消息的区别，持久化表示消息到达broker后是否保存到磁盘上！

## dispatch queue，pending cursor，message store
默认情况下，消息到来后，执行两个操作：保存到`message store`(磁盘)（仅针对persistence消息），放入`dispatch queue`(内存)中。
如果生产者生产消息的速度大于消费者消费的速度，那么broker在将消息放入`dispatch queue`操作时，`dispatch queue`中积压的消息会越来越多；
此时broker会启动Cursor游标策略，`dispatch queue`中积压的消息称作`pending Message`。游标策略其实就是开辟一块空间存储积压信息，解决生产者消费者速度不均衡时引入的一个缓冲区。而这个存储空间可以在内存中：`vmCursor`，也可以在磁盘中：`storeCursor`，也可以同时在内存和磁盘中：`fileCursor`。

# 二. 配置

## 从一个示例出发
下面配置了destinationPolicy的相关规则：
```
<broker>
    <destinationPolicy>
        <policyMap>
            <policyEntries>
                <policyEntry topic="FOO.>">
                    <!-- 仅topic且no-durable，对于slow consumer的消息保留策略 -->
                    <pendingMessageLimitStrategy>
                        <constantPendingMessageLimitStrategy limit="1000"/>
                    </pendingMessageLimitStrategy>
                    <!-- 消息剔除策略 -->
                    <messageEvictionStrategy>
                        <oldestMessageWithLowestPriorityEvictionStrategy/>
                    </messageEvictionStrategy>
                    <!--仅topic且no-durable，存在slow consumer时pending消息策略-->
                    <pendingSubscriberPolicy>
                        <vmCursor/>
                    </pendingSubscriberPolicy>
                    <!--仅topic且durable，存在slow consumer时pending消息策略-->
                    <pendingDurableSubscriberPolicy>
                        <vmDurableCursor/>
                    </pendingDurableSubscriberPolicy>

                    <!-- 仅topic消息分发策略 -->
                    <dispatchPolicy>
                        <RoundRobinDispatchPolicy/>
                    </dispatchPolicy>

                    <!-- no-durable subscriber上线后可以追溯的消息策略 -->
                    <subscriptionRecoveryPolicy>
                        <!-- 1 minute -->
                        <timedSubscriptionRecoveryPolicy recoverDuration="60000"/>
                    </subscriptionRecoveryPolicy>
                </policyEntry>

                <policyEntry queue="ORDERS.>">
                    <dispatchPolicy>
                        <strictOrderDispatchPolicy/>
                    </dispatchPolicy>
                </policyEntry>
            </policyEntries>
        </policyMap>
    </destinationPolicy>
</broker>
```

## policyEntry

一个policyEntry代表对一类队列的策略，这个我们也可以在client端配置，通常在broker中policyEntry上可以配置三个参数：
1. `producerFlowControl`，生产者流控制策略，触发条件是：当broker的RAM或磁盘到达设置的极限时，`ture`表示broker通过`延迟发送ACK`和`ResourceAllocationException返回异常`到生产者断的方法控制生产者的发生消息速度，false表示如果RAM满了则将消息存到磁盘，如果磁盘也满了则block住生产者直到磁盘有空间。

2. `optimizedDispatch`，**仅queue**，默认false：对于每个queue不使用一个独立的线程来dispatch消息;true：每个queue分配一个线程专用于dispatch消息到consumer。

3. `memoryLimit`，设置保存在内存中的数据量，默认没有限制，上限受限于`systemUsage`的设置，通常是用于cursor

eg：
`<policyEntry queue=">" producerFlowControl="true" optimizedDispatch="true"  memoryLimit="16mb">`
其中，`>`是通配符，这里指所有queue队列，`optimizedDispatch`每个队列分配一个独立线程分发消息，`memoryLimit`每个队列的最大内存使用16M。

Tip：这些也可以通过Client端配置，可以参考：http://activemq.apache.org/per-destination-policies.html （详细说明了哪些是topic哪些是queue独有，共有的属性）



### 1. 限制策略（pendingMessageLimitStrategy）
**针对Slow Consumer，Topic，no-durable订阅者有效**，当通道中有大量的消息积压时，broker可以pending在RAM中的消息量。
- 为了防止Topic中有慢速消费者，导致整个通道消息积压，从而导致broker减缓producer的发送速度,从而影响正常的消费者消费速度。
- 对于Topic，一条消息只有所有的订阅者都消费才会被删除
- 超过limit的消息会采用`MessageEvictionStrategy`策略进行剔除。

可选参数：
`<constantPendingMessageLimitStrategy limit="1000"/>`: 内存保留1000条消息。
`<prefetchRatePendingMessageLimitStrategy multiplier="2.5"/>`: 如果prefetchSize为1000，则保留2.5 * 1000条消息

参考：http://activemq.apache.org/slow-consumer-handling.html
参考：http://activemq.apache.org/slow-consumer-handling.html

### 2. 消息剔除策略（MessageEvictionStrategy）
**针对Slow Consumer，Topic，nondurable订阅者有效**，PendingMessage的数量超过限制时，broker该如何剔除多余的消息。
当一条Topic消息达到broker后，两步操作：1.若是持久化消息，先保存到磁盘；2.放入Dispatch queue，此时`pendingMessageLimitStrategy`策略会检测pending Message是否到达limit，如果是则采用`MessageEvictionStrategy`策略将内存中（dispatch queue）的消息剔除到`cursor`中。这个cursor策略由`pendingSubscriberPolicy`和`pendingDurableSubscriberPolicy`控制。

可选参数：
`<oldestMessageEvictionStrategy/>`: 移除旧消息，默认策略。
`<oldestMessageWithLowestPriorityEvictionStrategy/>`: 旧数据中权重较低的消息，将会被移除。权重在生产者端设置（`producer.setPriority(0~9)`，默认4）
`<uniquePropertyMessageEvictionStrategy propertyName="test" />`: 移除具有指定property的旧消息。此属性值在生产者端设置，并移除较旧的。

### 3. Topic游标策略 pendingSubscriberPolicy和pendingDurableSubscriberPolicy
- **pendingSubscriberPolicy**：是针对nondurable subscriber，三种策略：`storeCursor`, `vmCursor`和`fileCursor`。
- **pendingDurableSubscriberPolicy**：是针对durable subscriber，三种策略：`storeDurableSubscriberCursor`, `vmDurableCursor`和 `fileDurableSubscriberCursor`。
都默认是store，一般都使用默认，每个具体含义参见：`http://activemq.apache.org/message-cursors.html`

### 4. Queue游标策略 pendingQueuePolicy
同上，也有三种策略：`fileQueueCursor` ， `storeCursor` 和 `vmQueueCursor`

可选参数：
`vmQueueCursor`: 将待转发消息保存在额外的内存(JVM linkeList)的存储结构中。是“非持久化消息”的默认设置，如果Broker不支持Persistent，它是任何类型消息的默认设置。有OOM风险。
`fileQueueCursor`: 将消息保存到临时文件中。文件存储方式有broker的tempDataStore属性决定。是“持久化消息”的默认设置。
`storeCursor`: “综合”设置，对于非持久化消息，将采用vmQueueCursor存储，对于持久化消息采用
`fileQueueCursor`：这是强烈推荐的策略，也是效率最好的策略。

eg：对持久化和非持久化进行个性化配置：
持久化和非持久化采用vmQueueCursor：
    ```
    <pendingQueuePolicy>
    <vmQueueCursor/>
    </pendingQueuePolicy>
    ```
非持久化采用fileQueueCursor：
    ```
    <pendingQueuePolicy>
    <storeCursor>
    <nonPersistent>
    <fileQueueCursor/>
    </nonPersistent>
    </storeCursor>
    </pendingQueuePolicy>
    ```


**Tip: 无论是pendingQueuePolicy还是pendingSubscriberPolicy和pendingDurableSubscriberPolicy，都是解决由于消费者速度相对慢情景下，RAM的Dispatch queue无法保存所有积压的消息引入的中间层，解决速度不均衡导致的消息积压问题的方案。且与消息是否是persistent无关。**

### 5. 转发策略dispatchPolicy
**仅针对Topic有效**，消息发送给多个subscriber的顺序问题：

可选参数：
`<RoundRobinDispatchPolicy/>`: 轮询，消息将依次发送给每个订阅者
`<strictOrderDispatchPolicy/>`: 严格有序，消息依次发送给每个订阅者，按照订阅的时间先后。
`<PriorityDispatchPolicy/>`: 基于`property`，subscriber初始化时可以指定priority，默认每个consumer的priority相同。
`<SimpleDispatchPolicy/>`: 默认值
`<NoSubscriptionRecoveryPolicy/>`: 关闭恢复机制。默认值，即不发送订阅开始之前的消息。
strictOrderDispatchPolicy的解释参考：`http://activemq.apache.org/total-ordering.html`

### 6. 消息恢复策略 subscriptionRecoveryPolicy
**针对Topic，no-durable订阅者有效**，表示no-durable上线后broker将其上线前的消息发送给subscriber的策略：
默认情况下，subscriber只能获取建立连接完成之后的消息，如果`Retroactive=true`（在publisher端设置），那么订阅者就可以获取其创建之前的消息列表。
subscriptionRecoveryPolicy就是用来控制“retroactive”的消息量的，也即broker保留多少已经消费了的消息。

可选参数：
`<timedSubscriptionRecoveryPolicy recoverDuration="60000"/>`：保留1分钟内的消息。
`<fixedSizedSubscriptionRecoveryPolicy maximumSize="1024"/>`：保留1Kb消息。
`<fixedCountSubscriptionRecoveryPolicy maximumSize="100"/>`：保留100条。
`<LastImageSubscriptionRecoveryPolicy/>`: 只保留最新的一条数据

### 7. 慢速消费者策略

    ```
    <slowConsumerStrategy>
          <abortSlowConsumerStrategy abortConnection="false" maxTimeSinceLastAck="30000"/>
    </slowConsumerStrategy>
    ```
Broker将如何处理慢消费者。Broker将会启动一个后台线程用来检测所有的慢速消费者，并定期关闭关闭它们。
`abortConnection`表示是否关闭连接，但不关闭底层链接。`maxTimeSinceLastAck`表示最后一个ACK距离现在的时间间隔阀值。

### 8. "死信"策略 DLQ策略
    ```
    <deadLetterStrategy>
        <individualDeadLetterStrategy  queuePrefix="DLQ." useQueueForQueueMessages="true"/>
        <sharedDeadLetterStrategy deadLetterQueue="DLQ-QUEUE"/> 
    </deadLetterStrategy>
    ```
`IndividualDeadLetterStrategy`: 把DeadLetter放入各自的死信通道中,queuePrefix自定义死信前缀，useQueueForQueueMessages使用队列保存死信，还有一个属性为“useQueueForTopicMessages”，此值表示是否将Topic的DeadLetter保存在Queue中，默认为true。 
`SharedDeadLetterStrategy`: 将所有的DeadLetter保存在一个共享的队列中，这是ActiveMQ broker端默认的策略。共享队列默认为“ActiveMQ.DLQ”，可以通过“deadLetterQueue”属性来设定。还有2个很重要的可选参数，“processExpired”表示是否将过期消息放入死信队列，默认为true；“processNonPersistent”表示是否将“非持久化”消息放入死信队列，默认为false。
`DiscardingDeadLetterStrateg`y: broker将直接抛弃DeadLeatter。如果开发者不需要关心DeadLetter，可以使用此策略。
AcitveMQ提供了一个便捷的插件：DiscardingDLQBrokerPlugin，来抛弃DeadLetter：
指定队列的所有消息，全部，正则匹配：
    ```
    <discardingDLQBrokerPlugin dropOnly="MY.EXAMPLE.TOPIC.29 MY.EXAMPLE.QUEUE.87" reportInterval="1000" />
    <discardingDLQBrokerPlugin dropAll="true" dropTemporaryTopics="true" dropTemporaryQueues="true" />
    <discardingDLQBrokerPlugin dropOnly="MY.EXAMPLE.TOPIC.[0-9]{3} MY.EXAMPLE.QUEUE.[0-9]{3}"reportInterval="3000" />
    ```
Tip： 默认所有死信消息都会放入`ActiveMQ.DLQ`队列

# 三. policyEntry相关问题及配置

## 持久订阅者`Durable Subscribers`pending太多消息在broker的问题
如果持久订阅者下线了很长一段时间，或者持久订阅者不再订阅某topic，默认情况下，由于broker要将没有被持久订阅者消费的所有消息保存起来，以便在其上线后发给它；
然而，如果这些消息大量积压在broker，会导致broker的磁盘内存等资源耗尽，且policyEntry没有相关配置，另外，虽然我们可以通过管理页面进行处理，但不一定能及时处理，官方提供两种办法对这种情况进行处理：

### 1. 过期消息
我们可以在client端将指定destination的消息设置一个过期属性（`setTimeToLive(timeToLive)`）；
然后在指定队列中配置`<policyEntry topic=">" expireMessagesPeriod="300000"/>`，这里在broker配置对所有topic类型消息，每5分钟检查一次，将过期的消息从broker删除！

### 2. 移除不活动的持久订阅者
ActiveMq提供两个参数，来配置移除策略：
`offlineDurableSubscriberTimeout`，默认`-1`不移除，单位毫秒，表示`Durable Subscriber`不活动多久后，就移除这个Subscriber。
`offlineDurableSubscriberTaskSchedule`，默认`300000`，单位毫秒，即默认5分钟检查一次，表示broker每间隔多久去检查一次。
eg：
`<broker name="localhost" offlineDurableSubscriberTimeout="86400000" offlineDurableSubscriberTaskSchedule="3600000">`
表示每隔1小时去检测一次，将下线1天的持久订阅者删除掉！



参考：http://activemq.apache.org/manage-durable-subscribers.html


Tip：具体配置方法查看官方文档，每个参数可以在http://activemq.apache.org/xml-reference.html中找到。


# 参考
1. http://activemq.apache.org/per-destination-policies.html
2. http://activemq.apache.org/dispatch-policies.html
3. https://my.oschina.net/coderedrain/blog/724943?utm_source=tuicool&utm_medium=referral
4. http://shift-alt-ctrl.iteye.com/blog/2061859
5. **http://activemq.apache.org/xml-reference.html**
6. http://activemq.apache.org/xml-configuration.html
7. http://activemq.apache.org/slow-consumer-handling.html