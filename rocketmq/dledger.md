# [RocketMQ](/blog/component/rocketmq)

> Dledger
>> 基于Raft协议的主从切换及数据同步。

## 基本概念

主要步骤是Leader选举和日志复制，至少需要三个节点，且节点数最好为奇数，因为4个节点和3个节点的可用性相同（最多允许一个节点宕机）。

## Leader选举

## DledgerClientProtocol

- get
- append

## DledgerProtocol

- vote
- heartBeat
- pull
- push

## DledgerRpcService

## DledgerLeaderElector

- RoleChangeHandler
- memberState
默认状态为candidate，启动即会尝试发起投票，第一轮达成的概率很低。

## 存储实现

### DLedgerCommitLog extends CommitLog

- private final DLedgerServer dLedgerServer;
- private final DLedgerConfig dLedgerConfig;
- private final DLedgerMmapFileStore dLedgerFileStore;
- private final MmapFileList dLedgerFileList;

### DLedgerMmapFileStore extends DLedgerStore

存储实现类，对应DefaultMessageStore

### MmapFileList

物理文件映射，对应MappedFileQueue

获取当前正在写的MMap，如果申请的空间超过可写的空间，会先写入一条空白的Entry，填入魔数和长度，方便解析；创建下一个MMap，然后预占住需要写入的空间；写入则是通过mappedByteBuffer写入PageCache。

### DLedgerEntry

## 日志转发

### DledgerEntryPusher

- private EntryHandler entryHandler;
- private QuorumAckChecker quorumAckChecker;
- private Map<String, EntryDispatcher> dispatcherMap = new HashMap<>();

日志转发核心类，包括三个线程：EntryHandler为日志接受处理线程，从节点启动；QuorumAckChecker为日志追加ACK投票处理线程，主节点启动；EntryDispatcher为日志转发线程，主节点启动。

### EntryHandler

### QuorumAckChecker

### EntryDispatcher

支持四种操作：COMPARE，与从节点日志对比；TRUNCATE，对比完成后，从节点截取日志；APPEND，追加日志；COMMIT，通知提交索引，通常会与APPEND操作合并。