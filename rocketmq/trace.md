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
// 批量合并发送，keySet为多个TraceTransferBean的transKey的聚合，data多个TraceTransferBean的transData的拼接。
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
        if (traceBrokerSet.isEmpty()) { // 发送消息，如果topic还未创建
            // No cross set
            traceProducer.send(message, callback, 5000);
        } else { // 发送消息，topic已存在则轮询
            traceProducer.send(message, new MessageQueueSelector() {
                @Override
                public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                    Set<String> brokerSet = (Set<String>) arg;
                    List<MessageQueue> filterMqs = new ArrayList<>();
                    // Q：为什么还需要过滤一遍mq？
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

```java
private String transData;
private Set<String> transKey = new HashSet<>();

// TraceDataEncoder
public static TraceTransferBean encoderFromContextBean(TraceContext ctx) {
    ...
    transferBean.setTransData(sb.toString());
    // traceBeans固定只有一个元素
    for (TraceBean bean : ctx.getTraceBeans()) {
        transferBean.getTransKey().add(bean.getMsgId());
        // 添加原消息的key，如果原消息是批量发送的，则为原消息的key组合。
        if (bean.getKeys() != null && bean.getKeys().length() > 0) {
            String[] keys = bean.getKeys().split(MessageConst.KEY_SEPARATOR);
            transferBean.getTransKey().addAll(Arrays.asList(keys));
        }
    }
}
```

### SendMessageTraceHookImpl implements SendMessageHook

生产消息的钩子类。

```java
@Override
public void sendMessageBefore(SendMessageContext context) {
    if (context == null || context.getMessage().getTopic().startsWith(((AsyncTraceDispatcher) localDispatcher).getTraceTopicName())) {
        return;
    }
    //build the context content of TraceContext
    TraceContext traceContext = new TraceContext();
    traceContext.setTraceBeans(new ArrayList<>(1));
    context.setMqTraceContext(traceContext);
    traceContext.setTraceType(TraceType.Pub);
    traceContext.setGroupName(NamespaceUtil.withoutNamespace(context.getProducerGroup()));
    //build the data bean object of message trace
    TraceBean traceBean = new TraceBean();
    traceBean.setTopic(NamespaceUtil.withoutNamespace(context.getMessage().getTopic()));
    // Q：如果原消息是批量发送的，则为MessageBatch，会无法获取到tags和keys。
    traceBean.setTags(context.getMessage().getTags());
    traceBean.setKeys(context.getMessage().getKeys());
    traceBean.setStoreHost(context.getBrokerAddr());
    int bodyLength = null == context.getMessage().getBody() ? 0 : context.getMessage().getBody().length;
    traceBean.setBodyLength(bodyLength);
    traceBean.setMsgType(context.getMsgType());
    traceContext.getTraceBeans().add(traceBean);
}

@Override
public void sendMessageAfter(SendMessageContext context) {
    ...
    if (context.getSendResult().getRegionId() == null
        || !context.getSendResult().isTraceOn()) {
        // 是否发送还会判断服务端的traceOn开关
        return;
    }
    TraceContext traceContext = (TraceContext) context.getMqTraceContext();
    TraceBean traceBean = traceContext.getTraceBeans().get(0);
    int costTime = (int) ((System.currentTimeMillis() - traceContext.getTimeStamp()) / traceContext.getTraceBeans().size());
    traceContext.setCostTime(costTime);
    if (context.getSendResult().getSendStatus().equals(SendStatus.SEND_OK)) {
        traceContext.setSuccess(true);
    } else {
        traceContext.setSuccess(false);
    }
    traceContext.setRegionId(context.getSendResult().getRegionId());
    // 如果是批量发送，msgId为多个msgId的拼接，offsetMsgId为多个offsetMsgId的拼接。
    traceBean.setMsgId(context.getSendResult().getMsgId());
    traceBean.setOffsetMsgId(context.getSendResult().getOffsetMsgId());
    traceBean.setStoreTime(traceContext.getTimeStamp() + costTime / 2);
    // 提交到TraceDispatcher
    localDispatcher.append(traceContext);
}
```

### ConsumeMessageTraceHookImpl

消费消息的消息轨迹会在提交给消费线程消费前发送SubBefore记录，消费完成后发送SubAfter记录。

```java
@Override
public void consumeMessageBefore(ConsumeMessageContext context) {
    if (context == null || context.getMsgList() == null || context.getMsgList().isEmpty()) {
        return;
    }
    TraceContext traceContext = new TraceContext();
    context.setMqTraceContext(traceContext);
    traceContext.setTraceType(TraceType.SubBefore);//
    traceContext.setGroupName(NamespaceUtil.withoutNamespace(context.getConsumerGroup()));//
    List<TraceBean> beans = new ArrayList<>();
    for (MessageExt msg : context.getMsgList()) {
        if (msg == null) {
            continue;
        }
        String regionId = msg.getProperty(MessageConst.PROPERTY_MSG_REGION);
        String traceOn = msg.getProperty(MessageConst.PROPERTY_TRACE_SWITCH);
        if (traceOn != null && traceOn.equals("false")) {
            // 是否发送还会判断服务端的traceOn开关
            continue;
        }
        TraceBean traceBean = new TraceBean();
        traceBean.setTopic(NamespaceUtil.withoutNamespace(msg.getTopic()));//
        traceBean.setMsgId(msg.getMsgId());//
        traceBean.setTags(msg.getTags());//
        traceBean.setKeys(msg.getKeys());//
        traceBean.setStoreTime(msg.getStoreTimestamp());//
        traceBean.setBodyLength(msg.getStoreSize());//
        traceBean.setRetryTimes(msg.getReconsumeTimes());//
        traceContext.setRegionId(regionId);//
        beans.add(traceBean);
    }
    if (beans.size() > 0) {
        traceContext.setTraceBeans(beans);
        traceContext.setTimeStamp(System.currentTimeMillis());
        localDispatcher.append(traceContext);
    }
}

@Override
public void consumeMessageAfter(ConsumeMessageContext context) {
    if (context == null || context.getMsgList() == null || context.getMsgList().isEmpty()) {
        return;
    }
    TraceContext subBeforeContext = (TraceContext) context.getMqTraceContext();

    if (subBeforeContext.getTraceBeans() == null || subBeforeContext.getTraceBeans().size() < 1) {
        // If subBefore bean is null ,skip it
        return;
    }
    TraceContext subAfterContext = new TraceContext();
    subAfterContext.setTraceType(TraceType.SubAfter);//
    subAfterContext.setRegionId(subBeforeContext.getRegionId());//
    subAfterContext.setGroupName(NamespaceUtil.withoutNamespace(subBeforeContext.getGroupName()));//
    subAfterContext.setRequestId(subBeforeContext.getRequestId());//
    subAfterContext.setAccessChannel(context.getAccessChannel());
    subAfterContext.setSuccess(context.isSuccess());// 是否消费成功

    // 批量消费消息，costTime为总消费时间/消息条数。
    int costTime = (int) ((System.currentTimeMillis() - subBeforeContext.getTimeStamp()) / context.getMsgList().size());
    subAfterContext.setCostTime(costTime);//
    subAfterContext.setTraceBeans(subBeforeContext.getTraceBeans());
    Map<String, String> props = context.getProps();
    if (props != null) {
        String contextType = props.get(MixAll.CONSUME_CONTEXT_TYPE);
        if (contextType != null) {
            // 设置具体消费返回结果
            subAfterContext.setContextCode(ConsumeReturnType.valueOf(contextType).ordinal());
        }
    }
    localDispatcher.append(subAfterContext);
}
```
