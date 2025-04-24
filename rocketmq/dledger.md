# [RocketMQ](/blog/component/rocketmq)

> Dledger
>> 基于Raft协议的主从切换及数据同步。

## 使用

至少需要三个节点，且节点数最好为奇数，因为4个节点和3个节点的可用性相同（最多允许一个节点宕机）。

## DLedgerCommitLog extends CommitLog
