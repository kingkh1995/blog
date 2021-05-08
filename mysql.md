## [首页](https://kingkh1995.github.io/blog/)

### 锁

#### 锁类型

#### 锁底层实现

#### RR级别下SQL加锁行为

##### select ... from  
不加任何类型的锁

##### select ... from lock in share mode
在扫描到的任何索引记录上加共享的（shared）next-key lock，还有主键聚集索引加排它锁

##### select ... from for update
在扫描到的任何索引记录上加排它的next-key lock，还有主键聚集索引加排它锁

##### update ... where & delete from ... where
在扫描到的任何索引记录上加next-key lock，还有主键聚集索引加排它锁

##### insert into

#### 索引

#### 索引类型

### SQL优化

##### 字段某些特殊值占据了绝大部分的比例

``` sql
status字段中值为0的数据比例超过90%，则status=0时强制不走索引idx_staus
select * from t ignore index (idx_stauts) where status = 0 limit 10;
```

##### 取出每个科目分数排名前2的数据

``` sql
使用子查询
select * from 
table t0 
where (
    select count(*) from 
    table t1 
    where 
    t1.subject = t0.subject and t1.score > t0.score
    ) < 2

使用exists
select * from
table t0
where exists (
    select 1 from 
    table t1
    where 
    t1.subject = t0.subject and t1.score > t0.score
    having count(*) < 2)

以上两种方式是一样的，主表都是走全表查询，子查询会走subject索引
```
