# [首页](/blog/)
> Shiro 学习笔记

***

## Subject

**主体**，操作的门面，可以理解为一个用户，其操作都是交由SecurityManager实现，持有Session。

通过SubjectFactory创建，一般不需要自定义，使用默认实现WebDelegatingSubject即可。

***

## Realm

**域**，提供认证和授权信息的安全数据源，需要自定义，可以配置多个。

- AuthenticationToken：认证凭证，可以理解为账户和密码 。
- AuthenticationStrategy：认证策略，默认AtLeastOneSuccessfulStrategy。
- Authenticator：认证者，拿到认证凭证按认证策略去域中进行验证。
- Authorizer：授权者，认证通过后，对用户的许可操作进行判断。

***

## Session相关

### SessionManager

**会话管理器**，创建、保存和清理用户的会话信息。

- ServletContainerSessionManager：操作servlet容器的session。
- NativeSessionManager：本地管理会话，指由程序控制会话的声明周期。
- AbstractValidatingSessionManager：实现了NativeSessionManager，添加了会话过期清理能力，默认清理执行类为ExecutorServiceSessionValidationScheduler。
- DefaultWebSessionManager：继承至AbstractValidatingSessionManager，使用类似servlet容器的方式管理自定义的会话（*会话保存在内存中，sessionId保存在cookie中。*）

### Session

- HttpServletSession：直接代理servlet容器的会话
- DelegatingSession：可对外暴露的会话，操作全部委派给持有NativeSessionManager。

### SessionFactory

会话工厂，只有需要自己管理会话信息才会使用。

### SessionDAO

会话数据访问接口，只有需要自己管理会话信息才会使用。

- CachingSessionDAO：配合SimpleSession使用，可缓存会话，需要自行实现数据的管理。

***

## SecurityManager

**Shiro的心脏**，管理所有的操作，实现了Authenticator, Authorizer, SessionManager接口。

DefaultWebSecurityManager：默认使用ServletContainerSessionManager和开启了记住我功能。

***

## CacheManager

缓存管理器，实现CacheManagerAware接口即开启功能，默认实现为内存缓存，使用了shiro的SoftHashMap类。

***

## EventBus

消息总线，实现EventBusAware接口即开启功能，默认实现是DefaultEventBus。

***

## Filter

shiro的实现是基于过滤链的。

- ShiroFilterFactoryBean：过滤器配置bean。
- SpringShiroFilter：spring环境下shiro过滤器的入口，通过注入FilterRegistrationBean实现，内部会去执行定义好的过滤器链。
- OncePerRequestFilter：保证每个请求只被处理一次的过滤器，所有过滤器均继承该抽象类。
- AdviceFilter：继承OncePerRequestFilter，抽象出切面方法。
- PathMatchingFilter：继承AdviceFilter，实现了路径匹配功能。
- AccessControlFilter：继承AdviceFilter，拦截类过滤器的抽象类，抽象了isAccessAllowed和onAccessDenied方法
- AuthenticatingFilter：继承自AccessControlFilter，重写了isAccessAllowed方法，添加了已登录逻辑判断。
- AuthenticationFilter：继承AuthenticationFilter，实现了自动验证功能。

***
        
