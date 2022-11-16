# [首页](/blog/)

> version: **jdk17**

***

## **Atomic**

### AtomicInteger

内部实现为使用Unsafe操作volatile的int变量。

- **Unsafe**：提供了一些绕开JVM的更低层次操作，并不建议使用。JDK9新增了jdk.internal.misc下的Unsafe类，并进行了重新设计，用于替换sun.misc下的原Unsafe类。

- Unsafe.weakCompareAndSetInt：用于根据指定偏移量使用CAS修改对象的变量，**仅保留了被操作的volatile变量的特性，无法保证其他变量的执行顺序和可见性**，默认实现为调用compareAndSet，但添加了@HotSpotIntrinsicCandidate，故允许HotSpot VM修改其实现以提升性能。

### LongAdder 

基于Striped64实现，而Striped64使用VarHandle类执行原子操作。

- **Striped64**：在高并发情况下CAS操作失败概率上升导致原子类效率降低，而Striped64类使用了**热点分离（空间换时间）** 的策略。其操作针对一个基本值和Cell数组，没有竞争时操作base值，存在竞争时不同线程针对不同的cell执行操作，竞争强度上升后会对Cell数组进行扩容，所以在低并发和高并发下都有很好的效率，**缺点是更新操作无法获取到实时的统计值**，只适用于并发统计的场景。

- **VarHandle**：变量句柄，JDK9新增，用于替代Unsafe类，JDK对其安全性和可移植性有保障，允许被用户使用。该类提供了多种对变量的访问模式，只能通过MethodHandles类的内部类Lookup创建。

### AtomicStampedReference

相比于AtomicReference，解决了ABA问题，操作的变量为私有内部类Pair，其为reference变量捆绑了版本号stamp变量，也是基于VarHandle类实现。

***

## **CopyOnWriteArrayList**

线程安全的List，COW写时复制思想设计，读写分离，适用于读多写少的并发场景。使用volatile和一个对象锁实现，读操作是直接读取效率高，写操作需要获取到对象的同步锁，写操作并不修改原数组，而是创建新数组，完成后使用新数组替换原数组。

***

## CopyOnWriteArraySet

线程安全的Set，使用CopyOnWriteArrayList实现。

- **获取线程安全的Set更推荐的方式：**
    - 使用ConcurrentHashMap的静态方法newKeySet()创建。
    - 使用Collections工具类的newSetFromMap方法包装ConcurrentHashMap。

***

## ConcurrentLinkedQueue （TODO）

线程安全的队列

***

### ConcurrentSkipListMap（TODO）

跳跃表，线程安全的NavigableMap。

***