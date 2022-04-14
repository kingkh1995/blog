# [首页](/blog/)

> version: **jdk17**

***
## Atomic

### AtomicInteger

内部实现为使用Unsafe操作volatile的int变量。

- Unsafe.weakCompareAndSetInt：用于根据指定偏移量使用CAS修改对象的变量，**仅保留了被操作的volatile变量的特性，无法保证其他变量的执行顺序和可见性**，默认实现为调用compareAndSet，但添加了@HotSpotIntrinsicCandidate，故允许HotSpot VM修改其实现以提升性能。

### LongAdder 
基于Striped64实现，而Striped64使用VarHandle实现。在高并发情况下CAS容易失败导致原子类效率降低，而Striped64类使用了热点分离和空间换时间的策略，将一个值分为多个cell之和，不同线程针对不同的cell执行CAS操作能提高成功率，cell的数量会随着CAS操作失败而增加，这种方式使得在低并发和高并发下都有很好的效率。

- VarHandle：变量句柄，jdk9新增，用于替代Unsafe类，只能通过MethodHandles类的内部类Lookup创建。jdk对其安全性和可移植性有保障，允许被用户使用，提供了多种对变量的访问模式。

### AtomicStampedReference
相比于AtomicReference，解决了ABA问题，私有内部类Pair为reference变量捆绑了版本号stamp变量，同样都是基于VarHandle实现。

***

## AQS

***