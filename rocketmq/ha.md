# [RocketMQ](/blog/component/rocketmq)

> 主从同步
>> 最基础的主从模式，仅支持主从同步，并不支持主从切换。

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

## AcceptSocketService extends ServiceThread

用于Master监听Slave请求，单线程，基于NIO，为每个请求创建一个HAConnection。

### DefaultAcceptSocketService extends AcceptSocketService

监听成功后创建DefaultHAConnection。

## HAConnection

负责同步逻辑，接受到HAClient的连接请求后创建，包括一个单线程读服务和一个单线程写服务。

### DefaultHAConnection implements HAConnection

### ReadSocketService

处理客户端同步进度的请求，最多等待一秒执行一次，并包括心跳检测，超时未收到客户端请求会关闭Connection。

- processReadEvent

    核心逻辑，处理HAClient的请求，更新从服务的同步进度，并通知等待中的写请求。

    ```java
    private boolean processReadEvent() {
            int readSizeZeroTimes = 0;

            // 缓冲区使用完成，其实可以直接调用clear()清空缓冲区。
            if (!this.byteBufferRead.hasRemaining()) {
                this.byteBufferRead.flip();
                this.processPosition = 0;
            }

            while (this.byteBufferRead.hasRemaining()) {
                try {
                    int readSize = this.socketChannel.read(this.byteBufferRead);
                    if (readSize > 0) {
                        readSizeZeroTimes = 0;
                        this.lastReadTimestamp = DefaultHAConnection.this.haService.getDefaultMessageStore().getSystemClock().now();
                        // 完整的读到一条从服务上报同步进度的数据才进行处理
                        if ((this.byteBufferRead.position() - this.processPosition) >= DefaultHAClient.REPORT_HEADER_SIZE) {

                            // 由于可能收到了多条数据（从服务每同步完一条消息都会上报同步进度，也会定时上报同步进度），只处理最新的同步进度。
                            int pos = this.byteBufferRead.position() - (this.byteBufferRead.position() % DefaultHAClient.REPORT_HEADER_SIZE);
                            long readOffset = this.byteBufferRead.getLong(pos - 8);
                            this.processPosition = pos;

                            DefaultHAConnection.this.slaveAckOffset = readOffset;
                            if (DefaultHAConnection.this.slaveRequestOffset < 0) {
                                DefaultHAConnection.this.slaveRequestOffset = readOffset;
                                log.info("slave[" + DefaultHAConnection.this.clientAddress + "] request offset " + readOffset);
                            }

                            // 从服务器同步完成后通知GroupSevice
                            DefaultHAConnection.this.haService.notifyTransferSome(DefaultHAConnection.this.slaveAckOffset);
                        }
                    } else if (readSize == 0) {
                        if (++readSizeZeroTimes >= 3) {
                            break;
                        }
                    } else {
                        log.error("read socket[" + DefaultHAConnection.this.clientAddress + "] < 0");
                        return false;
                    }
                } catch (IOException e) {
                    log.error("processReadEvent exception", e);
                    return false;
                }
            }

            return true;
        }
    }
    ```

### WriteSocketService

用于向客户端同步数据，使用MMAP传输CommitLog文件。

- run

    ```java
    @Override
        public void run() {

            while (!this.isStopped()) {
                try {
                    this.selector.select(1000);

                    // -1表示还没收到客户端的同步进度请求
                    if (-1 == DefaultHAConnection.this.slaveRequestOffset) {
                        Thread.sleep(10);
                        continue;
                    }

                    // -1表示初次传输数据给客户端，slaveRequestOffset为0时，从当前偏移量往前一个commitlog文件大小的位置开始传输。
                    // 即一个从服务启动时，并不会同步所有的消息，只会最多同步一个commitlog文件大小的数据
                    // Q: 目的是否是防止在数据多的情况下，从服务器启动后一直处理同步过程中，由于同步一直无法完成，导致生产消息无法成功？这样会导致如果消费者要从从服务器获取之前的消息则无法获取到。
                    if (-1 == this.nextTransferFromWhere) {
                        if (0 == DefaultHAConnection.this.slaveRequestOffset) {
                            long masterOffset = DefaultHAConnection.this.haService.getDefaultMessageStore().getCommitLog().getMaxOffset();
                            masterOffset =
                                masterOffset
                                    - (masterOffset % DefaultHAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig()
                                    .getMappedFileSizeCommitLog());

                            if (masterOffset < 0) {
                                masterOffset = 0;
                            }

                            this.nextTransferFromWhere = masterOffset;
                        } else {
                            this.nextTransferFromWhere = DefaultHAConnection.this.slaveRequestOffset;
                        }
                    }

                    // 由于是非阻塞IO，需要判断缓存区的数据是否全部写完。
                    if (this.lastWriteOver) {

                        long interval =
                            DefaultHAConnection.this.haService.getDefaultMessageStore().getSystemClock().now() - this.lastWriteTimestamp;

                        // 心跳机制，如果缓存区数据写完了，也会定时去发起写事件，数据为空。
                        if (interval > DefaultHAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig()
                            .getHaSendHeartbeatInterval()) {

                            this.byteBufferHeader.position(0);
                            this.byteBufferHeader.limit(TRANSFER_HEADER_SIZE);
                            this.byteBufferHeader.putLong(this.nextTransferFromWhere);
                            this.byteBufferHeader.putInt(0);
                            this.byteBufferHeader.flip();

                            this.lastWriteOver = this.transferData();
                            if (!this.lastWriteOver)
                                continue;
                        }
                    } else {
                        // 传输数据，写完selectMappedBufferResult的数据。
                        this.lastWriteOver = this.transferData();
                        if (!this.lastWriteOver)
                            continue;
                    }

                    // 获取传输起始位置的MMAP文件切片
                    SelectMappedBufferResult selectResult =
                        DefaultHAConnection.this.haService.getDefaultMessageStore().getCommitLogData(this.nextTransferFromWhere);
                    if (selectResult != null) {
                        // 确认本次传输的数据量，不超过haTransferBatchSize，不超过流控值canTransferMaxBytes
                        int size = selectResult.getSize();
                        if (size > DefaultHAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig().getHaTransferBatchSize()) {
                            size = DefaultHAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig().getHaTransferBatchSize();
                        }

                        int canTransferMaxBytes = flowMonitor.canTransferMaxByteNum();
                        if (size > canTransferMaxBytes) {
                            if (System.currentTimeMillis() - lastPrintTimestamp > 1000) {
                                log.warn("Trigger HA flow control, max transfer speed {}KB/s, current speed: {}KB/s",
                                    String.format("%.2f", flowMonitor.maxTransferByteInSecond() / 1024.0),
                                    String.format("%.2f", flowMonitor.getTransferredByteInSecond() / 1024.0));
                                lastPrintTimestamp = System.currentTimeMillis();
                            }
                            size = canTransferMaxBytes;
                        }

                        long thisOffset = this.nextTransferFromWhere;
                        this.nextTransferFromWhere += size;

                        selectResult.getByteBuffer().limit(size);
                        this.selectMappedBufferResult = selectResult;

                        this.byteBufferHeader.position(0);
                        this.byteBufferHeader.limit(TRANSFER_HEADER_SIZE);
                        this.byteBufferHeader.putLong(thisOffset);
                        this.byteBufferHeader.putInt(size);
                        this.byteBufferHeader.flip();

                        // 传输数据
                        this.lastWriteOver = this.transferData();
                    } else {
                        // SelectMappedBufferResult为null，则表示主节点数据都还没写入MMAP，使所有写请求等待。
                        DefaultHAConnection.this.haService.getWaitNotifyObject().allWaitForRunning(100);
                    }
                } catch (Exception e) {

                    DefaultHAConnection.log.error(this.getServiceName() + " service has exception.", e);
                    break;
                }
            }

            // 如果写服务终止了，则关闭Connection。
            DefaultHAConnection.this.haService.getWaitNotifyObject().removeFromWaitingThreadTable();
            ...
        }
    ```

    极端情况下，如果Slave的偏移量对应的commitlog已被删除，则同步将再也无法继续了，因为Master不会响应Slave的同步请求，只会一直发送心跳；如果从节点中断较长时间，在ASYNC模式（SYNC模式非ALL_ACK）下，仍然可以继续生产消息，当生成新的commitlog后，旧的commitlog就可能被删除（磁盘空间不足），从节点恢复连接后就会继续从之前的位置同步，但是对应的commitlog已经被删除了；即只要从节点下线后仍然可以生产消息就会导致这个问题。

## GroupTransferService extends ServiceThread

同步阻塞式的主从同步实现，单线程。

- putRequest

    消息写入磁盘后需要等数据传输到Slave完成，在CommitLog的handleDiskFlushAndHA方法中调用。

    ```java
    // DefaultHAService
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

    Slave向Master同步offset（同步请求），Slave侧发起写操作，对应Master侧则为读事件；等待Master同步数据，监听读事件，对应Master侧发起写操作。

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
                int bodySize = this.byteBufferRead.getInt(this.dispatchPosition + 8);

                long slavePhyOffset = this.defaultMessageStore.getMaxPhyOffset();

                // 校验偏移量是否一致
                // Q: 如果slavePhyOffset至masterPhyOffset之间的数据已被删除会怎么样？
                if (slavePhyOffset != 0) {
                    if (slavePhyOffset != masterPhyOffset) {
                        log.error("master pushed offset not equal the max phy offset in slave, SLAVE: "
                            + slavePhyOffset + " MASTER: " + masterPhyOffset);
                        return false;
                    }
                }

                // 如果已经获取到一条完整的数据（非一条消息，haTransferBatchSize）了则处理
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

            // 如果未获取到一条完整的数据则表示缓冲区已使用完成，则需要交换缓冲区。
            if (!this.byteBufferRead.hasRemaining()) {
                this.reallocateByteBuffer();
            }

            break;
        }

        return true;
    }
    ```
