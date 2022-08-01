# [首页](/blog/)

> version: **jdk17**

***

## Object

- ```java
  public native int hashCode();
  ```
  返回对象的哈希值，与对象地址有一定关联，但并不一定如此，具体取决于运行时库和JVM的具体实现。

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

  // 最终都是调用该本地方法
  public final native void wait(long timeoutMillis) throws InterruptedException;
  ```
  **wait方法必须在同步方法或同步块内使用，调用后线程会释放同步锁**；参数为0L时，Java线程状态转变为WAIT，一直等待直到被唤醒或被打断；大于0L则Java线程状态转变为TIMED_WAITING，等待被唤醒、打断或达到超时时间后被自动唤醒。

- ```java
  // 唤醒任意一个在此对象监视器上等待的线程（非公平）
  public final native void notify();

  // 唤醒所有在此对象监视器上等待的线程
  public final native void notifyAll();
  ```
  **必须在同步方法或同步块内使用**，线程被唤醒后Java线程状态转变为RUNNABLE，尝试竞争锁，如果失败则进入锁等待池，Java线程状态转变为BLOCKED。

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

- Unicode字符集：是一种通用字符集，它的编码空间可以划分为17个平面（plane），每个平面包含65536个码位（code point），第一个平面称为基本多语言平面BMP（含中文），其他平面称为辅助平面。**Unicode字符集只是指定了字符的编号，但是却有多种编码方式去表示这个编号**。

- utf-16：Unicode字符集规定的标准编码实现，固定使用两个字节去表示BMP内的字符，BMP之外的字符则需要四个字节去表示。

- utf-8：可变长字符编码，使用1到6个字节表示一个Unicode字符。对于ASCII字符，utf-8编码能完全兼容，即只需要一个字节，但是表示一个中文字符却需要三个字节。
  > **为Java字节码文件使用的编码**，因为代码中使用的字符几乎都是ASCII字符，故使用utf-8相比于utf-16能大大节约空间，因为utf-16表示一个ASCII字符需要两个字节。

### 构造方法

**只有通过""创建的字符串才会被加入字符串常量池中，而new出来的String对象会存放在堆中。**

- ```java
  public String(String original) {
      this.value = original.value;
      this.coder = original.coder;
      this.hash = original.hash;
  }
  ```
  唯一会复用value数组的构造方法，如果字符串常量池中不存在original，则需要创建两个String对象和一个byte数组。

- ```java
  public String(byte[] bytes, int offset, int length, Charset charset) { ... }
  ```
  将byte数组按给定的编码方式解码为字符串，StandardCharsets类中定义了常用的Charset类型常量。

### 实例方法

- ```java
  public native String intern();
  ```
  如果字符串常量池已经存在和当前字符串相等的字符串则返回常量池内的对象，否则将当前字符串对象加入字符串常量池后返回自身。

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
  // StringLatin1
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

- ```java
  // 获取BMP内的Unicode字符，故返回值类型为char
  public char charAt(int index) { ... }

  // 获取CodePoint，故返回值类型为int
  public int codePointAt(int index) { ... }

  // 将字符串转换为字符流，会把char扩展为int，因为没有CharStream
  public IntStream chars() { ... }

  // 将字符串转换为CodePoint流
  public IntStream codePoints() { ... }
  ```

- ```java
  // 将字符串按指定的编码方式编码为byte数组
  public byte[] getBytes(Charset charset) { ... }
  ```
  注意如果是UTF_16编码需要使用额外的两个字节来标识字节序（FEFF或FFFE）。

- ```java
  public boolean contentEquals(CharSequence cs) { ... }
  ```
  当前字符串与给定CharSequence进行比较，**如果cs是StringBuffer类型会加上同步**。

- ```java
  // 参数为CodePoint值
  public int indexOf(int ch) { ... }
  ```
  indexOf和lastIndexOf均是直接遍历查找。

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
  每次都会使用regex参数创建一个Pattern对象，**同理其他有regex参数的方法也是一样**。

- ```java
  public String[] split(String regex) {
      return split(regex, 0);
  }

  public String[] split(String regex, int limit) { ... }
  ```
  - regex参数为正则表达式，特殊字符如需按普通字符匹配则需要使用\\\\转义；
  - limit参数如果大于0，则拆分后数组长度不会超过limit；
  - 因为每次都会编译regex，**推荐使用Guava的Splitter工具类替代**。

- ```java
  // 去除首位所有Unicode空白字符
  public String strip() { ... }

  // 判断是否为空（只包含Unicode空白字符）
  public boolean isBlank() { ... }
  ```
  - JDK11新增，StringUTF16使用Character.isWhitespace()判断Unicode空白字符；
  - **不应该再使用trim()，它只移除空格、tab键、换行符。**

- ```java
  public String formatted(Object... args) {
      return new Formatter().format(this, args).toString();
  }
  ```
  JDK15新增，自身作为模式字符串去格式化字符串。不推荐多次调用该方法以及其他静态format方法，不仅每次都会创建一个Formatter对象，且Formatter对象在执行format方法时才会去解析模式字符串。

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

- 至少要保证容量不能低于所需的最小容量，**如果足够则增长为两倍当前容量加2**，但value数组的长度也不能超过虚拟机的限制（比Integer.MAX_VALUE略小），若都无法满足则抛出OutOfMemoryError。

### StringBuffer

线程安全，默认容量16，所有方法全是直接调用父类方法，但都加上了同步，同时toString方法有缓存（使用toStringCache属性）。

### **StringBuilder**

非线程安全，默认容量16，所有方法全是直接调用父类方法。

### **要点**

1. 除了delete之外的其他变更操作都会尝试压缩字符串（inflate方法）；
1. append和insert操作如果对象参数为null，则会视作"null"；
1. 不存在char参数的构造方法，你可以这么写但实际上执行的构造方法是AbstractStringBuilder(int capacity)；
2. 使用 + 拼接字符串会被优化为使用StringBuilder，即不应该在循环中使用 + 拼接字符串，因为每次都会new一个StringBuilder对象。

***

## 包装类

- ```java
  public static final Class<Integer> TYPE = (Class<Integer>) Class.getPrimitiveClass("int");
  ```
  包装类的TYPE属性为其对应的基本数据类型的Class对象，它不是由类加载机制创建，而是运行时由JVM创建。

- ```java
  // Boolean
  public static int hashCode(boolean value) {
      return value ? 1231 : 1237;
  }
  // Byte、Short、Character
  public static int hashCode(byte/short/char value) {
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
  包装类均重写了hashcode和equals方法。

- ```java
  public static Integer valueOf(int i) {
      // IntegerCache为私有缓存内部类
      if (i >= IntegerCache.low && i <= IntegerCache.high)
          return IntegerCache.cache[i + (-IntegerCache.low)];
      return new Integer(i);
  }
  ```
  - **包装类的构造方法均已被标识为@Deprecated**，Boolean应该直接使用常量，其他则应该使用静态valueOf方法创建，可以利用到缓存；
  - 整型的缓存区间均为\[-128, 127\]，**但只有Integer可以通过JVM参数调整缓存区间的上限**；
  - Character的缓存区间为\[0, 127\]，即所有ASCII字符。

- ```java
  public static int parseInt(String s) throws NumberFormatException {
      return parseInt(s,10);
  }
  
  public static int parseInt(String s, int radix) throws NumberFormatException { ... }
  ```
  包装类的静态parse方法用于将字符串转换为对应的基本数据类型，整型还可以按指定的进制解析（radix：基数）。

- ```java
  // Long，返回当前值作为无符号整型的字符串表示
  public static String toUnsignedString(long i) {
      return toUnsignedString(i, 10); // 默认十进制
  }

  public static String toUnsignedString(long i, int radix) { ... }
  ```
  Unsigned相关方法为无符号整型操作，**无符号整型即整数的二进制表示中最高位不再视为符号位**。

***

##  大数

**为不可变对象，每次变更操作都会返回新对象。**

### BigInteger

- ```java
  // 符号位，-1：负数，0：0，1：正数。
  final int signum;

  // 数值按32位拆分存储，即每个int都是无符号整型。
  final int[] mag;

  // 使用除留余数法计算哈希值
  public int hashCode() {
      int hashCode = 0;
      for (int i = 0; i < mag.length; i++)
          // 使用LONG_MASK转换为long后再参与计算
          hashCode = (int)(31 * hashCode + (mag[i] & LONG_MASK));
      return hashCode * signum;
  }
  ```

- ```java
  // 使用除留余数法计算哈希值
  public int hashCode() {
      int hashCode = 0;
      for (int i = 0; i < mag.length; i++)
          // 因为每个int都是无符号的，故使用LONG_MASK转换为long后再参与计算
          hashCode = (int)(31 * hashCode + (mag[i] & LONG_MASK));
      return hashCode * signum;
  }
  ```

- ```java
  // 缓存区间为[-16, 16]
  public static BigInteger valueOf(long val) { ... }

  // 可以指定基数，默认为10。
  public BigInteger(String val, int radix) { ... }
  ```
  应该尽可能的使用静态valueOf方法创建BigInteger。

### MathContext

- ```java
  // 精度，表示有效数字的长度，0则表示不限制。
  final int precision;

  // 舍入模式，默认是HALF_UP。
  final RoundingMode roundingMode;
  ```

### BigDecimal

#### 实现及构造

- ```java
  // 有效数字在long能表示的范围内时使用
  private final transient long intCompact;

  // 有效数字
  private final BigInteger intVal;

  // 有效数字中小数点的位置，正数表示小数点应该左移，负数表示应该右移。
  private final int scale;

  // 精度，即有效数字长度，可通过MathContext指定。
  private transient int precision;
  ```

- ```java
  // 使用字符串创建时，MathContext默认为UNLIMITED。
  public BigDecimal(String val, MathContext mc) { ... }

  // 不应该使用
  public BigDecimal(double val) {
      this(val, MathContext.UNLIMITED);
  }

  // 浮点数应该转为字符串后使用字符串参数的构造方法
  public static BigDecimal valueOf(double val) {
      return new BigDecimal(Double.toString(val));
  }

  // 整型使用，能利用到缓存。
  public static BigDecimal valueOf(long val) { ... }

  // 尽量使用
  public static BigDecimal valueOf(long unscaledVal, int scale) { ... }
  ```

#### 方法

- ```java
  public BigDecimal setScale(int newScale) {
      return setScale(newScale, ROUND_UNNECESSARY);
  }

  public BigDecimal setScale(int newScale, RoundingMode roundingMode) { ... }
  ```
  设置有效数字的小数点位，如果需要舍去且舍入模式为UNNECESSARY则会抛出ArithmeticException，**不修改原对象而是返回新对象**。

- ```java
  // 移除有效数字的所有尾部0，返回新对象。
  public BigDecimal stripTrailingZeros() { ... }
  ```

- ```java
  public BigInteger toBigInteger() {
      // 直接舍去小数位
      return this.setScale(0, ROUND_DOWN).inflated();
  }

  public BigInteger toBigIntegerExact() {
      // 如果需要舍入则抛出ArithmeticException
      return this.setScale(0, ROUND_UNNECESSARY).inflated();
  }
  ```

- ```java
  // 有缓存，必要时输出为科学计数法表示。
  public String toString() {
      String sc = stringCache;
      if (sc == null) {
          stringCache = sc = layoutChars(true);
      }
      return sc;
  }

  // 无缓存，必要时输出为工程计数法表示（10的幂是3的倍数）。
  public String toEngineeringString() {
      return layoutChars(false);
  }

  // 无缓存，输出原始文本格式。
  public String toPlainString() { ... }
  ```

***

## 枚举

枚举类是由JVM保证的绝对单例模式

```java
// 使用枚举实现饿汉式单例模式
public enum Singleton {
    INSTANCE;
    
    ...
}
```

### abstract class **Enum**\<E extends **Enum**\<E\>\> implements Constable, Comparable<E>, Serializable

所有枚举类的基类，由编译器添加继承关系，**故枚举类不可继承类，但是可以实现接口**。

- ```java
  // 绝对的单例模式，不允许克隆。
  protected final Object clone() throws CloneNotSupportedException {
      throw new CloneNotSupportedException();
  }
  ```

- ```java
  public static <T extends Enum<T>> T valueOf(Class<T> enumClass, String name) {
      T result = enumClass.enumConstantDirectory().get(name);
      if (result != null)
          return result;
      if (name == null)
          throw new NullPointerException("Name is null");
      throw new IllegalArgumentException(
            "No enum constant " + enumClass.getCanonicalName() + "." + name);
  }
  ```
  使用valueOf方法时需要注意，当枚举值不存在时会抛出IllegalArgumentException而不是返回null。

- ```java
  @java.io.Serial
  private void readObject(ObjectInputStream in) throws IOException,
        ClassNotFoundException {
      throw new InvalidObjectException("can't deserialize enum");
  }

  @java.io.Serial
  private void readObjectNoData() throws ObjectStreamException {
      throw new InvalidObjectException("can't deserialize enum");
  }
  ```
  为了保证绝对的单例禁用了默认反序列化机制，**反序列化时使用valueOf方法**。

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

软引用，由垃圾收集器根据内存需求决定回收，在抛出OutOfMemoryError之前，会保证清除所有软引用。不适合做高频缓存，因为一旦触发大规模回收，将会导致压力极具增大。

### WeakReference

弱引用，每次执行GC时，弱引用都可以被回收。

### PhantomReference

虚引用，用于跟踪GC的回收活动，每次执行GC时，虚引用都可以被回收。创建时必须指定ReferenceQueue，get方法永远返回null。

***