# [首页](/blog/)

> version: **jdk17**

***

## Object

- ```java
  public native int hashCode();
  ```
  返回对象的哈希值，与对象地址有一定关联，但并不一定如此，具体取决于运行时库和JVM的具体实现。
  ```java
  // Boolean
  public static int hashCode(boolean value) {
    return value ? 1231 : 1237;
  }
  // Byte、Short、Character
  public static int hashCode(byte value) {
    return (int)value;
  }
  // Integer
  public static int hashCode(int value) {
    return value;
  }
  // Long
  public static int hashCode(long value) {
    return (int)(value ^ (value >>> 32));
  }
  // Float
  public static int hashCode(float value) {
    return floatToIntBits(value); // 返回二进制表示 
  }
  // Double
  public static int hashCode(double value) {
    long bits = doubleToLongBits(value); // 返回二进制表示
    return (int)(bits ^ (bits >>> 32));
  }
  ```

- ```java
  public boolean equals(Object obj) {
    return (this == obj);
  }
  ```
  重写equals方法需要遵守hashCode方法的常规协定（**散列集合是基于此协定设计的**）：equals方法对比相等的两个对象必然具有相等的哈希值，哈希值不等的两个对象使用equals方法对比必然不等。

- ```java
  protected native Object clone() throws CloneNotSupportedException;
  ```
  实现为浅拷贝，一个类必须实现Cloneable接口才能调用clone方法，否则会抛出CloneNotSupportedException，**注意Object类未实现Cloneable**。
    > **数组可以调用clone方法（显然访问限定修饰符被重写为public），但拷贝数组更推荐使用Arrays的copyOf相关方法或System的arraycopy方法。**

- ```java
  public final void wait() throws InterruptedException {
    wait(0L);
  }

  public final void wait(long timeoutMillis, int nanos) throws InterruptedException {
    ...
    // 并不会真的精确到nanos
    if (nanos > 0 && timeoutMillis < Long.MAX_VALUE) {
      timeoutMillis++;
    }
    wait(timeoutMillis); 
  }

  // 最终都是调用该方法
  public final native void wait(long timeoutMillis) throws InterruptedException;
  ```
  **wait方法必须在同步方法或同步块内使用，调用后线程会释放同步锁**；参数为0L时，Java线程状态转变为WAIT，一直等待直到被唤醒或被打断；大于0L则Java线程状态转变为TIMED_WAITING，等待被唤醒、打断或达到超时时间后被自动唤醒。

- ```java
  // 唤醒任意一个在此对象监视器上等待的线程（非公平）
  public final native void notify();

  // 唤醒所有在此对象监视器上等待的线程
  public final native void notifyAll();
  ```
  **必须在同步方法或同步块内使用**，线程被唤醒后Java状态转变为RUNNABLE，尝试竞争锁，如果失败则进入锁等待池，Java线程状态转变为BLOCKED。

- ```java
  @Deprecated(since="9")
  protected void finalize() throws Throwable { }
  ```

***
    
## String

### 字符存储

```java
private final byte[] value;
private final byte coder; // 字符编码方式，0: Latin1，1: UTF-16BE。
static final boolean COMPACT_STRINGS; // 是否压缩字符，默认true。
```
JDK9开始支持字符压缩，即如果字符全部在Latin1能表示的范围内会使用Latin1编码，否则使用UTF-16BE编码（之前的编码方式），并使用byte\[]存储字符（之前使用char\[]），Latin1编码的操作类为StringLatin1，UTF-16BE编码的操作类为StringUTF16（**使用getChar方法将两个连续的byte值读取为一个char值**）。

### 字符编码

- ASCII & Latin1：**Latin1是iso-8859-1的别名，Latin1和ASCII都是单字节字符**；ASCII定义了128个字符，只使用了后7位最高位默认为0；Latin1定义了256个字符，使用了全部8位，能完全向下兼容ASCII。

- Unicode字符集：是一种通用字符集，它的编码空间可以划分为17个平面（plane），每个平面包含65536个码位（code point），第一个平面称为基本多语言平面（BMP）（含中文），其他平面称为辅助平面。**Unicode字符集只是指定了字符的编号，但是却有多种编码方式去表示这个编号**。

- utf-16：Unicode字符集规定的标准编码实现，固定使用两个字节去表示BMP内的字符，BMP之外的字符则需要四个字节去表示。

- utf-8：可变长字符编码，使用1到4个字节表示一个Unicode字符。对于ASCII字符，utf-8编码能完全兼容，即只需要一个字节，但是表示一个中文字符却需要三个字节。
  > **为Java字节码文件使用的编码**，因为代码中使用的字符几乎都是ASCII字符，故使用utf-8相比于utf-16能大大节约空间，因为utf-16表示一个ASCII字符需要两个字节。

### 构造方法

- ```java
  public String(String original) {
    this.value = original.value;
    this.coder = original.coder;
    this.hash = original.hash;
  }
  ```
  唯一会复用value数组的构造方法，如果字符串常量池中不存在original，则需要创建两个String对象和一个byte数组。
    > **只有通过""创建的字符串才会被加入字符串常量池中，而new出来的String对象会存放在堆中。**

- public String(byte bytes[], Charset charset)：将byte数组按给定的编码方式解码为字符串，StandardCharsets类中定义了常用的Charset类型常量。

### 实例方法

- ```java
  public native String intern();
  ```
  如果字符串常量池已经存在和该字符串相等的字符串则返回常量池内的对象，否则将当前字符串对象加入字符串常量池然后返回自身。

- ```java
  private int hash; // 哈希值缓存，默认0。
  private boolean hashIsZero; // 标识哈希值是否为0，默认false。
  // 重写hashCode方法
  public int hashCode() {
    int h = hash;
    if (h == 0 && !hashIsZero) {
      // 使用除留余数法计算哈希值
      h = isLatin1() ? StringLatin1.hashCode(value) : StringUTF16.hashCode(value);
      if (h == 0) {
        hashIsZero = true;
      } else {
        hash = h; // 首次计算后才设置到hash属性
      }
    }
    return h;
  }
  //  StringLatin1
  public static int hashCode(byte[] value) {
    int h = 0;
    for (byte v : value) {
      h = 31 * h + (v & 0xff);
    }
    return h;
  }
  // StringUTF16
  public static int hashCode(byte[] value) {
    int h = 0;
    int length = value.length >> 1;
    for (int i = 0; i < length; i++) {
      h = 31 * h + getChar(value, i);
    }
    return h;
  }
  ```

- ```java
  public int length() {
    return value.length >> coder();
  }
  ```
  需要知道BMP外的字符会占用四个字节，所以返回值并不一定等于字符串的长度。

- public char charAt(int index)：获取BMP内的Unicode字符，所以返回两个字节的char。

- public int codePointAt(int index)：获取CodePoint，所以返回四个字节的int。

- public IntStream chars()：转换为字符流，会把char扩展为int，因为没有CharStream。

- public IntStream codePoints()：转换为CodePoint流。

- public byte[] getBytes(Charset charset)：将字符串按指定的编码方式编码为byte数组。
  > UTF_16编码需要使用额外的两个字节来标识字节序（FEFF或FFFE）。

- public boolean contentEquals(CharSequence cs)：与给定CharSequence进行比较，**如果是StringBuffer类型会加上同步**。

- public int indexOf(int ch, int fromIndex)：ch是单个CodePoint值。

- public int indexOf(String str, int fromIndex)：没有使用任何算法优化，直接遍历查找。

- public String strip()：JDK11新增，移除首尾的Unicode空白字符，使用Character.isWhitespace()识别。
  > **trim()只移除空格、tab键、换行符。**

- public boolean isBlank()：JDK11新增，判断是否为空字符串，也是使用Character.isWhitespace()识别Unicode空白字符。

- ```java
  public boolean matches(String regex) {
    return Pattern.matches(regex, this);
  }
  // Pattern
  public static boolean matches(String regex, CharSequence input) {
    Pattern p = Pattern.compile(regex);
    Matcher m = p.matcher(input);
    return m.matches();
  }
  ```
  不建议使用，因为每次都会使用regex创建一个Pattern对象，同理其他有regex参数的方法也是一样。

- public String[] split(String regex, int limit)：
  - 因为参数为regex，即正则表达式的特殊字符如需按普通字符匹配则需要使用\\\\转义；
  - limit参数如果大于0，则拆分后数组长度不会超过limit；
  - 因为每次都会编译regex，**故更推荐使用Guava的Splitter工具类**。

- ```java
  public String formatted(Object... args) {
    return new Formatter().format(this, args).toString();
  }
  ```
  JDK15新增，使用自身作为模式字符串生成格式化字符串。不推荐多次调用该方法以及静态format方法，不仅每次都会创建一个Formatter对象，且在执行format方法时才会去解析模式字符串。

***

## 可变字符串

### abstract class **AbstractStringBuilder** implements Appendable, CharSequence

StringBuilder和StringBuffer的基类，内部实现与String基本相同。

- ```java
  int count; // 记录实际字符（非codePoint）数量
  public int length() {
    return count;
  }
  ```

- 执行变更操作时至少要保证容量不能低于所需的最小容量，如果足够则增长为两倍当前容量加2，但value数组的长度也不能超过虚拟机的限制（比Integer.MAX_VALUE略小），若都无法满足则抛出OutOfMemoryError。

### **StringBuilder**

非线程安全，默认容量16，所有方法全是直接调用父类方法。

### StringBuffer

线程安全，默认容量16，所有方法全是直接调用父类方法，但都加上了同步，同时toString方法有缓存（使用toStringCache属性）。

### **要点**

1. 除了delete其他变更操作都会尝试使用inflate()方法压缩字符串；
1. append和insert操作如果对象参数为null，则会视作"null"；
1. 不存在char参数的构造方法，你可以这么写但实际上执行的构造方法是AbstractStringBuilder(int capacity)；
1. 使用 + 拼接字符串会被优化为使用StringBuilder。
    ```java
    // 不应该在循环中使用 + 拼接字符串，因为每次都会new一个StringBuilder对象。
    for(String s = ""; ; ){
      s = s + "123";
    }
    ```

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