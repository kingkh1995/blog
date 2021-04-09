## [首页](https://kingkh1995.github.io/blog/)
> spring cloud alibaba 学习笔记

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
