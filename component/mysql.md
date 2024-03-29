# [首页](/blog/)
> MySQL（存储引擎为InnoDB）

***

## 三大范式

第一范式（1 NF）：字段不可再拆分。
第二范式（2 NF）：表中任意一个主键或任意一组联合主键，可以确定除该主键外的所有的非主键值。
第三范式（3 NF）：在任一主键都可以确定所有非主键字段值的情况下，不能存在某非主键字段 A 可以获取 某非主键字段 B。

***

## 基础架构

客户端 -> 服务器（连接器-查询缓存-分析器-优化器-执行器）-> 存储引擎 -> 文件系统

- 连接器：身份认证和权限相关。
- 查询缓存：8.0版本移除，执行查询语句前会先查询缓存。
- 分析器：分析SQL语句。
- 优化器：服务器为SQL语句选择出其认为的最优的方案。
- 执行器： 执行语句，从存储引擎中读取数据返回，首先会判断是否存在权限。
- 存储引擎 ： 插件式架构，主要负责数据的存储和读取，支持InnoDB、MyISAM、Memory等多种存储引擎。

***

## 数据存储

依次可以分为：表空间、段、区、页、行。

- Buffer Pool：使用缓冲池来减小CPU和磁盘速度上的差异。

- 表空间：作为存储结构的最高层，所有数据都存放在表空间中，默认情况下使用一个共享表空间。

- 段：表空间由各个段构成，包括索引段，数据段，回滚段。

- 区：段由区组成，每个区固定大小是1MB。

- 页：区由连续的页组成，每个页的大小默认为16kb，所以默认情况下一个区包括64个连续的页；是磁盘读取的最小单位，页中的用户记录行是使用链表的方式存储；页也包括页目录（slot），用于二分查找快速定位，每个slot包含4到8个数据行。

- 行：页中存储的具体的记录。组成包括：变长字段长度列表（变长分配的空间就等于字符大小）、字段NULL标志位（列为空则在对应的位上置为1，同时不存储该列值，也导致索引字段为空会进行额外的操作）、记录头信息、mvcc版本信息、列数据、（如果未声明主键还会存在一个内置的记录ID）。

### 分区表

分区表是一个独立的逻辑表，但是底层由多个物理子表组成。MySQL使用【PARTITION BY】定义每个分区存放的数据，**分区表的索引只是在各个底层表上各自加上一个完全相同的索引**，主要目的是将数据按照一个较粗的粒度分在不同的表中（如历史数据归纳）。

***

## redolog & undolog & binlog

- redo log：用于崩溃恢复，只有数据库启动时才会读取；为日志文件组，顺序写，同时是环形写，会覆盖旧的文件；记录的是在物理页的操作。
  - **redo log buffer**：操作同时还会写入redo log buffer中，后台线程会每隔1秒或缓冲区大小达到阈值后，把缓冲区中的内容写到文件系统缓存（page cache），然后调用fsync刷盘。

- undolog：用来事务回滚和MVCC；记录的是和原SQL逻辑相反的SQL，快照读通过undolog计算得到记录原值，事务则通过执行undolog的SQL进行回滚；**undolog一定存在对应的redolog**；有一个后台线程会定时处理所有事务都不会再使用的undolog。

- binlog：属于MySQL服务器，用于数据同步；记录的是原始逻辑；追加写不会覆盖旧记录；有三种格式：statement、row、mixed。
  - statement：原始SQL，如果包含now()函数等则会导致执行结果不一致，RR以下隔离级别不允许此格式；
  - row：会额外记录下操作数，需要更大的资源；
  - mixed：自动判断选择statement或row。

***

## 事务

### ACID

- 原子性（Atomicity）：事务内的所有操作要么全部成功，要么全部失败；
- 一致性（Consistency）：执行事务前后，数据的完整性不被破坏，**C是目的，，而AID是手段**；
- 隔离性（Isolation）：并发执行的事务之间互不影响；
- 持久性（Durabilily）：事务一旦提交，它对数据的改变就是永久的。

### 并发事务带来的问题

- 脏读：**读取到其他事务未提交的数据**；
- 不可重复读：事务内多次读取同一数据内容不一致，**读取到其他事务已提交的数据**；
- 幻读：事务内读取多行数据，之后的读取会多出几条数据，**其他事务仍然可以在当前事务读取范围内新增数据**。

### 隔离级别

- Read Uncommited：读未提交，会出现脏读、不可重复读、幻读；
- Read Committed (RC)：读已提交，允许读取到并发事务提交的数据，可以阻止脏读；
    > 通过加锁可以解决不可重复读的问题，但是仍然无法解决幻读。 
- Repeatable Read (RR)：可重复读，多次读取（**快照读**）结果一致，允许出现幻读；
    > **InnoDB默认隔离级别，使用MVCC（快照读） + Next-key Lock（当前读）防止了幻读。**
- Serializable：可序列化，事务不允许并发执行。
    > InnoDB下会由MVCC退化为基于锁的并发控制，所有的读均默认为当前读。

### 事务执行流程

1. 分配事务ID（自增值），开启事务，获取锁，没有获取到锁则等待；
2. 执行器先通过存储引擎找到对应的数据页，如果Buffer Pool（缓冲池）存在数据则直接取出，没有则回查主键索引，从磁盘取出并放入缓冲池；
3. 在数据页内找到需要具体的记录，修改后写入Buffer Pool；
4. 存储引擎生成redolog和undolog到内存中，**将redolog状态设为预提交**；
5. 将redolog和undolog写入文件中并调用fsync刷盘；
6. 事务提交，服务器生成binlog并写入binlog文件中，调用fsync保证刷盘；
7. 将redolog状态改为已提交，并释放所有锁。

- *如果redolog设置提交阶段故障，服务器恢复后会通过事务ID找到对应的binlog日志，并继续提交事务，恢复数据。*

### AUTOCOMMIT机制

MySQL默认采用自动提交模式，即如果不显式使用START TRANSACTION语句来开始一个事务，那么每个查询操作都会被当做一个事务并自动提交。

***

## MVCC

多版本并发控制协议，InnoDB会为每条记录添加额外字段：
- DB_TRX_ID：最后更新该记录的事务ID
- DB_ROLL_PTR：回滚指针，指向该记录的undolog
- DB_ROW_ID：如果该表没有主键且没有唯一非空索引，会使用该ID生成聚簇索引
- 删除位：标识是否被删除

优点是快照读不需要获取锁，提高了系统的并发度；缺点是需要维护每条记录的版本信息，且在检索行时需要判断版本是否可见，降低了查询的效率，同时还需要定期清理及时回收空间。

### 更新数据流程

1. 获取排他锁；
2. 修改记录；
3. 写redolog和undolog；
4. 设置当前事务ID，将回滚指针指向undolog历史数据。

### ReadView

配合MVCC使用，组成：当前事务ID、当前进行中的事务ID集合、当前进行中的事务ID集合的最小值、当前将要分配的下一个事务Id（即事务ID的上限）。

**RR级别只会在第一个查询时创建ReadView，而RC级别每次查询都会创建一个ReadView，所以RR级别快照读不会出现幻读。**

查询过程：
   1. 查询到某条记录后，判断该版本记录的trx_id是否等于ReadView中的creator_trx_id是否相等，相等则表示可读直接返回，否则进行以下判断：
      1. 小于ReadView记录的最小事务号，则可读；
      2. 大于等于ReadView记录的最大事务号，则不可读；
      3. 在两者之间，**则在ReadView记录的进行中的事务ID集合中查找当前版本事务ID，如果找不到则表示创建当前ReadView时该事务已经提交故可读，否则表示事务还未提交不可读**。
   2. 如果当前版本不可读，通过回滚指针沿着undolog链向上查找历史版本，重复上面步骤。

***

## 锁

### 锁策略

- X锁（排他锁-写锁）：lock table write / select for update
  - 同一时刻只能由一个事务加锁，与任何类型的锁都不兼容。

- S锁（共享锁-读锁）：lock table read / lock in share mode
  - 可以同时被多个并行事务加锁，保证数据不能被其他事务修改；
  - **申请到S锁后可以继续申请X锁，如果还有其他并行事务也申请到S锁则会失败。**

### 锁类型

**表级锁主要用于执行DDL语句。**

**意向锁**：表级锁，包括意向共享锁（IS 锁）和意向排他锁（IX 锁）。
- “表明”加锁的意图，申请行级锁前要先申请对应的意向锁，**申请表级锁时只需要先判断是否存在其他表级锁，再判断是否存在意向锁即可**。
- 完全由数据库引擎自己维护，用户无法手动操作。

**行级锁：**
- 记录锁（Record Lock）：锁住单条数据，**是锁住索引而不是记录本身**。
  - 更新或删除单条数据前默认会先获取其排他记录锁；
- 间隙锁（Gap Lock）：锁住索引的一个范围，**用于阻塞插入意向锁**。
  - 间隙锁之间不会互相阻塞。
- 插入意向锁：**不属于意向锁，而是由INSERT操作产生的一种间隙锁**，表面在加锁区间的插入意图。
  - 插入意向锁不会互相阻塞，插入操作时才会判断数据之间是否冲突；
  - 实际上不会阻塞任何锁，包括间隙锁，且只会被间隙锁阻塞。
- 临键锁（Next-key Lock）：记录锁+间隙锁，锁定索引上的一个范围，包括记录本身，为InnodDB在RR级别下的加锁方式。

### 2PL

二阶段锁，事务期间加锁和解锁分为两个完全不相交的阶段，加锁阶段只加锁，解锁阶段只释放锁。

### RR级别加锁行为

- 快照读：不加任何类型的锁。

- 加S锁：在使用的索引上对记录加共享Next-key Lock，如果需要回表查询，则在主键索引上加上对应记录的共享记录锁。

- 加X锁/UPDATE/DELETE：在使用的索引上对记录加排他的Next-key Lock，同时在主键索引上加上对应记录的排它记录锁。
    - UPDATE和DELETE操作在执行前都会先执行一个当前读操作；
    - **因为加锁是引擎层面的行为，如果无法使用索引会对所有行加上间隙锁和记录锁后返回，不过执行器在过滤时会通知引擎释放掉不匹配行的记录锁，虽然这明显违背了2PL，注意间隙锁仍然不会释放。**

- INSERT：在插入数据之前，要在所有索引上对要插入的范围加上插入意向锁，在插入数据成功后会在插入的行上加上排它记录锁。
    - 如果检测到唯一键冲突，则会申请冲突行的S锁；t0为成功插入的事务，t1和t2是检查到唯一键冲突的事务，t0提交或回滚后，t1和t2都会成功申请到冲突位置的S锁；如果t0提交了，则t1和t2获取到S锁后会抛出主键冲突错误；如果t0回滚了，则t1和t2都需要再获取到X锁才可以插入数据，但由于对方都获取到了S锁，则永远不可能获取到X锁，故出现死锁，根据死锁机制，第一个尝试获取X锁的事务会成功，其他事务则抛出死锁错误；
    - 如果有自增列（一张表只允许一个自增列）会维护加一个表级排他锁以获取到自增值，不过这个表级锁会在插入完成后释放，**最新版本改为使用互斥量实现，提高了效率**。

- INSERT ... ON DUPLICATE KEY UPDATE：检测到唯一键冲突会直接申请加排它锁。

### 组合索引加锁

- **idx(a,b)**
- **a** : 1 | 3 | 3 | 3 | 3 | 5
- **b** : 1 | 1 | 2 | 2 | 3 | 5
- **c** : 1 | 2 | 3 | 1 | 4 | 5
- **id**: 3 | 5 | 1 | 2 | 4 | 6

1. delete from t where a>1 and a<5 and b=2 and c=1：组合索引上，gap锁5个（a值1到5之间），X锁两个（b=2），聚簇索引X锁两个（id=1,2）

2. delete from t where a=3 and b>1 and b < 3 and c=1：组合索引上，gap锁3个（a值等于3且b值大于1小于3），X锁两个（b=2），聚簇索引X锁两个（id=1,2）

### 死锁产生条件

- 多个并发事务（2个或者以上）；
- 每个事务都持有锁（或者是已经在等待锁）；
- 每个事务都需要再继续持有锁（为了完成事务逻辑，还必须更新更多的行）；
- 事务之间产生加锁的循环等待，形成死锁。

### 产生死锁原因

1. 两个事务行锁加锁顺序不一致；
   - 批量更新时，按固定的顺序（如ID顺序）操作。
2. 两个事务的间隙锁互斥，先执行删除就获得了间隙锁再执行插入就会被对方阻塞；
   - 事务级别调整到RC。
3. 唯一索引插入导致死锁，第一个事务插入后回滚之后，其他两个事务通过一个当前读获取到了共享间隙锁，导致互相等待；
   - 使用insert on duplicate key update。
4. 同一加锁行为，由于index_merge使用了多个索引，这多个索引对主键行锁的加锁顺序不一致。
   - 使用force index；
   - 创建组合索引。

***

## 索引

### 索引类型

- B树索引：
    - 使用B+树实现，有序存储，一般为1-3层，非叶子结点只保存索引值，在叶子结点才保存被索引的数据，同时叶子结点之间通过指针顺序连接。
    - 优点是减少了磁盘IO的次数，且每次查询都是稳定的，因为数据行到根节点的高度是相同的。

- 哈希索引：使用哈希表实现，无序存储，查询快，但无法排序和范围搜索。
    
- **自适应哈希索引**：InnoDB会自发的在B+树索引基础上对被频繁使用的索引值在内存中创建一个哈希索引以提高查找效率，此为完全存储引擎的行为只能开启功能。

### 索引策略

- 聚簇索引：将数据行和索引保存在一起，而不是通过指针指向数据行。
    - InnoDB的主键索引是聚簇索引，将数据行保存在B树索引的叶子结点中；
    - 优点是不再需要磁盘IO去查找数据，缺点是更新代价很高，会导致“页分裂”。

- 非聚簇索引：也称为二级索引，普通索引。
    - InnoDB的二级索引的叶子结点保存的是主键值而不是数据行指针，所以通过二级索引查找数据时可能需要两次b树查找。
    - **二级索引默认会将主键作为最后一列，即默认都是组合索引。**

- 唯一索引：索引值必须是唯一的，可以为Null，主键索引是唯一索引且不能为Null。

- 组合索引：多列索引，使用最左前缀匹配原则，所以应该将区分度高的列放在左边。

- 前缀索引：针对字符串列的索引不需要索引完整的字符串，只索引字符串前几位，以提升索引效率，因为只能使用左模糊进行查询。

- 覆盖索引：如果索引覆盖了查询需要的所有数据行，则不需要再去读取数据行，称为覆盖索引。
  
- 索引下推：5.6版本推出，用于优化非聚簇索引查询效率，**如果存在某些被索引的列的判断条件，MySQL服务器会将这一部分判断条件传递给存储引擎**，然后由存储引擎可以提前过了掉不匹配的记录，减少回表查询的次数。

### index_merge

index_merge是MySQL 5.1后引入的一项索引合并优化技术，**它允许对同一个表同时使用多个索引进行查询，并对多个索引的查询结果进行合并后返回**。在使用index_merge技术后，会同时执行两个索引，故可能导致死锁，可以使用【force index】操作避免。

### 索引失效场景

1. 查询范围太大，即服务器认为走全表扫描会更快，包含使用负向扫描（NOT）的场景；
2. 数据隐式转换，关联查询时字符集不同也会导致隐式转换字符；
3. 对列使用函数；
4. 对列进行运算；
5. like使用左模糊；
6. 组合索引不符合最左匹配原则。

### 索引选择策略

扫描行数是主要因素但并不是唯一的判断标准，优化器还会结合是否使用临时表、是否排序等因素进行综合判断。会统计基数用于预估行，即选取一定的数量的数据页，统计其不同值得到平均值。

1. 主键索引优先；
2. 可以使用覆盖索引；
3. 扫描行数少；
4. 可以用于排序；
5. 索引大小。

### show index from

- cardinality: 表示索引在当前列唯一值的数量，为估计值。

***

## 主从复制

1. 主库将数据库中数据的变化写入到binlog；
2. 从库连接主库，主库会创建一个binlog dump线程来发送binlog，从库中的I/O线程负责接收更新的binlog；
3. 从库的I/O线程将接收的binlog写入到relay log中；
4. 从库的 SQL 线程读取relay log同步数据本地（再执行一遍SQL）。

为异步复制，主库只保证将数据变更写入binlog，但不关心是否被从库接收，缺点就是如果主库出现宕机且还未来的及将新写入的binlog发送给从库，此时若将从库升级为主库则会出现数据丢失。

***

## EXISTS & IN
1. 如果无法使用索引的情况下，MySQL会把IN的查询语句改成EXISTS去执行；
2. IN查询在内部表和外部表上都可以使用到索引，Exists查询仅在内部表上可以使用到索引；
3. 当子查询结果集很大，而外部表较小的时候，Exists的BNL算法的作用开始显现，并弥补外部表无法用到索引的缺陷，查询效率会优于IN；而当子查询结果集较小，而外部表很大的时候，IN的外表索引优势占主要作用，此时IN的查询效率会优于Exists。

**Block Nested Loop**：将外层循环的行/结果集存入join buffer, 内层循环的每一行与整个buffer中的记录做比较，从而减少内层循环的次数。

***

## EXPLAIN

- id：查询的编号，分为简单查询和复杂查询，复杂类型分为：简单子查询（SELECT子句中）、派生表（FROM子句中）、UNION查询。
- select_type：查询类型，SIMPLE：简单查询；PRIMARY：复杂查询的最外层；SUBQUERY：简单子查询；DERIVED：派生表；UNION：UNION查询；UNION RESULT：UNION结果集，id为null，它是一个匿名临时表。
- table：表名或者对应的别名，派生表的外层查询则是\<derievedN\>（N为派生表查询的id）。
- type：访问类型，以下效率从差到优：
  - ALL：全表扫描；
  - index：只比ALL快一点，需要遍历整个索引树；
  - range：范围扫描索引树，即是一个有限制的index；
  - ref：是索引查找，返回匹配某个值的行，可能是多个行，因此属于查找和扫描的混合，**查找非唯一索引或唯一索引的非唯一前缀时才会发生**；
  - eq_ref：等值查找，明确只会返回一行，即在查找唯一性索引时发生；
  - const、system：命中主键或唯一索引，且查询条件是常量值。
  - NULL：意味着在优化阶段就可以分解查询条件，不需要再访问表或索引。
- possible_keys：显示可以使用哪些索引，在优化早期创建的，可能对后续优化过程并没有作用。
- key：实际使用的索引，表示最小的查询成本应该使用的索引。、
- key_len：表示索引的**最大字节数**，根据前缀模式可以推算出使用到的列。
- ref：记录了在索引中查找值使用的列或常量。
- rows：为了找到需要的行而大概需要读取的行数。
- filtered：EXPLAIN EXTENDED时出现，悲观的估计大概符合条件的行大概占需要扫描的行的百分比，使用ALL、index、range访问时会使用这个值。
- Extra：
  - Using index：表示使用覆盖索引；
  - Using index condition：表示使用了索引下推；
  - Using where：表示服务器将在检索到行之后再过滤，**意味着可以考虑优化索引结构**；
  - Using temporary：表示对查询结果排序时会建立临时表；
  - Using filesort：表示会对结果使用外部索引排序而不是读取出行，**可能是在内存或磁盘**；
  - Using join buffer：使用了连接缓存，即表连接时未使用到索引；

***

## 数据库调优策略

- 选择适合的数据库
- 优化表设计：合理的字段数据类型、合理的表结构、冷热分离、增加中间表、增加冗余字段
- 优化查询：SQL优化，索引优化
- 增加缓存层
- 库级别优化：读写分离，垂直拆分、水平拆分

***

## SQL题

### 取出每个科目所有分数排名前2的成绩

1. 使用子查询：
``` sql
select * 
from t t0 
where 2 >
(
  select count(distinct t1.score)
  from t t1
  where t1.subject = t0.subject and t1.score > t0.score
);
```

2. 使用exists：
```sql
select * 
from t t0 
where exists
(
  select id 
  from t t1
  where t1.subject = t0.subject and t1.score > t0.score
  having count(distinct t1.score) < 2
);
```
**以上两种方式是一样的，主表都是全表扫描，子查询会走subject索引。**

*如果是找出第N高的成绩，则将<改为=号即可。*

###  找出连续三天以上访客超过100的记录

```sql
select distinct t1.*
from t t1, t t2, t t3
where t1.people > 100 and t2.people > 100 and t3.people > 100
and
(
    (t1.id - t2.id = 1 and t1.id - t3.id = 2 and t2.id - t3.id =1)
    or (t2.id - t1.id = 1 and t2.id - t3.id = 2 and t1.id - t3.id =1)
    or (t3.id - t2.id = 1 and t2.id - t1.id =1 and t3.id - t1.id = 2)
)
order by t1.id;
```

### 树节点

```sql
select id, 
case when t.id=(select t1.id from tree t1 where t1.p_id is null) then 'Root'
when t.id in (select p_id from tree t2) then 'Inner'
else 'Leaf' end
as type
from tree t;
```
