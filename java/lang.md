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
  **wait方法必须在同步方法或同步块内使用（否则会抛出IllegalMonitorStateException），调用后线程会释放同步锁**；参数为0L时，Java线程状态转变为WAIT，一直等待直到被唤醒或被打断；大于0L则Java线程状态转变为TIMED_WAITING，等待被唤醒、打断或达到超时时间后被自动唤醒。

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
  如果字符串常量池已经存在和当前字符串相等的字符串则返回常量池内的对象，否则将当前字符串对象加入字符串常量池后返回自身。**可以用于将String引用指向字符串常量池内的String对象，这样就可以回收不在常量池中的String对象。**

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

**均为不可变对象，每次变更操作都会返回新对象。**

### BigInteger

- ```java
  // 符号位，-1：负数，0：0，1：正数。
  final int signum;

  // 数值按32位拆分存储，即每个int都是无符号整型。
  final int[] mag;
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
  // 精度，限制有效数字的长度，0则表示不限制。
  final int precision;

  // 舍入模式，默认是HALF_UP。
  final RoundingMode roundingMode;
  ```

### BigDecimal

#### 实现及构造

- ```java
  // unscaledValue（有效数字）
  private final BigInteger intVal;

  // 正数表示小数点左移，负数表示右移。
  private final int scale;

  // unscaledValue在long能表示的范围内时使用
  private final transient long intCompact;

  // 精度，即有效数字长度，可通过MathContext指定。
  private transient int precision;
  ```
  **decimal = unscaledValue × 10 ^ -scale**

- ```java
  // 可以指定MathContext，默认为UNLIMITED，即不处理unscaledValue。
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
  // 设置scale为给定值，重新计算unscaledVal。
  public BigDecimal setScale(int newScale) {
      return setScale(newScale, ROUND_UNNECESSARY);
  }

  // 如需要舍入且舍入模式为UNNECESSARY则会抛出ArithmeticException
  public BigDecimal setScale(int newScale, RoundingMode roundingMode) { ... }
  ```

- ```java
  // 移除有效数字的所有尾部0
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

## abstract class **ClassLoader**

- ```java
  protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
      synchronized (getClassLoadingLock(name)) { // 获取同步锁
          Class<?> c = findLoadedClass(name); // 首先检查是否已经加载
          if (c == null) {
              ...
              try {
                  // 父级存在则委托给父级加载，不存在则委托给Bootstrap加载。
                  if (parent != null) {
                      c = parent.loadClass(name, false);
                  } else {
                      c = findBootstrapClassOrNull(name);
                  }
              } catch (ClassNotFoundException e) {
                  // 直接丢弃异常
              }
              // Bootstrap和父类无法加载才尝试由自身加载
              if (c == null) {
                  ...
                  c = findClass(name);
                  ...
              }
          }
          if (resolve) {
            resolveClass(c); // 解析Class对象，默认判断其为null时抛出NullPointerException
          }
          return c;
      }
  }
  ```
  类加载方法，双亲委派模型的保证，非final，可以在自定义加载器中重写以破坏双亲委派模型。

- ```java
  // 类加载器类为parallelCapable时才会创建
  private final ConcurrentHashMap<String, Object> parallelLockMap;

  // 获取同步锁锁定的对象，如果类非parallelCapable则锁定加载器自身。
  protected Object getClassLoadingLock(String className) {
      Object lock = this;
      if (parallelLockMap != null) {
          Object newLock = new Object();
          lock = parallelLockMap.putIfAbsent(className, newLock);
          if (lock == null) {
              lock = newLock;
          }
      }
      return lock;
  }
  
  // 将当前类加载器类注册为parallelCapable，要求：1.父类必须是parallelCapable；2.调用时未创建实例。
  protected static boolean registerAsParallelCapable() { ... }

  // 由JVM调用，清理parallelCapable相关的集合。
  private void resetArchivedStates() { ... }
  ```
  并发加载支持。

- ```java
  private final native Class<?> findLoadedClass0(String name);
  ```
  本地方法，用于获取当前类加载器对象加载过的Class对象，JVM中使用**SystemDictonary**存储，本质上是一个哈希表，key是【类加载器对象+类全限定名】，value是【指向klass的地址】。

- ```java
  protected Class<?> findClass(String name) throws ClassNotFoundException {
      throw new ClassNotFoundException(name);
  }
  ```
  自定义类加载器时需要重写findClass方法，重写逻辑为：先获取到类的二进制流，再调用defineClass方法创建Class对象。

- ```java
  protected final Class<?> defineClass(String name, byte[] b, int off, int len, ProtectionDomain protectionDomain) throws ClassFormatError { ... }
  ```
  在自定义的findClass方法中使用，将字节码文件转换为Class对象，**每个ClassLoader实例只能转换同一个类一次（因为是将类元信息保存到SystemDictonary）**，除非该类对象已被卸载，但是类卸载是不可控的（Full GC时触发）。

- ```java
  final Class<?> loadClass(Module module, String name) {
      synchronized (getClassLoadingLock(name)) {
          Class<?> c = findLoadedClass(name);
          if (c == null) {
              c = findClass(module.getName(), name);
          }
          if (c != null && c.getModule() == module) {
              return c;
          } else {
              return null;
          }
      }
  }

  // 属于模块的加载器需要重写该方法
  protected Class<?> findClass(String moduleName, String name) {
      if (moduleName == null) {
          try {
              return findClass(name);
          } catch (ClassNotFoundException ignore) { }
      }
      return null;
  }
  ```
  在 Class.forName(Module module, String name) 中调用，使用模块的加载器去加载类，**没有遵守双亲委派模型**。

***

## 运行时类

### System

**系统工具类**

- ```java
  // 获取时间
  public static native long currentTimeMillis();

  // 用于计时
  public static native long nanoTime();
  ```
  **nanoTime方法返回从某一固定但任意的时间算起的纳秒数**，或许从以后算起，所以返回值可能为负。

- ```java
  // 返回对象的标识哈希码，即hashcode方法的默认值。
  public static native int identityHashCode(Object x);
  ```
  返回结果是与内存地址相关的，但是因为返回值为int，故不同对象的返回值可能是相同的。

- ```java
  public static void exit(int status) {
      Runtime.getRuntime().exit(status);
  }
  ```
  部分方法均是直接引用Runtime。

### Runtime

**JVM的运行时环境类**，为饿汉式单例模式。

- ```java
  private static final Runtime currentRuntime = new Runtime();

  public static Runtime getRuntime() {
      return currentRuntime;
  }

  private Runtime() {}
  ```
  
- ```java
  // 退出JVM，status非0时表示异常退出，不会执行钩子线程。
  public void exit(int status) {
      ...
      Shutdown.exit(status);
  }

  // 添加钩子线程，用于优雅退出。
  public void addShutdownHook(Thread hook) {
      ...
      // 为同步方法
      ApplicationShutdownHooks.add(hook);
  }
  ```
  **状态码非0或使用【kill -9 \<pid\>】时均不会执行钩子线程。**

- ```java
  // 返回JVM可以使用的处理器（逻辑处理器）数量
  public native int availableProcessors();
  ```

- ```java
  // 执行垃圾回收，并不保证立即执行。
  public native void gc();
  ```

***