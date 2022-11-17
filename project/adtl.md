# [首页](/blog/)

> ThreadLocal增强

***

## Netty - FastThreadLocal

### **FastThreadLocal**

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
    ```
    
-   ```java
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

### **InternalThreadLocalMap**

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

### class **TransmittableThreadLocal**\<T\> extends InheritableThreadLocal\<T\> implements TtlCopier\<T\>

#### 属性

-   ```java
    // 将WeakHashMap作为Set使用，值默认为null。
    private static final InheritableThreadLocal<WeakHashMap<TransmittableThreadLocal<Object>, ?>> holder =
        new InheritableThreadLocal<WeakHashMap<TransmittableThreadLocal<Object>, ?>>() {
            ...
        };
    ```
    使用InheritableThreadLocal记录使用过的TTL，由get、set、remove方法维护，使用ttlTransmittee。

#### 方法

-   ```java
    @Override
    public final T get() {
        T value = super.get();
        if (disableIgnoreNullValueSemantics || value != null) addThisToHolder();
        return value;
    }

    @Override
    public final void set(T value) {
        if (!disableIgnoreNullValueSemantics && value == null) {
            remove();
        } else {
            super.set(value);
            addThisToHolder();
        }
    }

    @Override
    public final void remove() {
        removeThisFromHolder();
        super.remove();
    }
    ```
    get、set、remove均是直接使用父类InheritableThreadLocal的方法，并调用addThisToHolder方法和removeThisFromHolder方维护holder集合。

-   ```java
    // TtlCopier
    public T copy(T parentValue) {
        return parentValue;
    }
    ```
    capture方法调用，用于抓取ThreadLocal值。

### class **Transmitter**

工具类，转移器，用于注册Transmittee，供TtlRunnable、TtlCallable、TtlRecursiveTask、TtlRecursiveAction使用。

#### interface **Transmittee**

```java
public interface Transmittee<C, B> {

    // 1、抓取：创建任务时抓取当前线程的ThreadLocal值保存至任务对象中；
    C capture(); 

    // 2、回放：任务执行前将抓取的ThreadLocal值设置到任务执行线程中，并备份原ThreadLocal值后返回；
    @NonNull
    B replay(@NonNull C captured);

    // 清理：任务执行前清理ThreadLocal值并备份；
    @NonNull
    B clear();

    // 3. 恢复：任务执行的finally阶段，将备份的原ThreadLocal值重新设置到执行线程中。
    void restore(@NonNull B backup);
}
```
转移者的crr机制

#### 属性

-   ```java
    private static final Set<Transmittee<Object, Object>> transmitteeSet = new CopyOnWriteArraySet<>();

    static {
        registerTransmittee(ttlTransmittee); // TransmittableThreadLocal无需注册
        registerTransmittee(threadLocalTransmittee); // ThreadLocal兼容，需要手动注册。
    }
    ```
    使用registerTransmittee方法注册Transmittee。

-   ```java
    // 值为TtlCopier，表示如何抓取ThreadLocal值，默认为shadowCopier，即值复制。
    private static volatile WeakHashMap<ThreadLocal<Object>, TtlCopier<Object>> threadLocalHolder = new WeakHashMap<>();

    private static final TtlCopier<Object> shadowCopier = parentValue -> parentValue;
    ```
    需要手动调用registerThreadLocal方法注册其他类型ThreadLocal，使用threadLocalTransmittee。

#### CRR机制

```java
public static Object capture() { ... }

public static Object replay(Object captured) { ... }

public static Object clear() { ... }

public static void restore(Object backup) { ... }
```
静态的capture、replay、restore及clear方法，均是遍历transmitteeSet并调用其对应方法。

### **TtlRunnable & TtlCallable**

使用了装饰器模式，调用Transmitter的crr方法。

```java
public final class TtlCallable<V> implements Callable<V>, TtlWrapper<Callable<V>>, TtlEnhanced, TtlAttachments {
    private final AtomicReference<Object> capturedRef; // 创建时抓取的上下文
    private final Callable<V> callable; // 被包装的任务

    public V call() throws Exception {
        final Object captured = capturedRef.get(); // 1、获取抓取上下文；
        ...
        final Object backup = replay(captured); // 2、任务执行前将抓取的上下文设置到当前线程内；
        try {
            return callable.call(); // 3、执行任务，任务能获取到抓取的上下文；
        } finally {
            restore(backup); // 4、恢复当前线程原上下文。
        }
    }

    ...
}
```

### **TtlRecursiveAction & TtlRecursiveTask**

ForkJoinTask任务，调用Transmitter的crr方法。

```java
public abstract class TtlRecursiveTask<V> extends ForkJoinTask<V> implements TtlEnhanced {
    private final Object captured = capture(); // 创建时抓取的上下文

    protected final boolean exec() {
        final Object backup = replay(captured);
        try {
            result = compute();
            return true;
        } finally {
            restore(backup);
        }
    }

    ...
}
```

### **TtlExecutors**

工具类

-   ```java
    public static Executor getTtlExecutor(@Nullable Executor executor) {
        if (TtlAgent.isTtlAgentLoaded() || executor == null || executor instanceof TtlEnhanced) {
            return executor;
        }
        return new ExecutorTtlWrapper(executor, true);
    }

    class ExecutorTtlWrapper implements Executor, TtlWrapper<Executor>, TtlEnhanced {
        private final Executor executor; // 被包装的线程池

        @Override
        public void execute(@NonNull Runnable command) {
            // 将Runnable包装为TtlRunnable提交
            executor.execute(TtlRunnable.get(command, false, idempotent));
        }

        ...
    }
    ```
    将线程池包装为对应的TTL线程池（ExecutorTtlWrapper、ExecutorServiceTtlWrapper、ScheduledExecutorServiceTtlWrapper），实现则是将任务包装为对应的TTL任务类。

-   ```java
    public static ThreadFactory getDisableInheritableThreadFactory(@Nullable ThreadFactory threadFactory) {
        ...
        return new DisableInheritableThreadFactoryWrapper(threadFactory);
    }

    class DisableInheritableThreadFactoryWrapper implements DisableInheritableThreadFactory {
        private final ThreadFactory threadFactory; // 被包装的线程工厂

        @Override
        public Thread newThread(@NonNull Runnable r) {
            final Object backup = clear(); // 1、清理当前线程的上下文；
            try {
                return threadFactory.newThread(r); // 2、创建线程，不会继承到当前线程的上下文；
            } finally {
                restore(backup); // 3、恢复当前线程上下文。
            }
        }

        ...

    }
    ```
    将线程工厂包装为**禁止上下文传递的线程工厂**，防止由于子线程继承父线程上下文导致的内存泄漏或业务异常问题。

***