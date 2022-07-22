# [首页](/blog/)
> Spring Cloud Alibaba

***

## Nacos

### Nacos Config

#### **动态配置原理**

- 客户端通过ClientWorker监听nacos服务器，配置变更后客户端发送RefreshEvent事件；
- RefreshEventListener收到事件后，执行其持有的ContextRefresher变量的refresh方法；
- 第一步，重新加载Environment，之后发送EnvironmentChangeEvent事件，ConfigurationPropertiesRebinder收到事件后，对所有的配置类执行rebind方法；
    - 对所有@ConfigurationProperties注解的bean，从ApplicationContext获取到AutowireCapableBeanFactory，先执行destroyBean再执行initializeBean，**会重新启动应用**。
- 第二步，调用ContextRefresher持有的RefreshScope变量的refreshAll方法，使得所有@RefreshScope注解的bean的缓存失效，**再次获取时才重新加载**；
    - bean的作用域除了Singleton和Prototype之外都会创建Scope对象，@RefreshScope表示设置作用域为refresh，实现为从缓存获取bean。
- 最后发送RefreshScopeRefreshedEvent事件，可自定义添加监听器。

***

## RocketMQ

***

## Seata

支持AT、TCC、SAGA 和 XA四种事务模式

### 分布式事务组件

- Transaction Coordinator (TC)： 事务协调器，维护全局事务的运行状态，负责协调并驱动全局事务的提交或回滚；
- Transaction Manager (TM)： 控制全局事务的边界，负责开启一个全局事务，并最终发起全局提交或全局回滚的决议；
- Resource Manager (RM)： 控制分支事务，负责分支注册、状态汇报，并接收事务协调器的指令，驱动分支（本地）事务的提交和回滚。

### AT事务模式
1. 一阶段：业务数据和回滚日志记录在同一个本地事务中提交，释放本地锁和连接资源。
   - branchRegister：RM与TC交互获取全局锁和BranchId；
   - branchReport：本地事务提交后，RM将结果通知TC；
2. 二阶段：由TM形成全局决议提交到TC，TC下方命令到RM。
   - branchCommit：只需要删除一阶段的undo_log和释放全局锁，并不是立即执行而是异步处理以提升性能；
   - branchRollback：使用undo_log表进行回滚，回滚前需要校验脏写，如果失败**则只能人工处理**（FailureHandler）。

- before image & after image：保存更新前后快照，用于恢复和校验是否发生脏写。
- 优点：性能高、无入侵；缺点：SDK较重、脏写无法处理。

#### 隔离性

默认只能保证RU，因为AT模式下本地分支事务在第一阶段提交，而全局事务在第二阶段提交，故本地事务提交后其他分支事务或未被seata管理的事务可以查到数据变更。为了保证RC需要使用select for update操作，该操作会额外去获取全局行锁，同一个全局事务内的其他分支事务可以再次获取，但其他全局事务会获取失败（**未被Seata管理的事务当然不受影响**）。

***

## Sentinel

***

```
### Nacos
#### 1.nacos docker
    > 使用方法：
        克隆项目：
            git clone https://github.com/nacos-group/nacos-docker.git
        启动：（使用/nacos-docker/example目录下的yaml文件）
            docker-compose -f example/standalone-derby.yaml up
        控制台：（端口：8848 路径：/nacos/）
            访问地址：http://127.0.0.1:8848/nacos/
            
#### 2.nacos config   
    > 使用方法：
        添加依赖：
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
        配置：
            在bootstrap.properties中配置nacos的地址和应用名
        添加配置文件：
            在nacos上添加配置文件，指定dataId（格式：${prefix}-${spring.profile.active}.${file-extension}）和group，也可以指定namespace
        使用：
            使用@Value注解和@ConfigurationProperties获取配置，并使用配置类上使用@RefreshScope或者@ConfigurationProperties实现配置实时刷新
            可以添加拓展配置spring.cloud.nacos.config.extension-configs[n]和共享配置spring.cloud.nacos.config.shared-configs[0]
            自定义data-id配置默认关闭自动刷新，同时n越大优先级越高，三者之间，自动生成的data-id优先级最高，其次是拓展配置，最后是共享配置
       原理：
        自动注入：
            实现了PropertySourceLocator接口，会在在Spring Cloud应用启动阶段（所以需要在bootstrap.properties中配置）会主动从nacos端获取配置文件，
            并将获取到的数据转换成PropertySource且注入到Environment的PropertySources属性中。
        动态刷新：
            为配置项添加了监听器，在监听到服务端配置发生变化时会实时触发ContextRefresher的refresh方法。
            
#### 3.nacos discovery
    > 使用方法：
        添加依赖：
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId> 
        配置：
            在application.properties中配置nacos地址
        服务注册：
            添加spring cloud默认的服务发现注解@EnableDiscoveryClient即可
        服务发现：
            使用带有负载均衡的RestTemplate和原生FeignClient
    原理：
        服务注册;
            遵循了spring cloud common标准，实现了AutoServiceRegistration、ServiceRegistry、Registration这三个接口。在应用的启动阶段，
            监听了WebServerInitializedEvent事件，当Web容器初始化完成后，会触发注册的动作，调用ServiceRegistry的register方法，将服务注册到Nacos。
        服务发现；
            NacosServerList实现了ServerList接口，并在@ConditionOnMissingBean的条件下进行自动注入，默认集成了Ribbon。
          
### Sentinel
    > 使用方法：
        添加依赖：
            <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
        配置控制台信息：
            配置控制台地址和对接控制台的Http Server的端口spring.cloud.sentinel.transport.port
        标识资源：
            在方法上使用@SentinelResource注解用来标识资源，定义value属性指定资源名。
            blockHandler/blockHandlerClass：被限流/降级/系统保护的时候调用，用于处理BlockException，方法必须在最后添加一个BlockException的参数。
            fallback/fallbackClass：会针对所有类型的异常，用于抛出异常时的处理，可以添加一个Throwable类型的参数。
            defaultFallback：通用的 fallback 逻辑，优先级最低，返回值必须与原方法相同，参数列表为空，或者有一个Throwable类型的参数
            若均未配置，则被限流降级时会将BlockException直接抛出（若方法本身未定义 throws BlockException则会被JVM包装一层UndeclaredThrowableException）
        Feign支持：
            直接兼容FeignClient，接口对应的sentinel中的资源名为httpmethod:protocol://requesturl（例：GET:http://service-provider/echo/{str}）
        RestTemplate支持：
            在构建bean时添加@SentinelRestTemplate注解，支持两种粒度httpmethod:schema://host:port/path和httpmethod:schema://host:port
        url支持：
            自动对所有url埋点，如需自定义拦截信息，实现BlockExceptionHandler。
    原理：
        RestTemplate支持实现原理：
            添加拦截器SentinelProtectInterceptor
            
### RocketMQ
    > docker部署：
        docker pull rocketmqinc/rocketmq
        docker run -d -p 9876:9876 -v /rmq/namesrv/logs:/root/logs -v /rmq/namesrv/store:/root/store --name rmqnamesrv  rocketmqinc/rocketmq sh mqnamesrv
        docker run -d -p 10909:10909 -p 10911:10911 -v /rmq/broker/logs:/root/logs -v /rmq/broker/store:/root/store --name rmqbroker --link rmqnamesrv:namesrv -e "NAMESRV_ADDR=namesrv:9876" rocketmqinc/rocketmq sh mqbroker -c ../conf/broker.conf 
        docker exec -it rmqbroker bash
        修改/conf/broker.conf 添加brokerIP1=127.0.0.1 autoCreateSubscriptionGroup=true 重新启动broker
      docker部署控制台：
        docker pull styletang/rocketmq-console-ng
        docker run -e "JAVA_OPTS=-Drocketmq.namesrv.addr=172.17.0.2:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=false" -p 8080:8080 -t styletang/rocketmq-console-ng
      使用方法：
        添加依赖，配置rocketmq服务地址，开启@EnableBinding注解，可以使用stream框架提供的Source、Sink和Processor接口或者自定义Binding接口，之后配置binding即可。
    原理：
        
### Dubbo
    > 使用方法：
        结合Nacos使用：
            添加dubbo starter依赖和nacos discovery依赖即可。
        配置：
            配置协议，默认使用dubbo协议，生产者需要配置包扫描和指定dubbo协议端口，消费者需要配置订阅的服务名称。
        使用：
            生产者使用@Service注解（dubbo包的），消费者使用@Refrence注解。
        调试：
            关闭服务注册：dubbo.registry.register=false
            直连提供者：@Reference(url = "dubbo://{ip}:{port}")
```