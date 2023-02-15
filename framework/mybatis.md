# [首页](/blog/)

> **MyBatis**

***
## 半自动ORM

MyBatis在查询关联对象或关联集合对象时，需要手动编写 sql 来完成，所以称之为半自动 ORM 映射工具；

Hibernate属于全自动ORM映射工具，使用Hibernate查询关联对象或者关联集合对象时，可以根据对象关系模型直接获取，所以它是全自动的。

***

## Xml 映射文件

namespace为对应的Mapper接口的全限定名，每个sql语句都会被解析为一个MappedStatement对象，通过namespace加上sql语句id即是其key值，**Mapper接口方法可以重载，但sql语句id无法重复**。实现方式即是jdk动态代理，为Mapper接口生成代理对象，每个方法通过key值找到对应的MappedStatement对象并执行。

***

## 插件

### Interceptor

- Object intercept(Invocation invocation)：需要实现拦截逻辑，invocation为被代理方法的封装对象。
- Object plugin(Object target)：确定拦截范围，一般选择通过注解定义，默认实现即是从类注解上获取拦截信息，所以不需要重写。
- setProperties(Properties properties)：如果拦截器需要读取配置文件则重写，properties为配置文件，一般不需要，直接使用Spring框架注入即可。
- @Intercepts：拦截器注解，value属性为@Signature数组。
- @Signature：具体的拦截点，type：拦截对象的类型，method：拦截对象的具体方法名，args：拦截方法的参数类型列表。

### 拦截对象

【Executor】：执行器；【StatementHandler】：Statement处理；【ParameterHandler】：参数处理；【ResultSetHandler】：结果集处理。

执行顺序：Executor->StatementHandler->ParameterHandler->TypeHandler->ResultSetHandler->StatementHandler->Executor

***

## 缓存

均使用Executor实现，一级缓存为SqlSession级别，不同SqlSession之间隔离，默认开启不能关闭，由BaseExecutor实现，commit、rollback以及update方法都会清除缓存；二级缓存是Mapper级别，默认关闭，由CachingExecutor实现。

## SqlSession

每次请求数据库会话都会创建SqlSession对象，每个事务都会创建一个SqlSession。Spring中如果未显式声明事务，则每次sql操作都相当于开启一个事务，即每次都会创建一个SqlSession。

## Executor

- Executor：执行器接口，每个SqlSession对象都有一个Executor对象，故生命周期都被限制在SqlSession中。
- BaseExecutor：执行器的基类，使用了模板方式设计模式，添加了一级缓存逻辑，使用HashMap，缓存key由以下部分组成：MappedStatement的id、Rowbounds的offset和limit、BoundSql解析后的sql、参数值、Environment的key。
- SimpleExecutor：默认使用，每执行一次update或select，就开启一个Statement对象（非MappedStatement），用完立刻关闭Statement对象，**只有Spring中需要开启事务才能使缓存生效**。
- ReuseExecutor：复用Statement，为Statement对象添加HashMap缓存，以sql作为key。
- BatchExecutor：批处理执行器，执行update（没有select，JDBC批处理不支持select），将所有sql都添加到批处理中（addBatch()），等待统一执行（executeBatch()），它缓存了多个Statement对象，每个Statement对象都是addBatch()完毕后，等待逐一执行executeBatch()批处理，与JDBC批处理相同。
- CachingExecutor：是一个Executor接口的装饰器，开启了二级缓存时会包装Executor对象，为Executor对象增加了二级缓存的相关功能，**它的生命周期也在SqlSession中，但是其是从外部查询缓存**。

## SqlSessionFactory

用于创建SqlSession，SqlSession会从Configuration对象中获取对应MappedStatement，交给Executor执行。

## MapperScannerConfigurer

用于获取MapperFactoryBean，然后为mapper接口创建代理。

***