## [首页](https://kingkh1995.github.io/blog/)

#### Mysql题解

##### 1.取出每个科目分数排名前2的数据
``` sql
# 使用子查询
select * from 
table t0 
where (
    select count(*) from 
    table t1 
    where 
    t1.subject = t0.subject and t1.score > t0.score
    ) < 2

#使用exists
select * from
table t0
where exists (
    select 1 from 
    table t1
    where 
    t1.subject = t0.subject and t1.score > t0.score
    having count(*) < 2)

#以上两种方式是一样的，主表都是走全表查询，子查询会走subject索引
```
