## [首页](https://kingkh1995.github.io/blog/)
> Shiro 学习笔记

### Shiro
#### 1.使用方法
    > 集成spring boot，web项目使用shiro-spring-boot-web-starter，自动配置ShiroWebAutoConfiguration和ShiroWebFilterConfiguration。
      实现自定义Realm，自定义ShiroFilterChainDefinition，非单机环境下需要自定义SessionManager，如需使用缓存，自定义CacheManager。
            
#### 2.原理   
    > 实现方式是通过过滤器实现的，默认筛选器实例在DefaultFilter枚举中定义。
    
#### 3.核心组件   
    > Subject：主体，与应用交互的门面
            使用代理模式，所有交互均是交由SecurityManager去实现。

    > Realm：域，可视为安全的数据源，提供认证和授权的数据，可以配置多个，定制AuthenticationStrategy
            AuthenticatingRealm：仅提供认证数据，使用的token类型为UsernamePasswordToken。
            AuthorizingRealm：继承自AuthenticatingRealm，提供认证和授权数据。
    
    > SecurityManager：Shiro的心脏，负责管理所有的操作，实现Authenticator, Authorizer, SessionManager
            DefaultWebSecurityManager：web系统使用，默认使用ServletContainerSessionManager和开启了记住我功能，配置useNativeSessionManager为true则使用DefaultWebSessionManager。

    > SessionManager：会话管理器的顶级接口
            ServletContainerSessionManager：DefaultWebSecurityManager默认使用，直接使用servlet容器的会话管理。
            AbstractValidatingSessionManager：默认开启了session清理定时任务，使用newSingleThreadScheduledExecutor实现，在初次使用session时加载定时任务。
            DefaultWebSessionManager：继承至AbstractValidatingSessionManager，使用类似servlet容器的方式管理自定义的session（自定义session保存在JVM中，sessionId保存在cookie中）。
        
    > SessionDAO：session的crud功能实现
            MemorySessionDAO：DefaultWebSessionManager的默认实现，session类型为SimpleSession，不对外暴露，暴露的是其代理DelegatingSession。
                使用ConcurrentHashMap在jvm中持久化session，并没有判断session是否过期的操作。

    > CacheManager：缓存管理器
            MemoryConstrainedCacheManager：完全使用JVM的方式管理缓存，继承自AbstractCacheManager，使用ConcurrentMap<String, Cache>管理所有缓存，缓存实现则是使用Shiro自己实现的SoftHashMap。
     
    > ShiroWebFilterConfiguration：过滤器自动配置，注入了ShiroFilterFactoryBean，以及使用了spring boot的FilterRegistrationBean，将shiro过滤器order设为最高。
            OncePerRequestFilter：保证每个请求只被处理一次的过滤器，所以Url配置拦截的时候使用LinkedHashMap，定义了preHandle方法执行拦截器逻辑。
            AdviceFilter：继承OncePerRequestFilter，提供类似aop的切面编程功能。
            PathMatchingFilter：继承AdviceFilter，提供了路径匹配功能。
            AccessControlFilter：authc拦截的抽象实现，提供了isAccessAllowed 是否允许访问和onAccessDenied 是否拒绝访问。
            AuthenticationFilter：继承自AccessControlFilter，重写了isAccessAllowed方法，添加了是否已经登录过了的逻辑判断。
            AuthenticatingFilter：继承AuthenticationFilter，提供了自动登录、是否登录请求等方法。
        
