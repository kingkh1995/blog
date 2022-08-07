# [首页](/blog/)

> version: **jdk17**

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
