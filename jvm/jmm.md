# [首页](/blog/)

> Java内存模型（JMM）

***

## Java Memory Model

- JMM是一种抽象的概念，并不真实存在，它描述的是一组规范，通过这组规范定义了Java程序中各个变量（包括实例字段，静态字段和构成数组对象的元素）的访问方式；
- 用于屏蔽掉各种硬件和操作系统的内存访问差异，以实现让Java程序在各种平台下都能达到一致的并发效果；
- **规定了一个线程如何和何时可以看到由其他线程修改过后的共享变量的值，以及在必须时如何同步的访问共享变量。**

### 规范

- 调用栈和本地变量存放在线程栈上，对象存放在堆上。
- 对象方法中的本地变量仍然存放在线程栈上，即使这些方法所属的对象存放在堆上。
- 对象的成员变量**可能**随着这个对象自身存放在堆上（对象逃逸分析）。
- 静态成员变量跟随着类定义一起存放在堆上。
- **线程使用堆内存中的变量时，需要将其复制到本地内存中（为抽象概念，包括缓存，缓冲区，寄存器等）。**

### happens-before

如果操作A happen before 操作B，那么操作A在内存上所做的操作对操作B都是可见的，不管它们在不在一个线程。

> **这并不意味着Java平台的具体实现必须要按照happens-before关系指定的顺序来执行，只要保证重排序之后的执行结果与按happens-before关系来执行的结果一致即可。**

***

## 主内存和工作内存

- **主内存**：主要存储的是Java实例对象，当然也包括了共享的类信息、常量、静态变量，即包括了JVM中的堆和方法区。**由于是共享数据区域故会发生线程安全问题**。
- **工作内存**：每个线程都有一块用于存储私有数据的工作内存，**工作内存中存储着主内存中的变量副本拷贝**，即包括JVM中的程序计数器、虚拟机栈以及本地方法栈。**由于线程间无法相互访问工作内存故不存在线程安全问题**。

### JMM定义的用于**主内存和工作内存之间同步**的原子性变量操作

- lock（锁定）：作用于**主内存变量**，把一个变量标识为线程独占状态。
- unlock（解锁）：作用于**主内存变量**，释放一个被锁定的变量。
- read（读取）：作用于**主内存**变量，把一个变量值从主内存传输到线程的工作内存中，**供load使用**。
- load（载入）：作用于**工作内存**变量，把read从主内存中得到的变量值放入工作内存的变量副本中。
- use（使用）：作用于**工作内存**变量，把工作内存中的一个变量值传递给执行引擎。
- assign（赋值）：作用于**工作内存**变量，把一个从执行引擎接收到的值赋值给工作内存变量。
- store（存储）：作用于**工作内存**变量，把工作内存中的一个变量的值传送到主内存中，**供write使用**。
- write（写入）：作用于**主内存**变量，把store从工作内存中一个变量的值传送到主内存的变量中。
  
> **Java内存模型只要求read和load操作、store和write操作顺序执行，而没有保证必须是连续执行。**

### 解决多线程问题

- **原子性**：
    - **除了64位的long和double，JMM对没有被volatile修饰基本数据类型的读写操作是原子性的**。32位系统下每次都是操作32位，不足的32位则补全至32位，64位则需要超过两次，故无法保证原子性。
    - 也可以使用synchronized关键字和lock操作实现原子性，即限制只有一个线程能操作。
- **可见性**：
  - 使用volatile关键字保证，修改变量后会立即把值刷新到主内存，使用变量时每次都会从主内存中读取。
  - synchronized关键字和lock操作也能保证，**释放锁之前会将修改的变量刷新到主内存中**。
- **有序性**：
  - volatile关键字能保证一定的有序性，通过内存屏障的方式禁止指令重排序。
  - synchronized关键字和lock操作加锁之后当然也是保证了有序性。

***

## volatile

### 作用

**可以将对volatile变量的读取操作视作加上了同步的get和set方法，只是没有显式加锁操作。**

- 保证变量可见性：通过lock操作实现
  - 修改变量后会立即把值刷新到主内存；
  - 使用变量时每次都会从主内存中读取。
  
- 禁止指令重排序：通过内存屏障实现
  - 每个volatile读操作之后先插入一个LoadLoad屏障（确保在volatile读操作之后的所有普通读操作不会被重排序到volatile读操作之前），再插入一个LoadStore屏障（确保在volatile读操作之后的所有普通写操作不会被重排序到volatile读操作之前）；
  - 在每个volatile写操作之前插入一个StoreStore屏障（确保在volatile写操作之前的所有普通写操作都对其他线程可见），之后插入一个StoreLoad屏障（确保在volatile写操作之后的所有普通读操作不会被重排序到volatile写操作之前）。

### 指令重排序

是减少CPU中断的一种技术，其可以保证结果与顺序执行的结果一致，但是没有义务保证与并行执行的结果一致。

- 编译器优化重排：不改变单线程程序语义的前提下，可以重新安排语句的执行顺序。
- 指令并行重排：不存在数据依赖性情况下，改变语句对应的机器指令的执行顺序，以将多条指令重叠执行。
- 内存系统重排：由于处理器使用缓存和读/写缓存冲区，这使得加载和存储操作看上去可能是在乱序执行。

### 内存屏障

**保证屏障之前和之后的代码不能交换执行顺序**。写屏障会让写入缓存中的最新数据写入主内存，读屏障会使得高速缓冲区的数据失效强制重新从主内存中加载数据。


- LoadLoad：后面的load操作的数据被访问前，前面的load操作的数据已经读取完毕；
- LoadStore：后面的store操作写入前保证前面的load操作的数据已经读取完毕；
- StoreStore：后面的store操作写入前保证前面的store操作写入的值对其他线程可见；
- StoreLoad：万能屏障，包括前面三种。

### 既然CPU有缓存一致性协议（MESI），为什么JMM还需要volatile关键字？

MESI协议是应用最广泛的多核CPU缓存一致性协议，但CPU并没有严格遵守，因为会降低执行效率；而volatile是Java语言的保证，用来防止指令重排，无论是单核还是多核都需要。

```java
public class Demo {
    private static boolean running = true;
    private static int count = 0;

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            while (running) {
                count++;
            }
        });
        Thread t2 = new Thread(() -> running = false);
        t1.start();
        Thread.sleep(1000);
        t2.start();
        t1.join();
        t2.join();
    }
}
```
此代码运行结果是t1线程会进入死循环无法退出，解决方案是使用volatile修饰running；**但并不是因为可见性的原因导致**，而是由于JIT优化，t2线程修改running值前，t1线程已经运行了1秒中，t1中的代码会被判断热点代码，进而直接将running替换为true，导致代码进入死循环；如果关闭掉JIT优化或者注释掉sleep代码，由于CPU缓存一致性协议，最终t1仍会读取到最新的running值并退出循环。

### 单例模式的双重检查锁为什么需要volatile?

```java
private volatile static Singleton instance = null;
 
public static Singleton getInstance() {
    if (instance == null) { // 保证可见性
        synchronized (Singleton.class) {
            if (instance == null) {
                instance = new Singleton();
            }
        }
    }
    return instance;
}
```
初始化操作分为三步（1）new：创建对象实例，分配内存空间、（2）invokespecial：调用构造器方法，初始化对象、（3）aload_0：存入局部方法变量表，synchronized并不能防止指令重排，故如果（2）（3）发生了重排，则可能出现当其他线程判断instance非空时，instance指向的对象还未调用构造器方法。

***

## 安全发布

### 不正确的发布

```java
public Holder holder;

public void init() {
    holder = new Holder(42);
}

public class Holder {
    private int n;

    pulic Holder(int n) { 
        this.n = n; 
    }

    public void assert() {
        if (n != n) {
            throw new AssertError();
        }
    }
}
```
多线程执行assert方法时可能抛出异常，因为域n不是final的，在Object的构造函数中会先将默认值0写入n中。

### **安全发布对象的方式**

- 在静态初始化函数中初始化对象；
- 将对象的的引用保存在volatile类型的域或AtomicRefrence对象内；
- 将对象的引用保存在final类型域中；
- 将对象的引用保存到一个由锁保护的域中。

***