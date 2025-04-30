# [RocketMQ](/blog/component/rocketmq)

> 消息轨迹

## 使用建议

建议使用一个单独的broker专门用来单独承载消息轨迹的流量，即只在一个broker上开启traceTopicEnable。

## 客户端

客户端开启enableMsgTrace后，每个消费者和生产者（目前只支持DefaultMQProducer/DefaultMQPushConsumer/DefaultLitePullConsumer）启动时都会创建一个AsyncTraceDispatcher用于发送消息轨迹，并针对不同的TraceType注册对应的Hook。

```java
# DefaultMQPushConsumer
@Override
public void start() throws MQClientException {
    setConsumerGroup(NamespaceUtil.wrapNamespace(this.getNamespace(), this.consumerGroup));
    this.defaultMQPushConsumerImpl.start();
    if (enableTrace) {
        try {
            AsyncTraceDispatcher dispatcher = new AsyncTraceDispatcher(consumerGroup, TraceDispatcher.Type.CONSUME, getTraceMsgBatchNum(), traceTopic, rpcHook);
            dispatcher.setHostConsumer(this.defaultMQPushConsumerImpl);
            dispatcher.setNamespaceV2(namespaceV2);
            traceDispatcher = dispatcher;
            this.defaultMQPushConsumerImpl.registerConsumeMessageHook(new ConsumeMessageTraceHookImpl(traceDispatcher));
        } catch (Throwable e) {
            log.error("system mqtrace hook init failed ,maybe can't send msg trace data");
        }
    }
    if (null != traceDispatcher) {
        if (traceDispatcher instanceof AsyncTraceDispatcher) {
            ((AsyncTraceDispatcher) traceDispatcher).getTraceProducer().setUseTLS(isUseTLS());
        }
        try {
            traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
        } catch (MQClientException e) {
            log.warn("trace dispatcher start failed ", e);
        }
    }
}
```

### AsyncTraceDispatcher implements TraceDispatcher

默认为异步批量发送，使用**独立的**Producer发送轨迹消息。

- append

    ```java
    @Override
    public boolean append(final Object ctx) {
        boolean result = traceContextQueue.offer((TraceContext) ctx);
        if (!result) {
            log.info("buffer full" + discardCount.incrementAndGet() + " ,context is " + ctx);
        }
        return result;
    }
    ```

    在Hook中将TraceContext提交给TraceDispatcher，直接提交到阻塞队列中，之后通过flushTraceContext()提交到traceExecutor异步发送。

- flushTraceContext

    ```java
    private void flushTraceContext(boolean forceFlush) throws InterruptedException {
        List<TraceContext> contextList = new ArrayList<>(batchNum);
        int size = traceContextQueue.size();
        if (size != 0) {
            // 批量提交异步发送任务，或定时提交。
            if (forceFlush || size >= batchNum || System.currentTimeMillis() - lastFlushTime > flushTraceInterval) {
                // Q: 这里可以直接使用drainTo()
                for (int i = 0; i < batchNum; i++) {
                    TraceContext context = traceContextQueue.poll();
                    if (context != null) {
                        contextList.add(context);
                    } else {
                        break;
                    }
                }
                asyncSendTraceMessage(contextList);
                return;
            }
        }
        Thread.sleep(5);
    }

    private void asyncSendTraceMessage(List<TraceContext> contextList) {
        AsyncDataSendTask request = new AsyncDataSendTask(contextList);
        traceExecutor.submit(request);
        lastFlushTime = System.currentTimeMillis();
    }
    ```

    shutdown()和ShutDownHook和会执行forceFlush，后台会单线程在AsyncRunnable中一直尝试flush。

#### AsyncDataSendTask

异步发送消息轨迹任务。

```java
private void sendTraceDataByMQ(Set<String> keySet, final String data, String traceTopic) {
    final Message message = new Message(traceTopic, data.getBytes(StandardCharsets.UTF_8));
    // Keyset of message trace includes msgId of or original message
    message.setKeys(keySet);
    try {
        Set<String> traceBrokerSet = tryGetMessageQueueBrokerSet(traceProducer.getDefaultMQProducerImpl(), traceTopic);
        SendCallback callback = new SendCallback() {
            @Override
            public void onSuccess(SendResult sendResult) {

            }

            @Override
            public void onException(Throwable e) {
                log.error("send trace data failed, the traceData is {}", data, e);
            }
        };
        if (traceBrokerSet.isEmpty()) {
            // No cross set
            traceProducer.send(message, callback, 5000);
        } else {
            traceProducer.send(message, new MessageQueueSelector() {
                @Override
                public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                    Set<String> brokerSet = (Set<String>) arg;
                    List<MessageQueue> filterMqs = new ArrayList<>();
                    for (MessageQueue queue : mqs) {
                        if (brokerSet.contains(queue.getBrokerName())) {
                            filterMqs.add(queue);
                        }
                    }
                    int index = sendWhichQueue.incrementAndGet();
                    int pos = index % filterMqs.size();
                    return filterMqs.get(pos);
                }
            }, traceBrokerSet, callback);
        }

    } catch (Exception e) {
        log.error("send trace data failed, the traceData is {}", data, e);
    }
}

private Set<String> tryGetMessageQueueBrokerSet(DefaultMQProducerImpl producer, String topic) {
    Set<String> brokerSet = new HashSet<>();
    TopicPublishInfo topicPublishInfo = producer.getTopicPublishInfoTable().get(topic);
    if (null == topicPublishInfo || !topicPublishInfo.ok()) {
        producer.getTopicPublishInfoTable().putIfAbsent(topic, new TopicPublishInfo());
        producer.getMqClientFactory().updateTopicRouteInfoFromNameServer(topic);
        topicPublishInfo = producer.getTopicPublishInfoTable().get(topic);
    }
    if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
        for (MessageQueue queue : topicPublishInfo.getMessageQueueList()) {
            brokerSet.add(queue.getBrokerName());
        }
    }
    return brokerSet;
}
```

### TraceTransferBean

消息轨迹数据结构
