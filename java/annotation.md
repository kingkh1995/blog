# [首页](/blog/)

> version: **jdk17**

***

## 注解

**注解即是特殊的接口。**

1. 可以声明在其他注解上；
2. 无继承列表；
3. 只能定义无参方法，方法名作为属性名，返回值作为属性类型，使用default关键字指定属性默认值；
4. 在声明注解时，如只指定了一个value属性则可以省略掉属性名；
5. 可以定义常量。

***

## interface **Annotation**

为所有注释的父接口，由编译器添加继承关系。

### annotationType() & getClass()

```java
// 由编译器实现，返回注解接口的class对象。
Class<? extends Annotation> annotationType();
```

**因为注解为接口而不是类，故通过反射获取到的注解对象是JDK的代理对象，getClass()返回的类对象为JDK代理类的类对象。**

- getClass()返回类型为Class<? extends **A**>，可以直接向上转型为Class<? extends Annotation>；
- 想要获取到注解上的注解信息，只能使用annotationType()返回的Class对象。

***

## 元注解

用于声明在注解上的注解，**元注解上也都有声明元注解**。

```java
// 元注解上声明的元注解
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
```

### @Documented

声明了@Documented的注解，在生成javadoc时会被记录。

### @Retention

用于指定注解的保留策略，属性为**RetentionPolicy**。

```java
public enum RetentionPolicy {
    SOURCE, // 只保留在源代码中，会被编译器丢弃；
    CLASS, // 会被编译器记录在字节码文件中，但运行时会被丢弃；
    RUNTIME // 运行时会保留，可以通过反射机制获取到注解信息。
}
```
*CLASS级别是@Retention缺省时的默认级别，一般很少使用，主要是供字节码解析器解析。*

### @Target

用于指定注解的使用范围，属性为**ElementType\[\]**，@Target缺省时注解可以被声明在任何元素上。

```java
public enum ElementType {
    TYPE, // 类、接口（包括注解）、枚举、记录
    FIELD, // 属性
    METHOD, // 方法
    PARAMETER, // 方法参数
    CONSTRUCTOR, // 构造器
    LOCAL_VARIABLE, // 局部变量
    ANNOTATION_TYPE, // 注解
    PACKAGE, // package-info文件
    TYPE_PARAMETER, // 类型变量
    TYPE_USE, // 任何使用了类型的位置（如Jakarta注解）
    MODULE, // module-info文件
    RECORD_COMPONENT; // @since 16
}
```

```java
// 记录组件上声明注解
public record MyRecord(@Anno Object field) {
}
```

***在记录组件上声明注解时支持四种目标：***
- *FIELD：会被传递到属性上；*
- *METHOD：会被传递到属性的获取方法上；*
- *PARAMETER：会被传递到构造方法的参数上；*
- *RECORD_COMPONENT：保留在记录组件上。*

### @Inherited

**仅仅意味着子类可以通过反射（getAnnotation方法和getAnnotations方法）获取到父类上声明的Inherited的注解**。

### @Repeatable

使注解可以在同一个元素上声明多次，**需要指定一个容器注解**。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.FIELD})
@Repeatable(RepeatableAnnoCol.class)
public @interface RepeatableAnno {
}

@Retention(RetentionPolicy.RUNTIME) // 保留长度必须超过Repeatable注解
@Target(ElementType.TYPE) // 目标必须是Repeatable注解目标的子集
public @interface RepeatableAnnoCol {
    RepeatableAnno[] value();
}
```

```java
// 反射获取Repeatable注解信息的正确方式
RepeatableAnno[] annotations = A.class.getAnnotationsByType(RepeatableAnno.class);

// 通过原注解，如果声明了多次则返回null
RepeatableAnno annotation = A.class.getAnnotation(RepeatableAnno.class); 

//通过容器注解，如果只声明了一次则返回null
RepeatableAnnoCol annotationCol = A.class.getAnnotation(RepeatableAnnoCol.class); 
```
即如果Repeatable注解被声明了多次，则会被编译为以其容器注解的方式进行声明，**同时多次声明时如容器注解不支持该目标会无法编译通过**。

***

### **Spring注解 (TODO)**

- AnnotationUtils：
  - findAnnotation方法：能从类自身、父类、接口以及注解上获取注解信息（**lang包下的注解除外，如@FunctionalInterface**），且无需@Inherited元注解。

- @Validated、@RequestMapping等功能性注解：只要能通过AnnotationUtils.findAnnotation方法获取注解到即开启注解相应功能。

- @Component等IOC注解：需要直接或间接（声明在其他注解上）的方式声明在类上后才能被IOC容器管理，即无法被子类继承。

***