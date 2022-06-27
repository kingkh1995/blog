# [首页](/blog/)

***

## IOC

### IOC容器注解

- @Component：注解在类上，@Repository、@Service、@Controller都注解了@Component注解，属于其衍生注解，只有语义区别。@ComponentScan（@SpringBootApplication上注解）用于扫描指定路径下含有@Component注解的类。

- @Bean：注解在方法上，可以指定initMethod和destroyMethod，适用于装配第三方库的类。

- @Configuration：用于配置类，也注解了@Component注解，proxyBeanMethods参数表示是否代理@Bean方法，即@Bean方法多次调用只返回同一个单例对象，**默认开启，建议关闭，能提高启动速度**。

- @ConfigurationProperties：使用指定的前缀从properties文件中绑定属性，可以注解在类和方法上，**需要被IOC容器管理**。

- @EnableConfigurationProperties：基于@Import，导入指定的@ConfigurationProperties注解的类。
  
- **@Import**：用于导入Bean，ImportSelector、DeferredImportSelector、ImportBeanDefinitionRegistrar必须依赖@Import使用，但@Import可以单独使用。

### FacotryBean、BeanFactory、ApplicationContext

- FactoryBean：属于设计模式，是创建Bean的一种方式，核心方法是getObject()。

- BeanFactory：即IOC容器的最顶级接口，负责配置、创建、管理Bean。
  
- ApplicationContext：BeanFactory的子接口，预先加载Bean，还支持国际化、资源文件读取和事件传播。

### @Scope

配合@Bean和@Component使用，指定bean的作用域。可以新增，使用ConfigurableBeanFactory的registerScope方法注册，或添加CustomScopeConfigurer，**除了Singleton和Prototype之外都会生成Scope代理对象**。

- value：等同于scopeName
  - singleton：单例模式，默认作用域；
  - prototype：多例模式，每次使用都会创建一个实例；
  - request：每次HTTP请求都会创建一个实例，仅在当前HTTP请求内有效；
  - session : 为每个HTTP会话创建一个新的实例，仅在当前会话内有效；

- proxyMode：代理模式
  - DEFAULT：按照默认策略
  - NO：无代理 
  - INTERFACES：基于接口代理
  - TARGET_CLASS：基于类代理（cglib）

- @RefreshScope：Spring Cloud Bus中使用，当配置中心配置更新后会通知服务器刷新作用域为refresh的bean，代理模式为TARGET_CLASS，对应的Scope类型为RefreshScope，实现为从缓存获取bean。

### **Bean的生命周期**

1. Bean容器（通常是ApplicationContext）先创建所有的PostProcessor实例；
2. 找到bean的定义，通过配置文件或扫描类文件；
3. 使用反射创建Bean实例，并注入属性；
4. 如果实现了BeanNameAware等Aware接口则调用对应方法；
5. 如果容器内存在BeanPostProcessor，则执行对应的postProcessBeforeInitialization()方法；
6. 如果bean实现了InitializingBean，则执行afterPropertiesSet()方法；
   - SmartInitializingSingleton会在所有的单例模式Bean创建完成后执行，在InitializingBean之后。
7. 执行Bean配置的init方法；
8. 执行BeanPostProcessor的postProcessAfterInitialization()方法；
9. 销毁Bean时，执行DestructionAwareBeanPostProcessor的postProcessBeforeDestruction()方法；
10. 如果实现了DisposableBean接口，则执行destroy()方法；
11. 执行Bean配置的destroy方法。

- *@PostConstruct和@PreDestroy是jdk规范，在**InitDestroyAnnotationBeanPostProcessor**的BeforeInitialization和BeforeDestruction阶段执行。*

***

## DI

### 注入方式

- 构造器注入：对象构造完成立马可用
  - 优点：属性可以被设置为final安全性高；
  - 缺点：比较繁琐，不利于拓展和继承。

- setter方法注入：对象构造完成后再重新进行注入
  - 优点：灵活度最高，适合拓展和继承；
  - 缺点：属性不能是final，会导致循环依赖无法被发现。
  
- 字段注入：不推荐，扩展性差，与DI框架强耦合。

### @Autowired

后置处理器为AutowiredAnnotationBeanPostProcessor，默认匹配方式byType，存在多个实现类时，则会使用byName方法，要指定具体的bean可以再添加一个@Qualifier注解或在实现类上添加@Primary注解，。

### @Resource

属于jdk依赖注入注解，后置处理器为CommonAnnotationBeanPostProcessor，默认byName，其次byType，也可通过name和type属性具体指定。缺点是无法注解在方法参数上，且不支持配置required。

### @Lazy

设置为延后加载，在依赖注入时为Bean生成代理类，首次被调用时才创建Bean，**可用于解决构造器注入时的循环依赖问题**。

### **如何解决循环依赖**

1. Bean实例化完成后，会往三级缓存中加入当前‘半成品’Bean，实际上是添加一个ObjectFactory函数，其getObject方法实现为顺序使用所有SmartInstantiationAwareBeanPostProcessor类型的后置处理器的getEarlyBeanReference方法处理半成品Bean；
2. 然后开始依赖注入，从各级缓存中查找，如果不存在则递归创建依赖的Bean；
3. 存在循环依赖时，之后处理的Bean可以从三级缓存获取到ObjectFacotry，通过其getObject方法获取到之前处理的Bean的半成品注入；
4. Bean依赖注入完成后，标记为完成并加入一级缓存中；
5. 递归回退依次完成所有Bean的创建。

***

## AOP

### **代理**

Spring AOP默认使用Jdk动态代理，如果没有实现接口则使用cglib动态代理，**只是使用了AspectJ的注解**。

#### 动态代理

**运行时增强**，在程序运行期间创建代理类的字节码文件。

- jdk动态代理：基于接口实现，核心为InvocationHandler，生成一个实现代理接口的匿名类，方法执行使用反射机制（**JDK8后效率与cglib不再有差距**）。
    ```    
    Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)
    ```

- cglib动态代理：基于子类继承（**所以无法处理final**），核心为MethodInterceptor，使用字节码处理框架 ASM 动态生成被代理类的子类。
    ```    
    Enhancer.create(Class type, MethodInterceptor callback)
    ```

#### 静态代理

**编译时增强**，效率高，代理类在编译阶段生成，程序运行前就已经存在。

- AspectJ静态代理：使用特定的编译器针对特定的语法，在编译期间生成代理类的字节码文件。

### @Pointcut

- execution(方法表达式)：匹配方法的执行
- within(类型表达式)：匹配target（目标对象）的类型
  - **target.getClass().equals(X.class)**
- this(类型全限定名)：匹配proxy（代理对象）的具体类型
  - **X.class.isAssignableFrom(proxy.getClass())**
  - **注意**：基于接口的代理则需要指定为被代理类的接口才能生效
- target(类型全限定名)：匹配target的具体类型
  - **X.class.isAssignableFrom(target.getClass())**
- args(参数类型列表)：匹配指定参数列表的方法
- @within(注解类型)：匹配方法归属类型上的注解
  - **method.getDeclaringClass().getAnnotation(A.class)**
- @target(注解类型)：匹配target上的注解
  - **target.getClass().getAnnotation(A.class)**
- @args(注解类型的参数列表)：匹配指定注解类型的参数列表的方法，即参数类型含有指定的注解
- @annotation(注解类型)：匹配方法上的注解
- bean(bean名称)：匹配IOC容器中指定的bean
- pointcut引用：指向具体的方法，使用该方法定义的pointcut
- pointcut组合：支持&&、\|\|、!

### **AOP下依赖循环问题如何处理**

AOP的AbstractAutoProxyCreator为SmartInstantiationAwareBeanPostProcessor类型，**其getEarlyBeanReference方法实现为使用半成品Bean创建代理对象并返回，同时还会将其加入二级缓存中**；在其postProcessAfterInitialization方法中会为Bean创建代理对象，如果二级缓存中已经存在代理对象则直接获取。

若A、B互相依赖，A先创建，则B注入A时，注入的是通过getEarlyBeanReference方法创建的半成品A的代理对象；B依赖注入完成后在postProcessAfterInitialization方法中创建了B的代理对象，然后加入一级缓存内；递归回退到A的依赖注入后，A直接从一级缓存中获取到B的代理对象注入；A依赖注入完成后，执行postProcessAfterInitialization方法时，直接从二级缓存中获取到自己的代理对象，然后加入一级缓存中。

***

## Spring MVC

### DispatcherServlet

是属于Servlet容器的Bean。

### 流程

1. 客户端请求会被提交到DispatcherServlet（前端控制器）；
2. DispatcherServlet请求一个或多个HandlerMapping（处理器映射器），并返回一个执行链（HandlerExecutionChain）；
3. DispatcherServlet将执行链返回的Handler信息发送给 HandlerAdapter（处理器适配器）；
4. HandlerAdapter根据Handler信息找到并执行相应的Handler（即Controller）；
5. Handler执行完毕后返回给HandlerAdapter一个ModelAndView对象，HandlerAdapter再将其返回给DispatcherServlet；
6. DispatcherServlet接收到ModelAndView对象后，请求 ViewResolver（视图解析器）对进行解析；
7. ViewResolver根据View信息匹配到相应的视图结果，并返回给 DispatcherServlet；
8. DispatcherServlet将Model交给接收到的View进行渲染，然后返回给客户端。

***

## Spring事务

### 编程式事务

- TransactionManager：事务管理器marker接口。

- PlatformTransactionManager：平台事务管理器接口，定义了getTransaction、commit以及rollback操作。

- TransactionDefinition：事务属性，包括隔离级别、传播行为、超时事件、回滚规则以及是否只读。
  - PROPAGATION_REQUIRED：**默认**，存在事务则加入，不存在则创建新事务；
  - PROPAGATION_REQUIRES_NEW：创建新事物，挂起外部事务，两者互不影响。
  - PROPAGATION_NESTED：创建新事务，如果存在外部事务则作为其嵌套事务，**外部事务会影响当前事务，但当前事务不会影响外部事务**；
  - PROPAGATION_MANDATORY：要求当前必须存在事务，否则抛出异常；
  - PROPAGATION_SUPPORTS：存在事务则加入，否则以非事务方式执行；
  - PROPAGATION_NOT_SUPPORTED：以非事务方式执行，挂起外部事务；
  - PROPAGATION_NEVER：当前不允许存在事务，否则抛出异常。

- TransactionStatus：事务状态，用于创建savepoint以及flush操作。

- TransactionTemplate：封装了事务操作，使用PlatformTransactionManager和TransactionDefinition创建。

- TransactionSynchronizationManager：工具类，事务同步管理器，用于监听Spring的事务操作。

### 声明式事务

基于AOP实现，故需要被spring管理、只能拦截public方法、内部调用无法生效。

- @Transaction：用于注解在类和方法上开启事务，若不配置rollbackFor参数，则只会在RuntimeException时回滚；

- @GlobalTransaction：Seata的分布式事务开启注解。

***

## Spring Data Jpa

通过JdkDynamicAopProxy创建代理对象，默认通过Hibernate执行数据库操作。

- Repository：marker接口；
- CrudRepository：仅crud，Spring Data Redis则使用该接口；
- PagingAndSortingRepository：在CrudRepository基础上支持分页排序；
- QueryByExampleExecutor：单独接口，支持Example方式查询；
- JpaSpecificationExecutor：单独接口，支持Specification查询；
- JpaRepository：标准Jpa操作接口，继承PagingAndSortingRepository和QueryByExampleExecutor。

***
