## [首页](https://kingkh1995.github.io/blog/)
> ThreadLocal增强实现

***

### ThreadLocal

- int threadLocalHashCode: 哈希值属性，通过一个全局的AtomicInteger获取，增长步幅为一个神奇数字0x61c88647，产生的哈希码能均匀的分布在2的N次方的数组里（原理是斐波那契散列法）。

- withInitial(Supplier)：静态工厂方法，返回一个SuppliedThreadLocal子类对象，该子类重写了initialValue方法，修改原返回值null为通过Supplier获取。

- get(): 调用ThreadLocalMap的getEntry的方法，第一次调用时通过调用setInitialValue方法初始化。

- set(): 调用ThreadLocalMap的set方法。

- remove(): 调用ThreadLocalMap的remove方法。

- createMap()：用于初始化ThreadLocalMap，创建一个ThreadLocalMap，并赋值到线程的threadLocals属性。

- static createInheritedMap()：静态方法，线程创建时如果父线程inheritableThreadLocals属性不为空（即存在可继承的ThreadLocal），则会调用该方法将其复制到子线程的inheritableThreadLocals属性，调用childValue方法拷贝value。

- childValue：createInheritedMap中调用，默认实现抛出UnsupportedOperationException，即ThreadLocal本身不支持线程继承。

***

### ThreadLocalMap

> ThreadLocal内部类，ThreadLocal真正的核心。特殊设计的Map，底层为Entry数组，初始容量16，容量要求为2的次方，使用线性探测法解决冲突，阈值为2/3容量。

> 每个线程都包含两个ThreadLocalMap类型的属性，threadLocals和inheritableThreadLocals，threadLocals是在首次调用ThreadLocal方法时初始化，inheritableThreadLocals是在创建子线程时赋值。

#### Entry

> ThreadLocalMap内部类，继承自WeakReference，弱引用ThreadLocal作为key，增加的value属性为强引用，如果ThreadLocal已被回收，则视为过期，可以对Entry进行清除。

#### 软引用、弱引用、虚引用

> 都继承自Reference<T>，创造一个对象的引用对象，只有当对象不存在强引用时才可能被回收。

- Reference:

    - 构造方法：单个对象的构造方法，或单个对象和一个ReferenceQueue的构造方法。

    - get()：通过该方法获取被引用对象，如已被回收或者为虚引用则会返回null。

    - ReferenceQueue：被引用对象被回收时，会将引用对象加入ReferenceQueue队列中。

- 区别：
    - 软引用：如gc后可用内存仍然不足，则会回收只存在软引用的对象。

    - 弱引用：每次gc均会回收只存在弱引用的对象。

    - 虚引用：创建时必须指定ReferenceQueue，创建虚引用后就无法通过虚引用获取到被引用对象，每次gc均会回收只存在虚引用的对象。

#### ThreadLocalMap源码

- getEntry()：通过ThreadLocal的threadLocalHashCode属性确定键的位置，获取到Entry后直接使用==对比传入的ThreadLocal和Entry引用的ThreadLocal，如果匹配则返回否则执行getEntryAfterMiss操作。

- getEntryAfterMiss()：进行线性探测，如果探测到已过期键则执行expungeStaleEntry操作清除过期键。

- expungeStaleEntry()：检查当前键簇剩余所有键，删除过期键，有效键则重新插入。

- set()：进行线性探测，如果键已存在则设置值并返回，如果探测到已过期键则执行replaceStaleEntry操作并返回，否则插入新键，然后执行一次cleanSomeSlots操作，如cleanSomeSlots操作未清除过期键，并且键数量已经超过了阈值，则执行rehash操作。

- replaceStaleEntry()：检查当前键簇所有键，如果过程中查找到键则设置值否则替换当前过期键，如果当前键簇还存在其他过期键，则全部清除，以上执行完后从当前键簇尾部开始执行一次cleanSomeSlots操作。

- cleanSomeSlots()：从给定位置开始探测一段长度，清除期间所有过期键，探测长度为log2(n)，n为入参。

- rehash()：检查所有键，清除所有过期键，执行完之后如果键数量仍大于阈值的3/4（容量的1/2）则扩容为两倍大小。

- **threshold**：线性探测法最佳使用率是1/8到1/2，而ThreadLocalMap的阈值却是2/3容量。是因为存在过期键，且所有操作都会顺带清除过期键，所以允许使用率达到2/3，但是真实使用率要保证不能超过1/2。rehash操作会先清除所有过期键，故resize判断阈值为1/2容量。

```java
    // 这段代码有问题，应该是threshold = len - len / 3
    private void setThreshold(int len) {
        threshold = len * 2 / 3;
    }
```

- remove()：线性探测，找到键并执行expungeStaleEntry操作。

***

### InheritableThreadLocal extends ThreadLocal

> 可继承的ThreadLocal，在父线程中设置的InheritableThreadLocal值在子线程也能获取到，实现逻辑是创建线程时将父线程的ThreadLocalMap复制到子线程中。

> 使用ThreadPoolExecutor、ForkJoinPool不会生效，因为线程池中线程会被复用，并不是每次提交任务的时候都会创建线程。且会有上下文丢失和内存泄漏的问题，**不推荐在线程池中使用InheritableThreadLocal**。

- 重写了createMap方法，创建ThreadLocalMap并赋值到线程的inheritableThreadLocals变量。

- 重写了getMap方法，返回线程的inheritableThreadLocals变量。

- 重写了childValue方法，直接返回原对象而不是进行拷贝。

***

### TransmittableThreadLocal extends InheritableThreadLocal
> InheritableThreadLocal增强实现，在线程池中也可以生效，通过其crr机制转移TTL值，且能保证上下文不丢失和不会出现内存泄露。

- holder：静态属性，InheritableThreadLocal<WeakHashMap<TransmittableThreadLocal<Object>, ?>>类型，用于记录当前线程使用过的TTL，WeakHashMap被当作Set使用，键的value均设为null。

#### Transmitter
> 内部工具类，转移器，转移逻辑实现。

- capture()：先抓取线程A的ThreadLocal值，包括TransmittableThreadLocal的holder属性记录的TTL（所有）和Transmitter的threadLocalHolder属性记录的其他ThreadLocal（手动注册）。

- replay()：在线程B中回放抓取到的值，并返回回放前TTL值的备份。

- restore()：线程B任务执行完毕后使用备份恢复到回放前的状态。

- *threadLocalHolder*：Transmitter类静态属性（volatile WeakHashMap<ThreadLocal<Object>, TtlCopier<Object>>），用于转移其他ThreadLocal值，需要手动调用registerThreadLocal方法注册。

- registerThreadLocal()：手动注册ThreadLocal到threadLocalHolder属性，不能用于注册TTL，**主要是作为兼容方案，不太建议使用**。

#### TtlRunnable、TtlCallable
> 装饰器模式，包装Runnable和Callable。创建实例时使用capture，执行run方法前replay，执行后restore。

#### TtlExecutors
> 工具类，提供静态工厂方法将线程池包装成对应的TTL线程池。

#### ExecutorTtlWrapper、ExecutorServiceTtlWrapper、ScheduledExecutorServiceTtlWrapper
> 线程池包装类，重写了线程池的方法，将传入的Runnable和Callable包装成TtlRunnable和TtlCallable并交由原线程池执行。

***

### InternalThreadLocal
> 特殊设计的ThreadLocal类（非子类），在Dubbo的RpcContext和FutureContext中使用，**核心思路是空间换时间**。

> ThreadLocal的优势是能清除过期键，缺点是会一定程度的影响效率，且因为多实例ThreadLocal的使用场景很少见，所以清除过期键的设计必要性不高，同时线性探测法也不如随机访问快。

- index：使用index而非hashcode访问，通过AtomicInteger自增获取，步幅为1。

- get()：调用InternalThreadLocalMap的get静态方法获取到线程的InternalThreadLocalMap，再通过index属性随机访问，如果不存在则初始化。

- set()：值为null或UNSET（*new Object()*）时进行移除（**ThreadLocal不会移除值**），否则调用InternalThreadLocalMap的get静态方法获取到线程的InternalThreadLocalMap，然后设置到index位置的槽上，之后将该InternalThreadLocal记录到variablesToRemove。

- remove：调用InternalThreadLocalMap的getIfSet静态方法获取到线程的InternalThreadLocalMap，将槽设置为UNSET，之后将该InternalThreadLocal从variablesToRemove中移除。

- removeAll()：移除所有的ITL值，并移除线程的InternalThreadLocalMap，**建议在拦截器中手动调用**。

#### InternalThreadLocalMap
> 特殊设计的ThreadLocalMap，没有清除过期键操作，能通过InternalThreadLocal的index属性随机访问。

> 因为index是通过一个全局的AtomicInteger获取，**所以使用InternalThreadLocal时一定不要重复创建实例，而应该设置为static**，否则随着InternalThreadLocal数量的增加，index的自增，必然会造成空间的极大浪费，也最终会导致应用无法继续运行。

- Object[] indexedVariables：底层实现为Object数组，初始大小32。

- slowThreadLocalMap：静态属性，ThreadLocal<InternalThreadLocalMap>类型，线程非InternalThread类型情况下使用。

- **variablesToRemove**：使用Set记录当前线程使用的所有InternalThreadLocal（底层为IdentityHashMap并包装为SetFromMap），保存到线程的InternalThreadLocalMap中索引值为0的槽上。

- get()：静态方法，获取线程的InternalThreadLocalMap，根据不同情况调用fastGet和slowGet获取。

- fastGet()：线程为InternalThread情况下调用，返回InternalThread的threadLocalMap属性，不存在则初始化并赋值。

- slowGet()：线程为非InternalThread情况下调用，通过静态属性slowThreadLocalMap获取，不存在则初始化并通过slowThreadLocalMap设置。

- getIfSet()：静态方法，获取线程的InternalThreadLocalMap，如未初始化则返回null，获取方式与get方法相同。

- expandIndexedVariableTableAndSet()：set操作如果index超出了数组大小，则进行扩容，扩容为2的次方大小，并使用UNSET填充。

#### InternalThread extends Thread
> 继承自Thread，持有一个InternalThreadLocalMap类型的属性threadLocalMap。

#### InternalRunnable implements Runnable
> 装饰器模式，包装Runnable对象，执行完任务后调用InternalThreadLocal的removeAll方法，**不建议在线程池中使用（理由同不要在线程池中使用ThreadLocal的set/remove）**。

***