# [首页](/blog/)

***

## IOC

### @Component

注解在类上，@Repository、@Service、@Controller都注解了@Component注解，属于其衍生注解，只有语义区别。@ComponentScan（@SpringBootApplication上注解）用于扫描指定路径下含有@Component注解的类。

### @Bean

注解在方法上，可以指定initMethod和destroyMethod，适用于装配第三方库的类。

### @Configuration

用于配置类，也注解了@Component注解，proxyBeanMethods参数表示是否代理@Bean方法，即@Bean方法多次调用只返回同一个单例对象，**默认开启，建议关闭，能提高启动速度**。

- @ConfigurationProperties：使用指定的前缀从properties文件中绑定属性，可以注解在类和方法上，**需要被IOC容器管理**。
- @EnableConfigurationProperties：基于@Import，导入指定的@ConfigurationProperties注解的类。
  
### **@Import**

## FacotryBean、BeanFactory、ApplicationContext、
- BeanFactory：工厂模式最顶级接口，为延迟加载，适用于多实例模式
- ApplicationContext：BeanFactory的子接口，预先加载，适合单例模式，支持国际化，资源文件读取，时间传播，

## Bean创建过程、生命周期、作用范围


### 创建过程

### 生命周期

### @Scope

配合@Bean和@Component使用，指定bean的作用域。可以新增，使用ConfigurableBeanFactory的registerScope方法注册，或添加CustomScopeConfigurer，**除了Singleton和Prototype之外都会生成Scope代理对象**。

- value：等同于scopeName
  - singleton：单例模式，默认作用域；
  - prototype：多例模式和，每次使用都会创建一个实例；
  - request：每次HTTP请求都会创建一个实例，仅在当前HTTP请求内有效；
  - session : 为每个HTTP会话创建一个新的实例，仅在当前会话内有效；

- proxyMode：代理模式
  - DEFAULT：按照默认策略
  - NO：无代理 
  - INTERFACES：基于接口代理
  - TARGET_CLASS：基于类代理（cglib）

- @RefreshScope：Spring Cloud Bus中使用，当配置中心配置更新后会通知bean刷新，代理模式为TARGET_CLASS，对应的Scope类型为RefreshScope，实现为从缓存获取bean。

### **如何解决循环依赖**

spring boot 2.6 版本默认禁止了循环依赖。

***

## DI

### 注入方式

- 构造器注入，优点是对象构造完成后就立马可用，且属性能被设置为final，缺点是构造器无法被继承，无法设置默认值，维护困难。
- setter注入，最推荐的方法，灵活度最高，可以被继承重写，缺点就是属性无法为final，创建完对象无法立即就绪。
- 字段注入，已经不再推荐此方案，扩展性差。

### @Autowired

默认匹配方式byType，存在多个实现类时，则会使用byName方法，要指定具体的bean可以再添加一个@Qualifier注解或在实现类上添加@Primary注解。

### @Resource

属于jdk依赖注入注解，默认byName，其次byType，也可通过name和type属性具体指定。

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
  

### AOP执行过程

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
- pointcut组合：支持&&、||、!

***

## Spring mvc处理流程
DispatcherServlet是Servlet容器的bean，

## Spring Data Jpa实现
通过JdkDynamicAopProxy创建代理对象，默认通过Hibernate执行数据库操作。
