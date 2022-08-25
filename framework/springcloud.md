# [首页](/blog/)

> Spring Cloud

***

## Spring Cloud Context
The Bootstrap Application Context
    spring cloud application 由bootstrap context来引导执行，是applcation的父上下文，当构建一个application context时会自动加入一个作为其父级的bootstrap上下文。
    主要用途：通过spring cloud config加载远程配置属性以及解密本地外部配置文件的属性。
    bootstrap属性有高优先级，默认情况下，它们不会被本地配置覆盖，注意这些属性不是bootstrap.yml文件中的属性，虽然bootstrap.yml文件中的属性也是在bootstrap阶段被加载的，但是其优先级很低。
    bootstrap.yml中应该配置一些和application启动相关的配置，也可以配置其他非启动相关的属性，但会被appliction.yml中的文件覆盖。
    如使用Spring Cloud Config Server时，应该通过bootstrap.yml中配置相关信息来配置bootstrap context以加载远程配置。

EnvironmentChangeEvent
    Re-bind any @ConfigurationProperties beans in the context
    Set the logger levels for any properties in logging.level.*
    不建议使用轮询方式检查环境变量修改，建议广播到所有实例。
    但是对于另一种bean只是在初始化时注入配置，重新绑定@ConfigurationProperties的bean无法解决问题。

Refresh Scope
    将@RefreshScope注解放在那些仅仅被初始化一次的bean上，或者设置spring.cloud.refresh.extra-refreshable属性
    实现方式为只有bean的方法被调用时才会初始化bean，行为表现和缓存一样，技术上被使用在@Configuration的类上，需要把bean以及其依赖的bean都加上RefreshScope

## Spring Cloud Commons
DiscoveryClient
    @EnableDiscoveryClient自动寻找DiscoveryClient接口的实现，目前已不在需要，会自动发现实现类并自动注册。
    实现HealthIndicator接口健康检查，默认自动配置了DiscoveryClientHealthIndicator。
    DiscoveryClient接口继承了Ordered接口，可以使用多个服务发现示例和设置顺序

ServiceRegistry
    服务注册需要实现ServiceRegistry接口，默认会自动注册，可以设置@EnableDiscoveryClient参数autoRegister=false关闭
    InstancePreRegisteredEvent和InstanceRegisteredEvent事件会被在服务注册前后被触发，使用ApplicationListener可以监听，注意只有当自动注册开启时才会触发。

RestTemplate & WebClient as a Load Balancer Client
    在RestTemplate上配置@LoadBalanced可以实现负载均衡，URI需要使用虚拟主机名（即服务名称，而不是主机名），负债均衡代理推荐使用BlockingLoadBalancerClient。
    负载平衡RestTemplate可以配置重试失败的请求，默认情况是禁用的，通过添加Spring Retry可以实现功能。
    WebClient.Builder上配置@LoadBalanced可以实现负载均衡，URI需要使用虚拟主机名（即服务名称，而不是主机名），负债均衡代理推荐使用ReactiveLoadBalancer，目前只有ribbon的实现。
    WebClient.Builder可以通过ReactorLoadBalancerExchangeFilterFunction来实现。
    ReactiveLoadBalancer支持缓存，如果cacheManager存在，ServiceInstanceSupplier的缓存会存在，缓存服务器负载均衡信息。

## Spring Cloud Config
    Spring Cloud Config为分布式系统中的外部配置提供服务器和客户端支持，服务器存储默认使用git，可以轻松实现基于标签的版本控制。
    使用@EnableConfigServer注解将一个spring boot项目变成一个配置服务器，核心是EnvironmentRepository来构造Environment。
    在客服端使用时需要在bootstrap.properties配置配置中心的url，因为加载配置是在应用程序上下文的引导bootstrap阶段.

## SpringBoot Configuration Processor自动生成spring-configuration-metadata.json文件，build后即生效，
    在配置文件中提示被@ConfigurationProperties修饰的配置类配置中变量的javadoc注释和默认值

## 2.4.x版本配置加载
    spring cloud 移除了bootstrap文件，application.yaml/properties，外部配置使用spring.config.import加载
    在application-<profile>.properties文件中使用spring.config.activate.on-profile指定生效的环境
    在application.yaml/properties使用spring.profiles.active声明环境变量以及application-<dev>.properties仍能生效
    在application.yaml/properties使用spring.profiles.group.{profile}启动指定的配置文件
    也可以使用spring.config.import加载配置文件，可配合使用spring.config.activate.on-profile在不同的环境加载不同的配置
    properties也支持多文档属性了，使用#---开启，yaml文件使用---