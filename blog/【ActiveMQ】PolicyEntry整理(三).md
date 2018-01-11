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

**前言: **本篇是根据网上资料编辑整理出来的，主要在broker端配置topic和queue的分发策略，二者有很多的相似之处。
个人不建议在broker端配置topic和queue的分发策略，只有在极端特殊情况下，进行配置，通常使用默认配置即可。

## 一. topic 订阅/发布
1. 所有topic策略（policyEntry）
`<policyEntry topic=">" producerFlowControl="true" optimizedDispatch="true"  memoryLimit="16mb">`
其中，`>`是通配符，这里指所有Topic消息，`optimizedDispatch`优化策略，`memoryLimit`非持久化消息最大内存。
Tip：这些也可以通过Client端配置。

2. 限制策略（pendingMessageLimitStrategy）

    ```
    <pendingMessageLimitStrategy>
    <!-- 如果prefetchSize为100，则保留10 * 100条消息 -->
    <prefetchRatePendingMessageLimitStrategy multiplier="10"/>
    </pendingMessageLimitStrategy>
    ```
针对Slow Consumer，Topic，nondurable订阅者有效，当通道中有大量的消息积压时，broker可以保留的消息量。
为了防止Topic中有慢速消费者，导致整个通道消息积压。(对于Topic，一条消息只有所有的订阅者都消费才会被删除)
可选参数：
`ConstantPendingMessageLimitStrategy`: 保留固定条数的消息，如果消息量超过limit，将使用“MessageEvictionStrategy”移除消息。
`PrefetchRatePendingMessageLimitStrategy`: 保留prefetchSize倍数条消息。

3. 消息剔除策略（MessageEvictionStrategy）
    ```
    <messageEvictionStrategy>
          <OldestMessageWithLowestPriorityEvictionStrategy />
    </messageEvictionStrategy>
    ```
针对Slow Consumer，Topic，nondurable订阅者有效，当通道中有大量的消息积压时，PendingMessage的数量超过限制时，broker该如何剔除多余的消息。
当Topic接收到信息消息后，会将消息“Copy”给每个订阅者，在保存这个消息时(保存策略"PendingSubscriberMessageStoragePolicy")，
将会检测pendingMessages的数量是否超过限制(由"PendingMessageLimitStrategy"来检测)，如果超过限制，将会在pendingMessages中使用MessageEvicationStrategy移除多余的消息，
此后将新消息保存在PendingMessages中。
可选参数：
`OldestMessageEvictionStrategy`: 移除旧消息，默认策略。
`OldestMessageWithLowestPriorityEvictionStrategy`: 旧数据中权重较低的消息，将会被移除。
`UniquePropertyMessageEvictionStrategy`: 移除具有指定property的旧消息。开发者可以指定property的名称，从此属性值相同的消息列表中移除较旧的（根据消息的创建时间）。

4. 慢速消费者策略

    ```
    <slowConsumerStrategy>
          <abortSlowConsumerStrategy abortConnection="false"/>
    </slowConsumerStrategy>
    ```
Broker将如何处理慢消费者。Broker将会启动一个后台线程用来检测所有的慢速消费者，并定期关闭关闭它们。
abortConnection表示是否关闭连接，但不关闭底层链接。maxTimeSinceLastAck表示最后一个ACK距离现在的时间间隔阀值。

5. 转发策略 将消息转发给消费者的方式
    ```
    <dispatchPolicy>
         <strictOrderDispatchPolicy/>
    </dispatchPolicy>
    ```
`RoundRobinDispatchPolicy`: “轮询”，消息将依次发送给每个“订阅者”。“订阅者”列表默认按照订阅的先后顺序排列，
在转发消息时，对于匹配消息的第一个订阅者，将会被移动到“订阅者”列表的尾部，这也意味着“下一条”消息，将会较晚的转发给它。
`StrictOrderDispatchPolicy`: 严格有序，消息依次发送给每个订阅者，按照“订阅者”订阅的时间先后。它和RoundRobin最大的区别是，没有移动“订阅者”顺序的操作。
`PriorityDispatchPolicy`: 基于“property”权重对“订阅者”排序。它要求开发者首先需要对每个订阅者指定priority，默认每个consumer的权重都一样。
`SimpleDispatchPolicy`: 默认值，按照当前“订阅者”列表的顺序。其中PriorityDispatchPolicy是其子类。

6. 恢复策略 MQ重启恢复数据策略
    ```
    <subscriptionRecoveryPolicy>
    <!--恢复最近30分钟内的信息-->
    <timedSubscriptionRecoveryPolicy recoverDuration="1800000"/>
    </subscriptionRecoveryPolicy>
    ```
可选参数：
`FixedSizedSubscriptionRecoveryPolicy`: 保存一定size的消息，broker将为此Topic开辟定额的RAM用来保存最新的消息。使用maximumSize属性指定保存的size数量
`FixedCountSubscriptionRecoveryPolicy`: 保存一定条数的消息。 使用maximumSize属性指定保存的size数量
`LastImageSubscriptionRecoveryPolicy`: 只保留最新的一条数据
`QueryBasedSubscriptionRecoveryPolicy`: 符合置顶selector的消息都将被保存，具体能够“恢复”多少消息，由底层存储机制决定；比如对于非持久化消息，只要内存中还存在，则都可以恢复。
`TimedSubscriptionRecoveryPolicy`: 保留最近一段时间的消息。使用recoverDuration属性指定保存时间 单位毫秒
`NoSubscriptionRecoveryPolicy`: 关闭“恢复机制”。默认值。

7. "死信"策略 DLQ策略
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
Tip： 默认所有死信消息都会放入`ActiveMQ.DLQ`队列，

7. 非持久化消息策略
    ```
    <pendingSubscriberPolicy>
    <fileCursor/>
    </pendingSubscriberPolicy>
    ```
支持三种策略：storeCursor, vmCursor和fileCursor。

8. 持久化消息策略
    ```
    <pendingDurableSubscriberPolicy>
        <storeDurableSubscriberCursor/>
    </pendingDurableSubscriberPolicy>
    ```
支持三种策略：`storeDurableSubscriberCursor`, `vmDurableCursor`和 `fileDurableSubscriberCursor`

## 二. queue 生产/消费
1. 所有queue策略（policyEntry）
`<policyEntry queue=">" producerFlowControl="true" optimizedDispatch="true" memoryLimit="16mb">`
`<policyEntry queue=">" producerFlowControl="true" memoryLimit="4mb" queuePrefetch="1000" useCache="true">`
2. 与Topic相同的部分
    ```
    <pendingMessageLimitStrategy>
        <prefetchRatePendingMessageLimitStrategy multiplier="10"/>
    </pendingMessageLimitStrategy>
    <messageEvictionStrategy>
        <OldestMessageWithLowestPriorityEvictionStrategy />
    </messageEvictionStrategy>
    <slowConsumerStrategy>
        <abortSlowConsumerStrategy abortConnection="false"/>
    </slowConsumerStrategy>
    <dispatchPolicy>
        <strictOrderDispatchPolicy/>
    </dispatchPolicy>
    <subscriptionRecoveryPolicy>
        <timedSubscriptionRecoveryPolicy recoverDuration="1800000"/>
    </subscriptionRecoveryPolicy>
    <deadLetterStrategy>
        <individualDeadLetterStrategy  queuePrefix="DLQ.QUEUE." useQueueForQueueMessages="true"/>
    </deadLetterStrategy>
    ```

3. 持久化和非持久化消息
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
AcitveMQ提供了几个的Cursor机制，它就是用来保存Pending Messages。
`vmQueueCursor`: 将待转发消息保存在额外的内存(JVM linkeList)的存储结构中。是“非持久化消息”的默认设置，如果Broker不支持Persistent，它是任何类型消息的默认设置。有OOM风险。
`fileQueueCursor`: 将消息保存到临时文件中。文件存储方式有broker的tempDataStore属性决定。是“持久化消息”的默认设置。
`storeCursor`: “综合”设置，对于非持久化消息，将采用vmQueueCursor存储，对于持久化消息采用
`fileQueueCursor`：这是强烈推荐的策略，也是效率最好的策略。

Tip：具体配置方法查看官方文档。


# 参考
1. http://activemq.apache.org/per-destination-policies.html
2. http://activemq.apache.org/dispatch-policies.html
3. https://my.oschina.net/coderedrain/blog/724943?utm_source=tuicool&utm_medium=referral
4. http://shift-alt-ctrl.iteye.com/blog/2061859