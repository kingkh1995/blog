# [首页](/blog/)

> version: **jdk17**

***

## Object

- native int hashCode()：返回对象的哈希值，**哈希值和对象地址有一定关联，但并不一定如此**，具体取决于运行时库和JVM的具体实现。
  > Boolean类型返回固定值；Byte、Short、Integer、Character直接返回int值；Float返回按int类型读取的值；Long和Double类型返回高32位和低32异或的结果；**String类型使用以31作为因子的除留余数法计算字符数组的所有字符，并且有软缓存（hash属性）**。

- boolean equals(Object obj)：默认实现为使用==对比。**重写equals方法时需要遵守hashCode方法的常规协定（散列集合是基于此协定设计的），equals方法对比相等的对象必须具有相等的哈希值，哈希值不等的对象equals方法对比必然不等**。

- **protected** native Object clone() throws CloneNotSupportedException：默认实现为浅拷贝，一个类必须实现Cloneable接口才能调用clone方法，否则会抛出CloneNotSupportedException，**注意Object类未实现Cloneable**。
  > **数组可以调用clone方法且访问限定修饰符为public**，推荐使用Arrays工具类的copyOf方法复制数组。

- protected void finalize() throws Throwable：**jdk9开始已经被标记为废弃。**

- final native notify() & notifyAll()：notify唤醒正在此对象监视器上等待的任意的一个线程，notifyAll唤醒正在此对象监视器上等待的所有线程。**必须在同步方法或同步块内使用，被唤醒的线程仍然需要竞争到同步锁才可以恢复执行**。

- final native void wait(long timeoutMillis)：线程状态变为TIMED_WAITING，等待被唤醒或者达到超时时间后被自动唤醒，**必须在同步方法或同步块内使用，当前线程会释放同步锁**。

- final void wait()：wait(0L)，线程状态变为WAIT，一直等待直到被唤醒或者被打断。

***
    
## String

### 属性

- final byte coder：字符编码方式，0表示Latin1，1表示UTF-16BE。**jdk9开始支持字符压缩，之前默认使用UTF-16BE编码**，如果字符全部在Latin1能表示的范围内，那么会使用Latin1编码，否则使用UTF-16BE。

- final byte[] value：**jdk9开始使用byte数组存储字符，之前使用char数组**。Latin1编码使用一个byte值，UTF-16BE编码使用两个连续的byte值。

- int hash：哈希值缓存。

- boolean hashIsZero：标识哈希值是否为0，因为hash属性默认值也为0。

### 字符编码

- Latin1 & ASCII：**Latin1是iso-8859-1的别名，Latin1和ASCII都是单字节字符**；ASCII定义了128个字符，只使用了后7位最高位默认为0；Latin1定义了256个字符，使用了全部8位，能完全向下兼容ASCII。

- Unicode字符集：是一种通用字符集，它的编码空间可以划分为17个平面（plane），每个平面包含65,536个码位（code point），**第一个平面称为基本多语言平面（BMP，包括中文）**，其他平面称为辅助平面。Unicode字符集只是指定了字符的编号，但是却有多种编码方式去表示这个编号。

- utf-16：Unicode字符集规定的标准编码实现，固定使用两个字节去表示BMP内的字符，而BMP之外的字符则需要四个字节去表示。

- utf-8：可变长字符编码，使用1到4个字节表示一个Unicode字符。**字节码文件使用的编码**，对于ASCII字符，utf-8编码能完全兼容，即只需要一个字节，但是表示一个中文字符utf-8却需要三个字节。
  > 代码使用的字符几乎都是ASCII字符，故字节码文件使用utf-8编码相比于utf-16能大大节约空间，因为utf-16表示一个ASCII字符需要两个字节。

### 构造方法

- String(String original)：创建对象，并复用value数组。
  > **String s = new String("abc")，如果字符串常量池中不存在abc，则需要创建两个String对象，但是只会创建一个value数组。**

- 其他构造方法均会创建新的value数组，因为要保证value数组是stable的。

- String(byte bytes[], Charset charset)：**将byte数组按给定的编码方式解码为字符串**，charset参数使用StandardCharsets类中的常量。

### 实例方法

- char charAt(int index) & int codePointAt(int index)：charAt获取BMP内的unicode字符，所以返回两个字节的char；而codePointAt能获取BMP外的unicode字符，所以返回四个字节的int。

- getBytes()：使用默认的utf-8编码或指定的编码方式将字符串编码为字节数组。
  > 指定编码为utf-16时会多出两个字节，因为需要使用额外的两个字节来标志字节序（FEFF或FFFE）。

- isBlank() & strip()：**jdk11新增**，isBlank()判断是否是空字符串，strip()移除首尾的空字符，都是使用Character.isWhitespace()方法识别Unicode空白字符。
  > **strip方法移除所有Unicode空白字符，而trim方法只移除空格、tab键、换行符。**

- length()：value.length >> coder()。**需要知道对于BMP外的字符是占用四个字节的，所以返回值并不一定代表字符串的长度**。

- split(String regex, int limit)：regex为正则表达式，如果正则表达式的特殊字符需要按普通字符匹配则需要使用\\\\转义；limit大于0时拆分后数组长度不能超过limit。

- formatted(Object... args)：jdk15新增，使用自身作为模式字符串生成格式化字符串，与String.format()相同，每次都会创建一个新的Formatter。
  > matches(String regex)每次也都会创建一个Pattern对象，所以这两个方法都不建议多次调用。

***

## AbstractStringBuilder

可变字符串的基类，内部实现与String基本相同。**使用一个int类型的属性count来记录字符串的实际长度；扩容操作会先考虑扩容为原容量的两倍加2，如果不足则会直接扩容为能刚好满足的容量。**

### **StringBuilder**

非线程安全，默认容量16。

### StringBuffer

线程安全，所有方法都为同步方法，默认容量16，toString方法有缓存。

### **要点**

- 不存在构造方法AbstractStringBuilder(char c)，你可以这么写并且编译会通过，因为实际上执行的是构造方法AbstractStringBuilder(int capacity)。

- 连续使用 + 拼接字符串会被优化为使用StringBuilder。

***

## 包装类

- Boolean直接使用常量创建。

- Byte、Short、Integer、Long都有自己的私有缓存内部类，为-128到127的区间预先加载了包装类缓存。
  > 只有Integer可以通过JVM参数调整缓存区间的上限。

- Character为ASCII字符添加了缓存，即值区间为0-127。

- valueOf方法返回包装类（**推荐使用该静态工厂方法创建包装类**），parse方法返回基本数据类型。

- signum方法用于输出数字的符号，1：正数，0：0，-1：负数。

- toUnsignedString方法用于将整形转化为无符号整形，**无符号整形即二进制表示的最高位不再视为符号位**。

***

## BigDecimal

有效数字使用BigInteger保存，使用scale属性记录小数点位，可以指定MathContext（精度和舍入模式），默认MathContext.UNLIMITED（精度0及HALF_UP舍入），精度表示有效数字的最大长度，必须大于等于0，0则表示不限制。

### 创建对象

- **不要使用new BigDecimal(double val)，而是使用BigDecimal valueOf(double val)**。

- **float应该转为字符串并使用new BigDecimal(String val)，不能转为double。**。

- **整数应该使用valueOf()方法创建，特殊值能利用到缓存。**

### 方法

- stripTrailingZeros()：去除所有尾随零，返回新的对象，可能转为科学计数法表示。
- toString()：有软缓存，返回MathContext处理过后的格式。
- toPlainString()：数字原始文本格式。
- toEngineeringString()：有必要时使用工程计数法格式输出。

***

## Enum

所有枚举的基类，是绝对的单例模式，重写了clone方法，不允许被clone，直接抛出CloneNotSupportedException。

**如果valueOf方法找不到枚举值会抛出IllegalArgumentException。**

***

## ClassLoader

- BootstrapClassLoader：启动类加载器，为JVM内置的加载器，无法被获取，用于加载JDK的APi类（java_home/jre/lib目录下）；

- PlatformClassLoader：替代了拓展加载器，用于加载Java SE平台API及其实现类、特定的JVM运行时所需的类等；

- SystemClassLoader：系统加载器，也称为应用加载器，用于加载classpath下的所有类。

### loadClass方法流程：

  1. 从已加载类中查找（JVM方法区），不存在则需要获取同步锁，锁对象使用ConcurrentHashMap保存，键为className，值为new Object()；
  2. 由于双亲委派模型，如果父加载器存在则先尝试使用父加载器加载（也会继续向上委托），否则尝试使用BootstrapClassLoader加载；
  3. 如都无法加载，最后才调用自身findClass方法加载。

### 要点

- 数组的类对象不是由类加载器创建的，而是运行时根据需要自动创建的；
- 对象数组的类加载器与其元素的类加载器相同；
- 基本数据类型和基本数据类型数组没有类加载器。

### JAVA主动破坏双亲委派模型

1. 1.2才新增的双亲委派的逻辑在loadClass方法中，为了向前兼容不是final方法，只需要重写即可破坏；
2. JNDI、JDBC等在classpath路径下，因此为Thread对象增加了getContextClassLoader()方法，启动类加载器使用该方法获取应用加载器以加载用户的具体实现；
3. 引入模块化后，新增final loadClass(Module module, String name)方法，其实现为直接使用findClass方法加载，然后判断类所属模块是不是给定模块，相同才返回。

***

## Throwable

所有Error和Exception的父类，RuntimeException是Exception的子类，**RuntimeException与其子类之外的其他Exception称为受检查异常**，方法如果抛出受检查异常，调用方必须使用try-catch捕获异常或者继续向上抛出。

***

## System

- nanoTime()：用来精准计时，不能用于计算时间。**返回从某一固定但任意的时间算起的纳秒数（或许从以后算起，所以该值可能为负）。**

- identityHashCode()：native方法，返回对象的标识哈希值，即Object的默认hashcode方法计算出的值。

***

## Runtime

JVM的运行时环境，饿汉式单例模式，**使用Runtime.getRuntime()获取实例**。

- availableProcessors()：返回JVM可以使用的处理器数量（逻辑处理器）。

- addShutdownHook(Thread hook)：为JVM添加一个钩子线程，**使用kill -9 pid则钩子线程无法执行**。

- exit(int status)：退出JVM，**如果使用非零的状态码则表示异常终止**。

- gc()：**不要显式调用，因为完全是不可控的。**

***

## Annotation

- **注解实际上是接口，隐式继承自Annotation接口**，故注解无法继承其他接口或注解，但是可以被接口继承以及被类实现；
- 注解中如果声明了value参数，则使用时可以忽略参数名value，通过default指定参数默认值；
- 注解中可以定义常量，但是无法定义default方法。

### 元注解

负责注解其他注解，元注解本身也都有被元注解注解。

```java
// 元注解上注解的元注解
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
```

- @Documented：表示注解会被包含在javadoc中。

- @Target：指定注解的使用范围，值为ElementType枚举，如果注解上不存在该元注解，则注解可以用于任何元素上。
  - TYPE：注解可以被使用在类、接口、注解以及枚举上；
  - TYPE_PARAMETER：注解可以被使用在类型变量上；
  - TYPE_USE：注解可以被使用在使用了类型的任何位置（如jakarta注解）。

- @Retention：指定注解的保留级别，值为RetentionPolicy枚举，如果注解上不存在该元注解，则注解保留级别默认为CLASS。
  - SOURCE：注解会被编译器丢弃；
  - CALSS：注解会被编译器记录在class文件中，但虚拟机运行时不保留。
  - RUNTIME：注解会被编译器记录，同时虚拟机运行时也会保留，所以可以通过反射机制读取到注解信息。

- @Inherited：表示子类可以从父类中继承此注解。**仅仅意味着子类class对象可以通过getAnnotation方法获取到父类的此注解**。

- @Repeatable：表示此注解可以在同一个元素上使用多次，需要一个容器注解配合实现。

  ```java
  @Retention(RetentionPolicy.RUNTIME)
  @Target({ElementType.TYPE, ElementType.FIELD})
  @Repeatable(RepeatableAnnoCol.class)
  public @interface RepeatableAnno {
  }

  // 容器注解的Target必须是可重复注解Target的子集
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.TYPE)
  public @interface RepeatableAnnoCol {
      RepeatableAnno[] value();
  }

  //通过原注解，使用了多次则无法获取
  RepeatableAnno annotation = XXX.class.getAnnotation(RepeatableAnno.class);
  //通过容器注解，只使用了一次则无法获取
  RepeatableAnnoCol annotationCol = XXX.class.getAnnotation(RepeatableAnnoCol.class);
  //推荐获取方式
  RepeatableAnno[] annotations = XXX.class.getAnnotationsByType(RepeatableAnno.class);
  ```

### **SpringBoot注解**

- AnnotationUtils的findAnnotation方法：SpringBoot内部使用，功能远比getAnnotation方法强大，能从类自身、父类、接口以及注解上获取注解信息（**lang包下的注解除外，如@FunctionalInterface**），无需@Inherited元注解。

- @Validated、@RequestMapping等：只要能通过AnnotationUtils.findAnnotation方法获取注解到即开启注解相应功能。

- @Component等DI注解：需要直接或间接（注解在注解上）的方式注解在类上才能被IOC容器管理，即无法继承，这是SpringBoot自身设计考量。

***

## Reference

引用对象类型的基类，用于创造一个对象的引用对象，与垃圾回收机制息息相关，**只有当对象非强可达时，对象才可能被回收**。

### 生命周期

创建完成后状态为active，当被引用对象变为不可达后，如果指定了ReferenceQueue会转变成pending，然后被加入pending list，等待ReferenceHandler处理，处理完成后状态变为enqueued，如未指定ReferenceQueue或被移出后，状态转变为最终态inactive，等待变为不可达后被GC回收。

### 属性

- T referent：被引用的对象，可以为null（**不建议**）。

- volatile ReferenceQueue<? super T> queue：如未指定或已被移出ReferenceQueue则为常量值NULL，如已入队则更新为常量值ENQUEUED。

- volatile Reference next：ReferenceQueue中下一个Reference。

- Reference<?> discovered：active状态时，是由GC维护的引用链；pending状态时，是等待入ReferenceQueue的pending list；inactive状态时，为null。

### ReferenceQueue

被引用对象已经被回收的Reference队列，由Reference中的next指针连接而成的。**如果创建Reference时指定了ReferenceQueue，则被引用对象被回收后Reference将会被添加到队列中**。

### ReferenceHandler

高优先级的**daemon**线程，用于处理pending list，执行清理工作和入队ReferenceQueue。

### SoftReference

软引用，由垃圾收集器根据内存需求决定回收，在抛出OutOfMemoryError之前，会保证清除所有软引用。

### WeakReference

弱引用，每次执行GC时，弱引用都可以被回收。

### PhantomReference

虚引用，用于跟踪GC的回收活动，每次执行GC时，虚引用都可以被回收。创建时必须指定ReferenceQueue，get方法永远返回null。

***