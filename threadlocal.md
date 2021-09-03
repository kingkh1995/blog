## [首页](https://kingkh1995.github.io/blog/)
> ThreadLocal增强实现

***

### ThreadLocal

- int threadLocalHashCode: 哈希值属性，通过AtomicInteger类型的静态属性nextHashCode获取，增长步幅为一个神奇数字0x61c88647，产生的哈希码能均匀的分布在2的N次方的数组里（原理是斐波那契散列法）。

- withInitial(Supplier)：静态工厂方法，返回一个SuppliedThreadLocal子类对象，该子类重写了initialValue方法，修改返回值null为通过Supplier获取。

- get(): 调用ThreadLocalMap的getEntry的方法，第一次调用时通过调用setInitialValue方法初始化。

- set(): 调用ThreadLocalMap的set方法。

- remove(): 调用ThreadLocalMap的remove方法。

- createMap()：用于初始化ThreadLocalMap，创建一个ThreadLocalMap，并赋值到线程的threadLocals属性。

- static createInheritedMap()：静态方法，线程创建时如果父线程inheritableThreadLocals属性不为空则会调用该方法将其复制到子线程的inheritableThreadLocals属性，拷贝时调用childValue方法复制value。

- childValue：createInheritedMap中调用，默认实现抛出UnsupportedOperationException，即不支持线程继承。

***

### ThreadLocalMap

> ThreadLocal内部类，ThreadLocal真正的核心。为特殊设计的map，底层为Entry数组，初始容量16，容量要求为2的次方，使用线性探测法解决冲突，阈值为2/3容量。

> 每个线程都包含两个ThreadLocalMap类型的属性，threadLocals和inheritableThreadLocals，threadLocals是在首次调用ThreadLocal方法时初始化，inheritableThreadLocals是在创建子线程时赋值。

#### Entry

> ThreadLocalMap内部类，继承自WeakReference，弱引用ThreadLocal作为key，增加的value属性为强引用，如果ThreadLocal已被回收，则视为过期，可以对Entry进行清除。

#### 软引用、弱引用、虚引用

> 都继承自Reference<T>，创造一个对象的引用对象，只有当对象不存在强引用时才可能被回收。

- Reference:

    - 构造方法：单个对象的构造方法，或单个对象和一个ReferenceQueue的构造方法。

    - get()：通过该方法获取被引用对象，如已被回收或者为虚引用则会返回null。

    - ReferenceQueue：被引用对象被回收时，会将引用对象加入ReferenceQueue队列中，之后取出引用对象调用get均返回null。

- 区别：
    - 软引用：gc后可用内存仍然不足，会回收只存在软引用的对象。

    - 弱引用：每次gc时均会回收只存在弱引用的对象。

    - 虚引用：创建时必须指定ReferenceQueue，创建引用后就无法通过虚引用获取到对象，每次gc时均会回收只存在虚引用的对象。

#### ThreadLocalMap源码

- getEntry()：通过ThreadLocal的哈希值计算出索引值，获取到Entry数组中的键，直接使用==对比传入的ThreadLocal和Entry引用的ThreadLocal，如果匹配则返回否则执行getEntryAfterMiss操作。

- getEntryAfterMiss()：进行线性探测查询，如果探测到已过期键则执行expungeStaleEntry操作清除过期键。

- expungeStaleEntry()：检查当前键簇剩余所有键，删除过期键，有效键则重新插入。

- set()：进行线性探测，如果键已存在则覆盖旧值并返回，如果探测到已过期键则执行replaceStaleEntry操作并返回，否则插入新键，然后执行一次cleanSomeSlots操作，如cleanSomeSlots操作未清除过期键，并且键数量已经超过了阈值，则进行rehash操作。

- replaceStaleEntry()：检查当前键簇所有键，如果查找到键则替换否则插入到当前过期键的位置，如果当前键簇还存在其他过期键，则全部清除，之后从当前键簇尾部开始执行一次cleanSomeSlots操作。

- cleanSomeSlots()：从给定位置开始探测一段长度，清除期间所有过期键，探测长度为log2(n)，n为入参。

- rehash()：检查所有键，清除所有过期键，之后如果键数量仍大于阈值的3/4（容量的1/2）扩容为两倍大小。

- threshold：线性探测法最佳使用率是1/8到1/2，而ThreadLocalMap的阈值却是2/3容量。是因为存在过期键，由于所有操作都会顺带清除过期键，所以允许使用率达到2/3，但是真实使用率要保证不能超过1/2。rehash时，先清除了所有过期键，故扩容判断的阈值为1/2容量。

```java
    // 这段代码有问题，应该是threshold = len - len / 3
    private void setThreshold(int len) {
        threshold = len * 2 / 3;
    }
```

- remove()：线性探测，找到键并执行expungeStaleEntry操作。

***

### InheritableThreadLocal extends ThreadLocal

> 可继承的ThreadLocal，子线程能获取到父线程中ThreadLocal的值，实现逻辑是创建线程时将父线程的ThreadLocalMap复制到子线程中，故ThreadPoolExecutor、ForkJoinPool不会生效，因为线程池中线程会被复用，且会有上下文丢失和内存泄漏的问题，**不推荐在线程池中使用**。

- 重写了createMap方法，创建ThreadLocalMap并赋值到线程的inheritableThreadLocals变量。

- 重写了getMap方法，返回线程的inheritableThreadLocals变量。

- 重写了childValue方法，直接返回原对象而不是拷贝。

***

### TransmittableThreadLocal extends InheritableThreadLocal
> InheritableThreadLocal增强实现，在线程池中也可以生效，通过其crr机制转移TTL值，且能保证上下文不丢失和不会出现内存泄露。

- holder：静态属性，InheritableThreadLocal<WeakHashMap<TransmittableThreadLocal<Object>, ?>>类型，用于记录当前线程使用过的ThreadLocal，WeakHashMap的value均设为null被当作set使用。

#### Transmitter
> 内部工具类，转移器，转移逻辑实现。

- capture()：先抓取线程A的ThreadLocal值，holder中的TTL（所有）和threadLocalHolder中的ThreadLocal（手动注册）。

- replay()：在线程B中回放抓取到的值，并返回回放前TTL值的备份。

- restore()：任务执行完毕后使用备份恢复到回放前的状态。

- threadLocalHolder：Transmitter类静态属性，volatile WeakHashMap<ThreadLocal<Object>, TtlCopier<Object>>类型，用于转移ThreadLocal值，需要手动调用registerThreadLocal注册。

- registerThreadLocal()：手动注册ThreadLocal到threadLocalHolder属性，不能用于注册TTL，**主要是作为兼容升级方案，不太建议使用**。

#### TtlRunnable、TtlCallable
> 装饰器模式，包装Runnable和Callable，创建实例时使用capture，执行run方法前replay，执行后restore。

#### TtlExecutors
> 提供静态工厂方法将线程池包装成对应的TTL线程池类。

#### ExecutorTtlWrapper、ExecutorServiceTtlWrapper、ScheduledExecutorServiceTtlWrapper
> 线程池包装类，重写了线程池的方法，将传入的Runnable和Callable包装成TtlRunnable和TtlCallable并交由原线程池执行。

***

#### InternalThreadLocal
> 特殊设计的ThreadLocal类（非子类），参照Netty的FastThreadLocal，核心思路是空间换时间，dubbo的RpcContext和FutureContext中使用。

> ThreadLocal能清除过期键，但是使用场景上需要非static的ThreadLocal的情况不多见，所以其为了清除过期键的复杂设计会一定程度的影响效率，不考虑清除过期键就是InternalThreadLocal对比ThreadLocal的优势。

- index：使用index而不是ThreadLocal的hashcode，通过AtomicInteger自增获取，步幅为1。

- get()：调用InternalThreadLocalMap的get静态方法获取到线程的InternalThreadLocalMap，然后通过index获取索引位置的值，如果不存在则初始化。

- set()：value为null或为UNSET时移除值（**ThreadLocal不会移除值**），否则调用InternalThreadLocalMap的get静态方法获取到线程的InternalThreadLocalMap，然后设置到index索引位置，之后记录到variablesToRemove。

- remove：调用InternalThreadLocalMap的getIfSet静态方法获取到线程的InternalThreadLocalMap，将槽设置为UNSET，之后从variablesToRemove中移除。

- removeAll()：移除所有的ITL值，并移除InternalThreadLocalMap，**建议在拦截器中手动调用**。

#### InternalThreadLocalMap
> 特殊设计的ThreadLocalMap，底层为Object数组，完全去除了ThreadLocal中清除过期键和线性探测的操作，因为是通过index属性确定槽，而index是通过一个全局的AtomicInteger获取，**所以使用InternalThreadLocal时必须要设置为static**，否则随着InternalThreadLocal数量的增加，必然会造成空间的极大浪费，同时index也会自增到上限导致应用无法继续运行。

- Object[] indexedVariables：初始大小32。

- slowThreadLocalMap：静态属性，ThreadLocal<InternalThreadLocalMap>类型，线程非InternalThread类型情况下使用。

- variablesToRemove：记录使用中的InternalThreadLocal，为Set（使用IdentityHashMap并包装为SetFromMap），保存在InternalThreadLocalMap中索引值为0的槽上。

- get()：静态方法，获取线程的InternalThreadLocalMap，如果不存在则初始化。

- fastGet()：线程为InternalThread类型情况调用，返回InternalThread的threadLocalMap属性，不存在则初始化并赋值。

- slowGet()：线程不为InternalThread类型情况调用，通过静态属性slowThreadLocalMap获取，不存在则初始化并通过slowThreadLocalMap设置。

- getIfSet()：静态方法，获取线程的InternalThreadLocalMap，如未设置不初始化并返回null，获取方式与get方法一致。

- expandIndexedVariableTableAndSet()：set操作如果index超出了数组大小，则进行扩容，扩容为2的次方大小，并使用UNSET填充。

#### InternalThread extends Thread
> 继承自Thread，持有一个InternalThreadLocalMap类型变量。

#### InternalRunnable implements Runnable
> 装饰器模式，包装Runnable对象，执行完任务后调用InternalThreadLocal的removeAll方法，**不建议在线程池中使用（理由同不要在线程池中使用ThreadLocal的set/remove）**。

***