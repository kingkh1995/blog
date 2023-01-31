# [首页](/blog/)

> **SPI（Service Provider Interface）**

***

## Java

### 步骤

1. 定义接口【SPI】；
2. 创建接口的实现类【ASPI】【BSPI】；
3. 在【META-INF/services/】目录下创建接口全限定名命名的文件【jdk.spi.SPI】；
4. 文件内容为实现类的全限定名；
    ```
    com.aa.ASPI
    com.bb.BSPI
    ```
5. 使用ServiceLoader加载所有的文件内定义的实现类。
    ```java
    ServiceLoader<SPI> spis = ServiceLoader.load(SPI.class); // 返回代理对象，懒加载方式。
    ```

### public final class **ServiceLoader\<S\>** implements Iterable\<S\>

***

## Spring

### 步骤

1. 定义接口和实现类；
2. 只读取【META-INF/spring.factories】文件；
    ```
    jdk.spi.SPI=com.aa.ASPI,com.bb.BSPI
    ```
3. 使用SpringFactoriesLoader加载实现类（非懒加载）。
    ```java
    List<SPI> spis= SpringFactoriesLoader.loadFactories(SPI.class, Thread.currentThread().getContextClassLoader());
    ```
   
### **SpringFactoriesLoader**

***

## Dubbo

### 步骤

1. 定义接口，需要使用dubbo的注解@SPI修饰；
2. 支持三个拓展点：【META-INF/dubbo/internal】内部拓展点；【META-INF/dubbo】自定义拓展点；【META-INF/services】兼容java；
3. 同样使用接口全限定名命名的文件，文件内容为键值对方式；
    ```
    a=com.aa.ASPI
    b=com.bb.BSPI
    ```
4. 使用ExtensionLoader加载文件内定义的实现类，通过键获取。
    ```java
    ExtensionLoader<SPI> spis = ExtensionLoader.getExtensionLoader(SPI.class);
    SPI a = spis.getExtension("a");
    SPI b = spis.getExtension("b");
    ```

#### @Adaptive

自适应扩展实现类，可以修饰类（接口，枚举）和方法，优先级高于@SPI，**使用时要求方法入参必须包含org.apache.dubbo.common.URL**。

```java
@SPI("impl1")
public interface SimpleExt {
    @Adaptive({"key1", "key2"})
    String yell(URL url, String s);
}
```
先判断URL种的key1参数，不存在则判断key2，均不存在则使用impl1对应的默认实现。

#### @Activate

自动激活扩展类，优先级最高，使用在实现类上，包含group、value、before、after、order五个参数。

### **ExtensionLoader**

***