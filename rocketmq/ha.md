# [RocketMQ](component/rocketmq)

> 主从同步

## HAService

### DefaultHAService implements HAService

- start

    ```java
    @Override
    public void start() throws Exception {
        this.acceptSocketService.beginAccept();
        this.acceptSocketService.start();
        this.groupTransferService.start();
        this.haConnectionStateNotificationService.start();
        if (haClient != null) {
            this.haClient.start();
        }
    }
    ```

### AutoSwitchHAService extends DefaultHAService

## AcceptSocketService extends ServiceThread

用于Master监听Slave请求，单线程，基于NIO，为每个请求创建一个HAConnection。

### DefaultAcceptSocketService extends AcceptSocketService

监听成功后创建DefaultHAConnection。

## HAConnection

负责同步逻辑

### DefaultHAConnection implements HAConnection

## GroupTransferService extends ServiceThread

同步阻塞式的主从同步实现，单线程。

- putRequest

    消息写入磁盘后需要等数据传输到Slave完成，在CommitLog的handleDiskFlushAndHA方法中调用。

    ```java
    # DefaultHAService
    @Override
    public void putRequest(final CommitLog.GroupCommitRequest request) {
        this.groupTransferService.putRequest(request);
    }
    ```

- doWaitTransfer

    核心逻辑，用于处理主节点向从节点同步数据的请求。

    ```java
    private void doWaitTransfer() {
        // 如果当前有未处理的同步请求
        if (!this.requestsRead.isEmpty()) {
            for (CommitLog.GroupCommitRequest req : this.requestsRead) {
                boolean transferOK = false;

                // 获取请求的超时时间
                long deadLine = req.getDeadLine();
                // 判断是否需要所有同步状态集合中的节点都确认
                final boolean allAckInSyncStateSet = req.getAckNums() == MixAll.ALL_ACK_IN_SYNC_STATE_SET;

                // 循环等待直到超时或同步完成
                for (int i = 0; !transferOK && deadLine - System.nanoTime() > 0; i++) {
                    if (i > 0) {
                        // 释放同步锁，等待CommmitLog/HAConnection通知
                        this.notifyTransferObject.waitForRunning(1);
                    }

                    // 如果不需要所有节点确认，且只需要一个节点确认
                    if (!allAckInSyncStateSet && req.getAckNums() <= 1) {
                        transferOK = haService.getPush2SlaveMaxOffset().get() >= req.getNextOffset();
                        continue;
                    }

                    // 如果需要所有同步状态集合中的节点确认
                    if (allAckInSyncStateSet && this.haService instanceof AutoSwitchHAService) {
                        final AutoSwitchHAService autoSwitchHAService = (AutoSwitchHAService) this.haService;
                        final Set<Long> syncStateSet = autoSwitchHAService.getSyncStateSet();
                        if (syncStateSet.size() <= 1) {
                            // 如果只有主节点，直接认为同步成功
                            transferOK = true;
                            break;
                        }

                        // 遍历所有连接，检查是否满足同步条件
                        int ackNums = 1; // 包括主节点
                        for (HAConnection conn : haService.getConnectionList()) {
                            final AutoSwitchHAConnection autoSwitchHAConnection = (AutoSwitchHAConnection) conn;
                            if (syncStateSet.contains(autoSwitchHAConnection.getSlaveId()) &&
                                autoSwitchHAConnection.getSlaveAckOffset() >= req.getNextOffset()) {
                                ackNums++;
                            }
                            if (ackNums >= syncStateSet.size()) {
                                transferOK = true;
                                break;
                            }
                        }
                    } else {
                        int ackNums = 1; // 包括主节点
                        for (HAConnection conn : haService.getConnectionList()) {
                            // 需要确保每个HAConnection代表不同的从节点
                            if (conn.getSlaveAckOffset() >= req.getNextOffset()) {
                                ackNums++;
                            }
                            if (ackNums >= req.getAckNums()) {
                                transferOK = true;
                                break;
                            }
                        }
                    }
                }

                if (!transferOK) {
                    log.warn("transfer message to slave timeout, offset : {}, request acks: {}",
                        req.getNextOffset(), req.getAckNums());
                }

                // 唤醒等待的请求，并设置结果状态
                req.wakeupCustomer(transferOK ? PutMessageStatus.PUT_OK : PutMessageStatus.FLUSH_SLAVE_TIMEOUT);
            }

            // 清空已处理的请求
            this.requestsRead = new LinkedList<>();
        }
    }
    ```

- notifyTransferSome

    用于通知doWaitTransfer执行，HAConnection处理完读事件后，Master获取到最新的slaveAckOffset之后触发。

## HAClient

Slave启动，单线程，用于同步数据，基于HAConnection与Master交互。

### DefaultHAClient extends ServiceThread implements HAClient

使用状态机管理，READY状态尝试连接Master，Master侧AcceptSocketService会监听socket并为Slave创建HAConnection；TRANSFER主动从Master拉取同步数据；同步失败/长时间未收到同步数据会切换至READY，**Master侧需要防止重复创建HAConnection**。

```java
public void run() {
    log.info(this.getServiceName() + " service started");

    this.flowMonitor.start();

    while (!this.isStopped()) {
        try {
            switch (this.currentState) {
                case SHUTDOWN:
                    this.flowMonitor.shutdown(true);
                    return;
                case READY:
                    if (!this.connectMaster()) {
                        log.warn("HAClient connect to master {} failed", this.masterHaAddress.get());
                        this.waitForRunning(1000 * 5);
                    }
                    continue;
                case TRANSFER:
                    if (!transferFromMaster()) {
                        closeMasterAndWait();
                        continue;
                    }
                    break;
                default:
                    this.waitForRunning(1000 * 2);
                    continue;
            }
            long interval = this.defaultMessageStore.now() - this.lastReadTimestamp;
            if (interval > this.defaultMessageStore.getMessageStoreConfig().getHaHousekeepingInterval()) {
                log.warn("AutoRecoverHAClient, housekeeping, found this connection[" + this.masterHaAddress
                    + "] expired, " + interval);
                this.closeMaster();
                log.warn("AutoRecoverHAClient, master not response some time, so close connection");
            }
        } catch (Exception e) {
            log.warn(this.getServiceName() + " service has exception. ", e);
            this.closeMasterAndWait();
        }
    }

    this.flowMonitor.shutdown(true);
    log.info(this.getServiceName() + " service end");
}
```

- transferFromMaster

    定时向Master同步offset，Slave侧发起写操作，对应Master侧则为读事件；等待Master同步数据，监听读事件，对应Master侧发起写操作。

    ```java
    private boolean transferFromMaster() throws IOException {
        boolean result;
        if (this.isTimeToReportOffset()) {
            log.info("Slave report current offset {}", this.currentReportedOffset);
            result = this.reportSlaveMaxOffset(this.currentReportedOffset);
            if (!result) {
                return false;
            }
        }

        this.selector.select(1000);

        result = this.processReadEvent();
        if (!result) {
            return false;
        }

        return reportSlaveMaxOffsetPlus();
    }
    ```

- processReadEvent

    数据同步核心逻辑，使用缓冲区读取数据，每次只写入一条完整的消息。

    ```java
    private boolean processReadEvent() {
        int readSizeZeroTimes = 0;
        while (this.byteBufferRead.hasRemaining()) {
            try {
                // nio读取数据
                int readSize = this.socketChannel.read(this.byteBufferRead);
                if (readSize > 0) {
                    flowMonitor.addByteCountTransferred(readSize);
                    readSizeZeroTimes = 0;
                    boolean result = this.dispatchReadRequest();
                    if (!result) {
                        log.error("HAClient, dispatchReadRequest error");
                        return false;
                    }
                    lastReadTimestamp = System.currentTimeMillis();
                } else if (readSize == 0) {
                    // 连续读取超过3次未获取到数据则中断
                    if (++readSizeZeroTimes >= 3) {
                        break;
                    }
                } else {
                    log.info("HAClient, processReadEvent read socket < 0");
                    return false;
                }
            } catch (IOException e) {
                log.info("HAClient, processReadEvent read socket exception", e);
                return false;
            }
        }

        return true;
    }

    private boolean dispatchReadRequest() {
        int readSocketPos = this.byteBufferRead.position();

        while (true) {
            int diff = this.byteBufferRead.position() - this.dispatchPosition;
            if (diff >= DefaultHAConnection.TRANSFER_HEADER_SIZE) {
                // 获取Master发送的同步起始偏移量，由Slave上报给Master的偏移量决定。
                long masterPhyOffset = this.byteBufferRead.getLong(this.dispatchPosition);
                // 一条一条的处理数据
                int bodySize = this.byteBufferRead.getInt(this.dispatchPosition + 8);

                long slavePhyOffset = this.defaultMessageStore.getMaxPhyOffset();

                // 校验偏移量是否一致
                // 问题：如果slavePhyOffset至masterPhyOffset之间的数据已被删除会怎么样？
                if (slavePhyOffset != 0) {
                    if (slavePhyOffset != masterPhyOffset) {
                        log.error("master pushed offset not equal the max phy offset in slave, SLAVE: "
                            + slavePhyOffset + " MASTER: " + masterPhyOffset);
                        return false;
                    }
                }

                // 如果已经获取到一条完整的数据了则处理
                if (diff >= (DefaultHAConnection.TRANSFER_HEADER_SIZE + bodySize)) {
                    byte[] bodyData = byteBufferRead.array();
                    int dataStart = this.dispatchPosition + DefaultHAConnection.TRANSFER_HEADER_SIZE;

                    // 写入CommitLog
                    this.defaultMessageStore.appendToCommitLog(
                        masterPhyOffset, bodyData, dataStart, bodySize);

                    this.byteBufferRead.position(readSocketPos);
                    this.dispatchPosition += DefaultHAConnection.TRANSFER_HEADER_SIZE + bodySize;

                    // 上报当前最大的物理偏移量，并更新currentReportedOffset
                    if (!reportSlaveMaxOffsetPlus()) {
                        return false;
                    }

                    continue;
                }
            }

            // 如果未获取到一条完整的数据，则需要交换缓冲区。
            if (!this.byteBufferRead.hasRemaining()) {
                this.reallocateByteBuffer();
            }

            break;
        }

        return true;
    }
    ```
