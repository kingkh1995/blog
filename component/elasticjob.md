# [首页](/blog/)

> [ElasticJob](https://shardingsphere.apache.org/elasticjob/current/cn/overview/)

***

## ElasticJob-Lite

使用jar包的形式提供分布式任务的协调服务

### 作业类型

- **SimpleJob**

```java
public class MyElasticJob implements SimpleJob {
    
    @Override
    public void execute(ShardingContext context) {
        // ...
    }
}
```
简单作业，只执行一次。

- **DataFlowJob**

```java
public class MyElasticJob implements DataflowJob<Foo> {
    
    @Override
    public List<Foo> fetchData(ShardingContext context) {
        // ...
        return null; // 如果返回null或空集合则不会执行processData
    }
    
    @Override
    public void processData(ShardingContext shardingContext, List<Foo> data) {
        // ...
    }
}
```
流式处理作业，先执行fetchData再执行processData，如果配置了streaming.process=true，则会一直运行，直到fetchData返回null或空集合。

- **脚本作业**

无需编码，通过script.command.line配置脚本。

### 使用 Spring Boot Starter

1. 添加依赖

    ```xml
    <dependency>
        <groupId>org.apache.shardingsphere.elasticjob</groupId>
        <artifactId>elasticjob-lite-spring-boot-starter</artifactId>
        <version>3.1.0-SNAPSHOT</version>
    </dependency>
    ```
2. 配置作业Bean
3. 添加配置文件
    ```yaml
    elasticjob:
        regCenter: # zookeeper配置
            serverLists: localhost:6181
            namespace: dev/elasticjob
        tracing: # 配置事件追踪，将调度日志持久化数据库。
            type: RDB
        jobs: # 作业配置
            dataflowJob: # 作业Bean名称
                # 基于class的job，需要指定class，可以是具体的class名，也可以是作业接口名
                elasticJobClass: com.example.job.MyDataFlowJob
                elasticJobClass: org.apache.shardingsphere.elasticjob.dataflow.job.DataflowJob
                cron: 0/5 * * * * ?
                shardingTotalCount: 3
                shardingItemParameters: 0=Beijing,1=Shanghai,2=Guangzhou
                jobErrorHandlerType: LOG # 配置作业错误处理策略，可自定义。
            scriptJob:
                # 基于type的作业，要执行作业type
                elasticJobType: SCRIPT
                cron: 0/10 * * * * ?
                shardingTotalCount: 3
                props:
                    script.command.line: "echo SCRIPT Job: "

    ```
4. 配置一次性调度作业（手动触发）
    ```yaml
    elasticjob:
        jobs:
            myOneOffJob:
                # 为作业配置OneOffJobBootstrap名称，无需配置cron
                jobBootstrapBeanName: myOneOffJobBean
                ....
    ```

    ```java
    @RestController
    public class OneOffJobController {

        // 通过 "@Resource" 注入
        @Resource(name = "myOneOffJobBean")
        private OneOffJobBootstrap myOneOffJob;


        // 通过 "@Autowired" 注入
        @Autowired
        @Qualifier("myOneOffJobBean")
        private OneOffJobBootstrap myOneOffJob;
        
        @GetMapping("/execute")
        public String executeOneOffJob() {
            myOneOffJob.execute();
            return "{\"msg\":\"OK\"}";
        }
    }
    ```
5. [更多配置](https://shardingsphere.apache.org/elasticjob/current/cn/user-manual/configuration/spring-boot-starter/)

***

## ElasticJob-Cloud

***
