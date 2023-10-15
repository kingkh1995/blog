# [首页](/blog/)

> [Spring Modulith](https://docs.spring.io/spring-modulith/docs/current/reference/html/#appendix)

***

## 使用

### @SpringBootApplication

应用的主包为@SpringBootApplication注解的Application类的包，所有与Application类同层级的包视作独立的模块，而他们的子包则作为各自的内部包。

### @org.springframework.modulith.ApplicationModule

```java
@org.springframework.modulith.ApplicationModule(
  displayName = "inventory", // 子模块名可选，默认为子包名
  allowedDependencies = {"order"} // 指定能直接依赖的子模块，缺省时表示无限制
)
package example.inventory;
```

**只能使用在与Application类同层级子包的package-info.java文件中**，将当前子包标识为模块，当前模块的root包为API包，root包下的类可以在其他模块中被使用；非root包下的类只能在当前模块内使用，同时也无法使用其他模块的类。

### @org.springframework.modulith.NamedInterface

**使用在模块的任意package-info.java文件中**，将当前包标记为命名接口，开放当前子包下的类，可以被其他模块root包下的类使用。

```java
@org.springframework.modulith.NamedInterface("spi")
package example.order.spi;
```

```java
@org.springframework.modulith.ApplicationModule(
  allowedDependencies = "order::spi" // 在其他模块中通过 <模块名>::<命名接口名> 形式引用
)
package example.inventory;
```

***

## 验证

Spring Modulith通过测试验证模块结构的正确性。

### ApplicationModules

```java
@Test
void verifiesModularStructure() {
    var modules = ApplicationModules.of(Application.class).verify();
    // 生成的文件位于/target/spring-modulith-docs目录下
    new Documenter(modules)
            .writeModulesAsPlantUml()
            .writeIndividualModulesAsPlantUml();
}
```

添加测试方法，直接使用ApplicationModules的API进行验证，并可以生成模块结构的PlantUML图。

### @ApplicationModuleTest

```java
package example.inventory

@ApplicationModuleTest(
    value = ApplicationModuleTest.BootstrapMode.ALL_DEPENDENCIES, // 会在控制台输出模块结构信息（DEBUG日志）
    verifyAutomatically = true) // 默认开启模块验证
class InventoryServiceTest {
}
```

在模块的测试类上声明@ApplicationModuleTest注解进行验证。

### @MockBean

由于Spring Bean实例化时使用的类可能是模块边界之外的类，需要使用@MockBean注解该Bean，以模拟该目标Bean。

***

## 模块事件驱动

模块之间使用事件驱动，能去除模块之间的耦合，但这会带来一致性问题，需要使用事件日志，即通过持久化事件日志保证一致性，支持多种持久化方式。

### @org.springframework.modulith.ApplicationModuleListener

```
@Async
@Transactional(propagation = Propagation.REQUIRES_NEW)
@TransactionalEventListener
```
为EventListener添加@ApplicationModuleListener注解，默认异步，开启新事务。

### EventPublicationRepository

需要提供EventPublicationRepository的实现，作为事件发布的持久化方式。

### spring-modulith-starter-jdbc

使用jdbc持久化需要添加以下依赖，可配置参数自动创建事件发布表。
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-jdbc</artifactId>
</dependency>
```

```
# 自动创建event_publication表
spring.modulith.events.jdbc.schema-initialization.enabled=true
```

