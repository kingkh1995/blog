## [首页](/blog/)
> version: **jdk11**

***

#### Object

  - registerNatives()

    > native方法，在静态代码块中被调用。

  - getClass()

    > native方法，返回一个运行时Class对象，返回的类对象是被该类static synchronized方法锁住的对象。

  - hashCode()

    > native方法，返回对象的哈希码值，**哈希码值和对象地址有一定关联，但并不一定如此**，具体取决于运行时库和JVM的具体实现。

    > 原生类型均重写了hashcode方法：Integer类型直接返回32位值；Long和Double类型返回前32位和后32异或的结果；Boolean类型直接返回两个特定值；String类型使用31作为因子对字符数组每一位使用除留余数法，并且有软缓存。

  - equals()

    > 默认是对象引用对比的结果，通常需要在重写此方法时覆盖hashCode方法，是**为了维护hashCode方法的常规协定（散列集合是基于这些协定设计的）**，equals方法对比相等的对象必须具有相等的哈希码值，哈希码值不相等的对象使用equals方法对比必然不相等（可推算出）。

  - clone()

    > **protected** native方法，实现为浅拷贝，一个类必须实现Cloneable接口才能使用clone方法，否则会抛出CloneNotSupportedException，可能需要通过重写修改其访问限定修饰符为public，**注意Object类未实现Cloneable**。

    > 数组重写了clone方法，会返回一个新数组，但元素复制是浅拷贝，**不推荐使用clone方法**，而是使用Arrays工具类复制数组。

  - notify()、notifyAll()

    > **final** native方法，notify唤醒正在此对象监视器上等待的任意的一个线程，notifyAll唤醒正在此对象监视器上等待的所有线程。被唤醒的线程仍然无法继续，直到当前线程放弃对对象的锁定。

  - wait()、wait(long timeoutMillis)、wait(long timeoutMillis, int nanos)

    > **final** native方法，调用的均是wait(long timeoutMillis)方法，使当前线程释放该对象的监视器锁，等待被唤醒或者时间达到超时时间，之后与其他等待中的线程公平竞争该对象的锁。

    > **wait() == wait(0L)，并不是释放锁之后立刻进入对象锁等待池，而是一直等待直到被notify方法唤醒**。

***
    
#### String

- 内部实现：
  - private final byte[] value
  - private final byte coder
  - static final boolean COMPACT_STRINGS

    > **jdk9开始使用byte数组存储字符**，存储时字符编码方式有两种：Latin1和UTF-16BE，分别对应的coder值为0和1。
    
    > COMPACT_STRINGS代表jvm是否允许压缩字符，如果允许压缩字符并且字符串中字符全在Latin1能表示的范围内，那么会使用Latin1编码，这样一个字符只会占用一个字节，否则会占用两个字节。

    - *字符集相关：*

      > Latin1 & ASCII：Latin1是iso-8859-1的别名，Latin1和ASCII都是单字节字符，ASCII定义了128个字符，只使用了后7位且最高位默认为0，而Latin1使用了全部8位，定义了256个字符，能完全向下兼容ASCII。

      > Unicode字符集：是一种通用字符集，它的编码空间可以划分为17个平面（plane），每个平面包含65,536个码位（code point），第一个平面称为基本多语言平面（BMP），其他平面称为辅助平面。字符集只是指定了字符的编号，但是却有多种编码方式去表示这个编号。

      > utf-16：Unicode字符集规定的标准编码实现，**也是jvm运行时字符的默认编码方式（jdk9之前）**，固定使用两个字节去表示BMP（包括中文）内的字符，BMP之外的字符则需要四个字节去表示。

      > utf-8：可变长字符编码，使用1到4个字节表示一个Unicode字符，**java字节码文件使用utf-8编码**。对于ASCII字符，utf-8编码表示与其相同，即都只需要一个字节，但是表示一个中文字符却需要三个字节。

  - private int hash

    > 重写了hashcode方法，第一次被调用计算出散列值后会被缓存到hash字段中。

- 构造方法：

  - new String(String original)

    > 该构造函数会使用传入的String对象构造一个新的String对象，相当于浅拷贝，即两个对象公用同一个value数组，**一般情况下不推荐使用**。

  - String(StringBuilder builder) & String(StringBuffer buffer)

    > 应该使用对应的toString方法。

  - String(byte bytes[], String charsetName)
            throws UnsupportedEncodingException
  - String(byte bytes[], Charset charset)
  
    > 使用特定的编码方式创建一个字符串，推荐使用Charset参数的构造方法，传入StandardCharsets类的一个静态常量。


- 常用方法：

  - getBytes()

    > 获取该字符串使用指定编码方式的表示，不带参数则默认为字节码编码方式（即utf-8），指定其他编码方式推荐选择Charset参数的方法。
    
    > 如果指定编码方式为utf-16会比预计多出两个字节，原因是其以两个字节为编码单元，要考虑字节序，所以返回的字节数组需要使用额外的两个字节表示来标志字节序（FEFF或FFFE），指定UTF-16BE或者UTF-16LE就会是原始的编码表示。

  - length()

    > 实现为value.length >> coder()，这也是为什么coder值为0和1的原因，**但需要知道对于BMP外的字符是占用四个字节的，所以该返回值可以说并不是完全正确**。

  - char charAt(int index) & int codePointAt(int index)

    > charAt获取BMP内的unicode字符，所以返回值是两个字节的char；而codePointAt能获取BMP外的unicode字符，最多会占四个字节，所以返回值是int。

  - strip()（jdk11添加） & trim()

    > 都是移除字符串两侧的空白字符，trim()方法移除空格、tab键、换行符，而strip()方法移除所有Unicode空白字符，**不推荐继续使用trim()方法**。
    
    > strip()和isBlank()内部实现都是调用Character.isWhitespace()，该方法能识别所有Unicode空白字符。

  - split(String regex)

    > 参数为正则表达式，所以对于正则表达式的特殊字符需要使用\\\\转义为普通字符，否则将使用正则表达式模式进行匹配。

***
 
#### StringBuffer & StringBuilder

  - 相同点：

    > 均是可变字符串，均继承自AbstractStringBuilder，默认容量均为16。

  - 不同点：

    > StringBuffer的所有方法都使用了synchronized修饰，所以它是线程安全的。
    
    > StringBuffer包含一个私有的transient的String类型属性（toStringCache），是针对toString方法的软缓存，第一次调用toString方法时缓存，字符串变更时则删除缓存。

  - **AbstractStringBuilder**

    > 设计与String基本相同，因为是可变字符串，拥有一个count字段来标识可变字符串的实际长度。
    
    > 扩容时变为原容量的两倍加2（why?），如果容量仍然不足则会直接加上新增的容量。

    > **注意**：并没有AbstractStringBuilder(char c)的构造方法！你可以这么书写而且编译会通过，因为实际上是将char转化为了int类型，所以会执行 StringBuilder(int capacity) 构造方法。

***

#### Byte、Integer、Long

- 都通过一个私有的缓存内部类，在静态代码块中为-128到127的值区间加载了缓存，但是只有Integer可以通过JVM参数调整缓存区间的上限（**下限固定为-128**）。

- valueOf方法返回基本数据类型对应的包装类（**推荐使用该方法获取包装类**），parseXXX(String s)方法返回基本数据类型。

***

#### Enum（所有枚举的父类）

  - clone()：重写了clone方法，会直接抛出CloneNotSupportedException，因为是绝对的单例模式所以不允许被克隆。

  - valueOf()：**如果枚举常量不存在会抛出IllegalArgumentException，而不会返回null。**

***

### annotation

#### ElementType枚举
> 配合Target注解指定注解能出现在Java程序中的语法位置。

#### Target元注解
> 如果注解声明中不存在Target元注解，则该注解可以java程序的任意位置使用。

#### RetentionPolicy枚举
> 配合Retention元注解，指定注解的保留时间。

- CALSS：编译器将把注解记录在类文件中，但在运行时JVM不需要保留注解。
- RUNTIME:：编译器将把注解记录在类文件中，同时在运行时JVM将保留注解，所以可以通过反射机制读取到注解信息。
- SOURCE：编译器要丢弃的注解。

#### Retention元注解
> 如果注解中不存在Retention注解，则保留策略默认为RetentionPolicy.CLASS。

#### Inherited元注解
> 指示注解会被自动继承，如果某个类声明中没有此类型的注解，则将在该类的超类中自动查询该类型注解，会一直向上追溯，直到找到该类型的注解或到达该类层次的顶层为止。

#### Repeatable元注解
> 如果声明了Reteatable元注解，代表该注解可以重复使用，需要一个容器注解配合实现。

***

#### Thread

#### ThreadLocal

#### InheritableThreadLocal

#### ThreadGroup
