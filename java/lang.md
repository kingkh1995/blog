## [首页](https://kingkh1995.github.io/blog/)
> version: **jdk11**

#### Object
  - registerNatives()
    > native方法，在静态代码块中被调用。
  - getClass()
    > native方法，返回一个运行时Class对象，返回的类对象是被该类static synchronized方法锁住的对象。
  - hashCode()
    > native方法，返回对象的哈希码值，**哈希码值和对象地址有一定关联，但并不一定如此**，具体取决于运行时库和JVM的具体实现。
  - equals()
    > 默认是对象引用对比的结果，通常需要在重写此方法时覆盖hashCode方法，是**为了维护hashCode方法的常规协定（散列集合是基于这些协定设计的）**，equals方法对比相等的对象必须具有相等的哈希码值，哈希码值不相等的对象使用equals方法对比必然不相等（可推算出）。
  - clone()
    > **protected** native方法，默认实现是浅拷贝，如果类未实现Cloneable接口调用clone方法会抛出CloneNotSupportedException，**注意Object类未实现Cloneable**。
    >> 所有数组都被视为已经实现了Cloneable，并且访问修饰符被修改为了public，clone方法会返回一个利用原数组内元素构造的新数组，使用equals方法会返回false，因为实际调用的是Object的equals方法。
  - notify()、notifyAll()
    > **final** native方法，notify唤醒正在此对象监视器上等待的任意的一个线程，notifyAll唤醒正在此对象监视器上等待的所有线程。被唤醒的线程仍然无法继续，直到当前线程放弃对对象的锁定。
  - wait()、wait(long timeoutMillis)、wait(long timeoutMillis, int nanos)
    > **final** native方法，调用的均是wait(long timeoutMillis)方法，使当前线程放弃对该对象的监视器锁，等待被唤醒或者时间达到超时时间，之后与其他等待中的线程公平竞争该对象的锁。
    >> **wait() == wait(0L)，并不是只放弃锁之后立刻进入对象锁等待池，而是一直等待直到被notify方法唤醒**。
    
#### String
  - （private final byte[] value） & （private final byte coder） & （static final boolean COMPACT_STRINGS）
    > jdk9开始使用byte数组存储字符，存储时字符编码方式有两种，Latin1（iso-8859-1）和Utf-16，分别对应的coder值为0和1，COMPACT_STRINGS代表jvm是否允许压缩字符，如果允许压缩字符并且字符串中字符全在Latin1能表示的范围内，那么会使用Latin1编码，这么一个字符只会占用一个字节，否则会占用两个字节。

    > unicode字符集：是一种通用字符集，它的编码空间可以划分为17个平面（plane），每个平面包含65,536个码位（code point），第一个平面称为基本多语言平面（BMP），其他平面称为辅助平面，字符集只是指定了字符的编号，但是却有多种编码方式去表示这个编号。

    > utf-16：unicode字符集规定的标准编码实现，**也是jvm运行时字符的默认编码方式**，固定使用两个字节去表示BMP（包括中文）内的字符，BMP之外的字符则需要四个字节去表示。

    > utf-8：可变长字符编码，使用1到4个字节表示一个字符，对于ASCII字符utf-8编码的表示与其相同，中文字符却需要三个字节，**java字节码文件是使用utf-8编码的**。

  - String(String original)
    > 该构造函数会创建一个新的String对象，相当于浅拷贝，即会共用value数组。
  - getBytes(String charsetName)
    > 获取该字符串使用指定编码方式的表示，对于unicode字符集BMP外的字符（如emoji表情），每一个code point需要四个字节。
    >> 如果是utf-16会比预计多出两个字节，原因是其以两个字节为编码单元，要考虑字节序，所以返回的字节数组需要使用额外的两个字节表示来标志字节序（FEFF或FFFE），指定UTF-16BE或者UTF-16LE就会是正确的字节表示。
  - length()
    > value.length >> coder()，这也是为什么coder值为0和1的原因，**但需要知道对于BMP外的字符是占用四个字节的，所以该返回值可以说并不是完全正确**。
  - char charAt(int index) & int codePointAt(int index)
    > 同样BMP外的字符是无法使用char表示的，因为char只占两个字节，codePointAt的返回值则是int。
  - equals()
    > 重写了equals方法，与字符串常量比较时可以使用==，但是**两个变量比较时要使用equals方法**，value值会被复用，但String对象是可以存在多个的，虽然它们都指向内存中同一个value。
  - strip()（jdk11） & trim() （**不推荐继续使用**）
    > 移除字符串两侧的空白字符，trim()方法移除空格、tab键、换行符，而strip()方法移除所有Unicode空白字符。
    >> strip()和isBlank()都是使用Character.isWhitespace()实现，能识别所有Unicode空白字符。
  - split(String regex)
    > 参数包含正则表达式的转义符则需要使用\\\\转义，否则将作为正则表达式去匹配字符串。
 
#### StringBuffer & StringBuilder
  - 相同点
    > 均继承自AbstractStringBuilder，均是可变字符串。
  - 不同点
    > StringBuffer的方法均使用了synchronized修饰，所以它是线程安全的。
    
    > StringBuffer包含一个toString方法的字串缓冲区（private transient String toStringCache）。
  - AbstractStringBuilder
    > 设计与String基本一致，因为是可变字符串，拥有一个count字段来标识可变字符串的实际长度，与Arraylist一样，默认容量为16，扩容时变为原容量的两倍加2（why?）。

#### Enum（所有枚举的父类）
  - clone()：会抛出CloneNotSupportedException，因为是绝对的单例模式所以无法被克隆。
  - valueOf()：如果枚举常量不存在会抛出IllegalArgumentException，而不会返回null。

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

#### Thread

#### ThreadLocal

#### InheritableThreadLocal

#### ThreadGroup
