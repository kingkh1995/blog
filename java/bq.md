# [首页](/blog/)

> version: **jdk17**

> **BlockingQueue(TODO)**

***

## **BlockingQueue extends Queue**

阻塞队列，线程安全的队列，主要用于生产者-消费者模式，不允许接受null元素。有四种操作模式，抛出异常和立即返回结果来自于Queue接口，**提供了可阻塞的put和take，以及支持定时的offer和poll**。

### ArrayBlockingQueue

数组实现的有界阻塞队列，只记录队首和队尾的索引，不会产生和销毁额外对象，GC压力小。只使用了一个ReentrantLock，可以设置是否使用公平锁，默认非公平锁。

### LinkedBlockingQueue

链表实现，可有界可无界（容量为Integer.MAX_VALUE），需要创建和销毁Node对象，GC压力大。put和take各使用了一个非公平的ReentrantLock，并发效率更高。

### DelayQueue

无界的延时队列，元素必须实现Delayed接口，使用以延时时间维护的PriorityQueue保存元素，只有元素延迟时间已到才允许被获取。只使用了一个非公平的ReentrantLock，**但使用leader属性保存了第一个阻塞等待获取元素的线程，take操作等同于是公平的**。

### DelayedWorkQueue

ScheduledThreadPoolExecutor使用的队列，与DelayQueue设计相同。不同之处是自己使用数组实现了堆，**堆内元素是ScheduledFutureTask，保存了其在堆中的索引，则remove操作不再需要遍历堆查找到元素**。

### PriorityBlockingQueue

基于优先级的无界阻塞队列，使用Object数组实现，只使用了一个非公平的ReentrantLock。

### SynchronousQueue (TODO)

特殊的阻塞队列，**不保存元素**，支持公平和非公平的方式。每个put必须等待一个take，反之亦然，都是使用内部类Transfer的transfer方法实现。

- TransferQueue：公平模式下使用，队列方式。
- TransferStack：非公平模式下使用，栈方式。

***