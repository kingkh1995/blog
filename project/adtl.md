# [首页](/blog/)

> ThreadLocal增强

***

## Netty - FastThreadLocal

### FastThreadLocal

更快的ThreadLocal，核心思路是空间换时间；因为ThreadLocal一般都是作为静态变量使用，故ThreadLocal引用不存在被GC回收的场景，这样ThreadLocal清理过期键的功能反而会影响效率，FastThreadLocal完全摒弃了清理过期键的功能，并支持随机访问，效率明显优于ThreadLocal的线性探测法；**多实例ThreadLocal场景下不要使用**，会造成空间的大量浪费甚至内存溢出。

#### 属性

-   ```java
    private final int index;

    public FastThreadLocal() {
        index = InternalThreadLocalMap.nextVariableIndex(); // 静态AtomicInteger自增获取
    }
    ```
    **每个FastThreadLocal创建时都指定了一个index，作为其在InternalThreadLocalMap的索引。**

-   ```java
    private static final int variablesToRemoveIndex = InternalThreadLocalMap.nextVariableIndex();
    ```
    使用数组索引0的位置记录**所有使用中的FastThreadLocal**，由set、get、remove操作维护，用于removeAll操作，只需要遍历variablesToRemove，并调用FastThreadLocal的remove方法即可。

#### 方法

-   ```java
    public final V get() {
        InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get(); // 从当前线程中获取
        Object v = threadLocalMap.indexedVariable(index); // 使用index属性以O(1)查找到值
        if (v != InternalThreadLocalMap.UNSET) { // 因为值支持null，故使用UNSET表示未设置。
            return (V) v;
        }
        return initialize(threadLocalMap); // 不存在则初始化
    }

    private V initialize(InternalThreadLocalMap threadLocalMap) {
        V v = null;
        try {
            v = initialValue(); // 默认返回null
        } catch (Exception e) {
            PlatformDependent.throwException(e);
        }
        threadLocalMap.setIndexedVariable(index, v); // 设置到index位置
        addToVariablesToRemove(threadLocalMap, this); // 记录到variablesToRemove集合
        return v;
    }
    ```

-   ```java
    public final void set(V value) {
        if (value != InternalThreadLocalMap.UNSET) {
            InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
            // 保存到InternalThreadLocalMap的index位置，并记录FastThreadLocal到variablesToRemove集合。
            setKnownNotUnset(threadLocalMap, value); 
        } else {
            remove(); // 值为UNSET时移除
        }
    }

    public final void remove() {
        remove(InternalThreadLocalMap.getIfSet());
    }

    public final void remove(InternalThreadLocalMap threadLocalMap) {
        if (threadLocalMap == null) {
            return;
        }
        Object v = threadLocalMap.removeIndexedVariable(index); // 从InternalThreadLocalMap的index位置移除并返回值
        removeFromVariablesToRemove(threadLocalMap, this); // 从variablesToRemove集合移除FastThreadLocal
        ...
    }
    ```

### InternalThreadLocalMap

#### 属性

-   ```java
    public class FastThreadLocalThread extends Thread {
        private InternalThreadLocalMap threadLocalMap;
    }
    ```
    FastThreadLocalThread继承自Thread，增加了threadLocalMap属性以支持FastThreadLocal。

-   ```java
    private static final ThreadLocal<InternalThreadLocalMap> slowThreadLocalMap = new ThreadLocal<>();
    ```
    **非FastThreadLocalThread也可以使用FastThreadLocal**，将InternalThreadLocalMap保存至ThreadLocalMap内使用。

    ```java
    public static final Object UNSET = new Object(); // 因为支持值为null，用于标记未设置状态。

    private Object[] indexedVariables;
    ```
    使用对象数组保存值，FastThreadLocal的index属性作为其关联的值在数组内的索引。

#### 方法

-   ```java
    // 获取当前线程的InternalThreadLocalMap
    public static InternalThreadLocalMap getIfSet() {
        Thread thread = Thread.currentThread();
        if (thread instanceof FastThreadLocalThread) {
            // FastThreadLocalThread则直接从其threadLocalMap属性获取InternalThreadLocalMap。
            return ((FastThreadLocalThread) thread).threadLocalMap();
        }
        return slowThreadLocalMap.get(); // 其他类型Thread则通过ThreadLocalMap获取。
    }
    ```

-   ```java
    // 设置值，因为初始创建时固定容量即为32，故可能会不可避免的多次扩容。
    public boolean setIndexedVariable(int index, Object value) {
        Object[] lookup = indexedVariables;
        if (index < lookup.length) { // 数组长度够则直接设置
            Object oldValue = lookup[index];
            lookup[index] = value;
            return oldValue == UNSET;
        } else { // 不够则扩容为2的幂再设置
            expandIndexedVariableTableAndSet(index, value);
            return true;
        }
    }
    ```

***

## TTL - TransmittableThreadLocal

InheritableThreadLocal增强实现，用于在线程池场景下优雅的实现上下文传递，通过其crr机制传递TTL值，并保证上下文的隔离性，以及防止内存泄漏。

### class TransmittableThreadLocal\<T\> extends InheritableThreadLocal\<T\> implements TtlCopier\<T\>

- holder：静态属性，InheritableThreadLocal\<WeakHashMap\<TransmittableThreadLocal\<Object\>, ?>>类型，用于记录当前线程使用过的TTL，WeakHashMap被当作Set使用，键的value均设为null。

### Transmitter
> 内部工具类，转移器，转移逻辑实现。

- capture()：先抓取线程A的ThreadLocal值，包括TransmittableThreadLocal的holder属性记录的TTL（所有）和Transmitter的threadLocalHolder属性记录的其他ThreadLocal（手动注册）。

- replay()：在线程B中回放抓取到的值，并返回回放前TTL值的备份。

- restore()：线程B任务执行完毕后使用备份恢复到回放前的状态。

- *threadLocalHolder*：Transmitter类静态属性（volatile WeakHashMap<ThreadLocal<Object>, TtlCopier<Object>>），用于转移其他ThreadLocal值，需要手动调用registerThreadLocal方法注册。

- registerThreadLocal()：手动注册ThreadLocal到threadLocalHolder属性，不能用于注册TTL，**主要是作为兼容方案，不太建议使用**。

### TtlRunnable、TtlCallable
> 装饰器模式，包装Runnable和Callable。创建实例时使用capture，执行run方法前replay，执行后restore。

### TtlExecutors
> 工具类，提供静态工厂方法将线程池包装成对应的TTL线程池。

### ExecutorTtlWrapper、ExecutorServiceTtlWrapper、ScheduledExecutorServiceTtlWrapper
> 线程池包装类，重写了线程池的方法，将传入的Runnable和Callable包装成TtlRunnable和TtlCallable并交由原线程池执行。

***