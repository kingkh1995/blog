# [首页](https://kingkh1995.github.io/blog/)

> Effective Coding

## CODING

- 单例模式要防止被反射机制访问私有构造器，私有构造器**要保证创建第二个实例时抛出异常**。

- if-else尽量不要嵌套，代码块少于3行使用卫语句（**fail-fast**），否则使用策略模式。

- 组合使用if、switch或策略模式，**超高频条件使用if判断**。

## JAVA

- 集合作为参数时，如果只会获取元素使用<? extend E>限定，如果只能新增使用<? super E>限定。
  
- 使用完ThreadLocal后记得要调用remove()方法，可以使用Spring提供的RequestContextHolder（请求结束后会清空上下文信息）。 
  > web请求使用线程池会复用线程，造成bug或内存泄漏。

## MYSQL

- 被索引的字段不要出现NULL值。 
  > NULL值不会走索引。

## 技术方案

- **重试**的等待时间不要使用固定值，使用随机值并逐渐增加。 
  > 防止雪崩效应