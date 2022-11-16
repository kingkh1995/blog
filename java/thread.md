# [首页](/blog/)

> version: **jdk17**

***

## public class **Thread** implements Runnable

### **线程状态**

- 操作系统主要线程状态：
    - ready：就绪状态，线程正在等待使用CPU，经调度程序调用之后可进入running状态；
    - running：执行状态，线程正在使用CPU；
    - waiting：等待状态，线程经过等待事件的调用或者正在等待其他资源（如I/O）。

- **JAVA线程状态**：
    - NEW：线程对象创建成功，还未调用start方法，即还未关联到操作系统；

    - RUNNABLE：线程在jvm中运行，**包含了操作系统的ready和running两个状态**；
  
    - BLOCKED：阻塞状态，线程正在等待锁的释放以进入同步区；
        > **线程在RUNNABLE状态时，如果无法获取到锁则转变为BLOCKED状态，竞争到锁之后再转变回RUNNABLE状态。**

    - WAITING：等待状态，线程需要被唤醒（或中断）才能进入RUNNABLE状态；
        > Object的wait()、Thread的join()和LockSupport.park()会使线程进入此状态。

    - TIMED_WAITING：超时等待状态，与WAITING状态不同的是线程在等待一个具体的时间后会被自动唤醒；
        > Thread.sleep()以及带参数的wait、join、park会使线程进入此状态。

    - TERMINATED：终止状态，线程执行完毕，由RUNNABLE转变为TERMINATED。

### 属性

-   ```java
    private Runnable target;

    @Override
    public void run() {
        if (target != null) {
            target.run();
        }
    }
    ```

-   ```java
    // 0 for NEW, write by VM.
    private volatile int threadStatus;
    ```

-   ```java
    private boolean daemon = false;
    ```
    是否为守护线程，所有守护线程会在JVM正常退出时被直接抛弃，**也不会执行finally代码**，故使用守护线程执行I/O操作是危险的行为。

-   ```java
    private ThreadGroup group;
    ```
    每个线程必然存在属于一个线程组，默认为其父线程所属的线程组；在调用start方法后线程才会被加入线程组，同时结束后会被移除；ThreadGroup是一个标准的向下引用的树状结构，能防止"上级"线程被"下级"线程引用而无法有效地被GC回收。

-   ```java
    private int priority;

    public final void setPriority(int newPriority) {
        ThreadGroup g;
        checkAccess();
        // 数值范围为[1, 10]
        if (newPriority > MAX_PRIORITY || newPriority < MIN_PRIORITY) {
            throw new IllegalArgumentException();
        }
        // 线程优先级不能大于其所在线程组的最大优先级
        if((g = getThreadGroup()) != null) {
            if (newPriority > g.getMaxPriority()) {
                newPriority = g.getMaxPriority();
            }
            // 通过本地方法修改线程优先级
            setPriority0(priority = newPriority); 
        }
    }
    ```
    线程优先级，可以指定为1~10，默认为5，但最终还是由操作系统决定，只是高优先级的线程会有更高的几率得到执行。

-   ```java
    private final long tid;

    private static long threadSeqNumber;

    // 用于生成tid
    private static synchronized long nextThreadID() {
        return ++threadSeqNumber;
    }
    ```

-   ```java
    private ClassLoader contextClassLoader;
    ```
    上下文类加载器，用于ServiceLoader。

-   ```java
    ThreadLocal.ThreadLocalMap threadLocals = null;

    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
    ```
    ThreadLocal及InheritableThreadLocal支持。
    
-   ```java
    // 中断标识
    private volatile boolean interrupted;

    // 不会中断线程，也不会让线程抛出InterruptedException，只是设置线程的中断标识为true。
    public void interrupt() {
        ...
        interrupted = true;
        interrupt0();
    }

    // 获取线程的中断标识
    public boolean isInterrupted() {
        return interrupted;
    }

    // 静态方法，获取当前线程的中断标识，并清除其中断标识，是唯一能清除中断标识的方法。
    public static boolean interrupted() {
        Thread t = currentThread();
        boolean interrupted = t.interrupted;
        if (interrupted) {
            t.interrupted = false;
            clearInterruptEvent();
        }
        return interrupted;
    }
    ```
    如果某个方法抛出InterruptedException，表示它可以被中断，用户程序必须对其进行捕获或继续向上抛出；表示线程执行该方法时如果检测到中断标识为true，会立即抛出InterruptedException，**并会清除中断标识**，交由用户程序处理；捕获InterruptedException后，最好的做法是执行当前线程的interrupt方法以恢复中断标识，交由调用方处理；**不要使用stop方法终止线程，而是应该主动检查中断标识以自行终止线程执行**。

### 静态方法

-   ```java
    public static native void yield();
    ```
    静态本地方法，表示当前线程愿意让出对当前CPU的占用，重新进入竞争CPU的状态。
    > **线程状态仍然是RUNNABLE，但操作系统线程状态从running变为ready。**

-   ```java
    public static native void sleep(long millis) throws InterruptedException;
    ```
    静态本地方法，让当前线程停止执行并让出对CPU的占用，进入TIMED_WAITING状态，不会释放同步锁。
    > 注意：与Object的wait方法不同，sleep(0)并不是一直睡眠。

-   ```java
    // since 9
    public static void onSpinWait() {}

    // 用于提高忙等待效率
    while (!match) {
        Thread.onSpinWait();
    }
    ```

### 实例方法

-   ```java
    public synchronized void start() {
        // 必须是NEW状态，即只能执行一次。
        if (threadStatus != 0)
            throw new IllegalThreadStateException();
        // 添加到线程组内
        group.add(this);
        boolean started = false;
        try {
            // 本地方法，关联到操作系统，会修改threadStatus。
            start0();
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {}
        }
    }
    ```
    启动线程，线程状态转变为RUNNABLE，获取到CPU资源后，JVM会执行线程的run方法。

-   ```java
    public final native boolean isAlive();
    ```
    判断线程是否是存活的，即已被启动且还没死亡。

-   ```java
    public final void join() throws InterruptedException {
        join(0);
    }

    public final synchronized void join(final long millis) throws InterruptedException {
        if (millis > 0) { // 大于0则等待指定时间
            if (isAlive()) {
                final long startTime = System.nanoTime();
                long delay = millis;
                do {
                    wait(delay);
                } while (isAlive() && (delay = millis -
                        TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime)) > 0);
            }
        } else if (millis == 0) { // 等于0则表示一直等待
            while (isAlive()) {
                wait(0);
            }
        } else {
            throw new IllegalArgumentException("timeout value is negative");
        }
    }
    ```
    在当前线程中调用另一个线程的join方法后，当前线程会进入等待状态，直到另一个线程死亡才会退出等待；使用wait实现，故必须是同步方法，会一直循环调用，直到isAlive方法返回false或达到指定时间。

***

## **ThreadLocal**

线程局部变量，并不实际存储值，而是使用ThreadLocalMap，**建议使用static修饰**。

### 哈希值

```java
private final int threadLocalHashCode = nextHashCode();
private static AtomicInteger nextHashCode = new AtomicInteger();
private static final int HASH_INCREMENT = 0x61c88647;
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```
不使用Object默认哈希值，而是使用**魔数0x61c88647**作为增长步幅生成固定的哈希值，能均匀的分布在2的N次方的数组里，原理是斐波那契散列法。

### 方法

-   ```java
    // Thread类
    ThreadLocal.ThreadLocalMap threadLocals = null;

    // 从线程中获取ThreadLocalMap
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    // 为线程初始化ThreadLocalMap并设置ThreadLocal的初始值
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
    ```
    值实际上保存于Thread类的ThreadLocalMap类型的threadLocals属性中。

-   ```java
    public void set(T value) {
        Thread t = Thread.currentThread(); // 获取当前线程
        ThreadLocalMap map = getMap(t); // 获取当前线程的ThreadLocalMap
        if (map != null) {
            map.set(this, value); // map存在则设置值
        } else {
            createMap(t, value); // map不存在则创建map并设置值
        }
    }
    
    public T get() {
        Thread t = Thread.currentThread(); // 获取当前线程
        ThreadLocalMap map = getMap(t); // 获取当前线程的ThreadLocalMap
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this); // 使用ThreadLocal作为键查找ThreadLocalMap中对应的值
            if (e != null) {
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue(); // 不存在则设置初始化值
    }

    public void remove() {
        ThreadLocalMap m = getMap(Thread.currentThread());
        if (m != null) { 
            m.remove(this); // 从map中移除ThreadLocal作为键的Entry
        }
    }
    ```
    set、get、remove都是以ThreadLocal对象作为键操作Thread对象的持有的ThreadLocalMap对象。

***

## **ThreadLocalMap**

### 结构

```java
static class Entry extends WeakReference<ThreadLocal<?>> {

    Object value; // ThreadLocal关联的值

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}

private Entry[] table; // Entry数组，初始容量16，且要求为2的次方。

private int size = 0; // Entry数量

private int threshold; // rehash阈值，为2/3容量，而resize阈值为1/2容量。
```
ThreadLocal真正的核心，**为线性探测法实现的Map**，Entry弱引用了ThreadLocal，但value为强引用，当ThreadLocal对象被回收后，视为Entry过期，需要对Entry进行清理。

### 方法
    
-   ```java
    // 线性探测法删除键
    // 1、删除当前位置的键；
    // 2、遍历处理所属键簇剩余所有键（until null），过期键直接删除，未过期键重新插入（必然不会在原位置之后）；
    // 3、返回清理完成后当前键簇之后第一个空槽位的索引。
    private int expungeStaleEntry(int staleSlot) { ... }

    // 线性探测法删除全部过期键，遍历数组，对每个位置的过期键调用expungeStaleEntry方法。
    private void expungeStaleEntries() { ... }
    ```
    线性探测法删除键后，需要重新插入所属键簇剩余所有键。

-   ```java
    private Entry getEntry(ThreadLocal<?> key) {
        int i = key.threadLocalHashCode & (table.length - 1); // 使用&操作快速确定槽位
        Entry e = table[i];
        if (e != null && e.refersTo(key)) // 判断当前位置找到
            return e;
        else
            return getEntryAfterMiss(key, i, e); // 线性探测查找
    }

    private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
        Entry[] tab = table;
        int len = tab.length;
        while (e != null) { // 遍历当前键簇查找
            if (e.refersTo(key))
                return e;
            if (e.refersTo(null)) // 清理过期键
                expungeStaleEntry(i); // i不加一，因为清理完成后，当前位置会更新。
            else
                i = nextIndex(i, len);
            e = tab[i];
        }
        return null;
    }

    // 确定到槽位，线性探测，找到则使用expungeStaleEntry方法移除键。
    private void remove(ThreadLocal<?> key) { ... }
    ```
    get操作和remove操作会执行少量过期键清理工作。

-   ```java
    private void set(ThreadLocal<?> key, Object value) {
        Entry[] tab = table;
        int len = tab.length;
        int i = key.threadLocalHashCode & (len-1); // 使用&操作快速确定槽位
        for (Entry e = tab[i];
                e != null;
                e = tab[i = nextIndex(i, len)]) { // 遍历当前键簇查找
            if (e.refersTo(key)) { // 查找到则直接替换值
                e.value = value;
                return;
            }
            if (e.refersTo(null)) { // 查找到过期键则替换Entry
                replaceStaleEntry(key, value, i);
                return;
            }
        }
        tab[i] = new Entry(key, value); // 当前键簇找不到则插入到键簇后的第一个位置
        int sz = ++size;
        if (!cleanSomeSlots(i, sz) && sz >= threshold) // 清理工作，超过阈值则rehash。
            rehash();
    }

    // 替换过期键并清理整个键簇所有的过期键
    private void replaceStaleEntry(ThreadLocal<?> key, Object value, int staleSlot) {
        Entry[] tab = table;
        int len = tab.length;
        Entry e;
        int slotToExpunge = staleSlot;
        // 1、向前找到当前键簇中第一个过期键的位置，记录为过期键清除的起点slotToExpunge；
        for (int i = prevIndex(staleSlot, len);
                (e = tab[i]) != null;
                i = prevIndex(i, len))
            if (e.refersTo(null))
                slotToExpunge = i;
        // 2、从要替换的位置继续往后查找键；
        for (int i = nextIndex(staleSlot, len);
                (e = tab[i]) != null;
                i = nextIndex(i, len)) {
            if (e.refersTo(key)) { // 如果能找到键，则替换到目标位置。
                e.value = value;
                tab[i] = tab[staleSlot];
                tab[staleSlot] = e;
                if (slotToExpunge == staleSlot) // 表示前半段都不存在过期键，只需从i位置开始清理即可。
                    slotToExpunge = i; 
                // 3、执行一次expungeStaleEntry，目的是清理整个键簇所有的过期键；
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                return;
            }
            // 如果slotToExpunge == staleSlot，表示前半段不存在过期键，将后半段找到的第一个过期键的位置设置为slotToExpunge。
            if (e.refersTo(null) && slotToExpunge == staleSlot)
                slotToExpunge = i;
        }
        // 整个键簇都找不到键则直接设置到目标位置
        tab[staleSlot].value = null;
        tab[staleSlot] = new Entry(key, value);
        if (slotToExpunge != staleSlot) // 表示整个键簇还存在其他过期
            // 3、执行一次expungeStaleEntry，目的是清理整个键簇所有的过期键；
            // 4、再执行一次cleanSomeSlots。
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
    }

    // 从i开始尝试使用expungeStaleEntry方法清理几个槽位，时间复杂度为O(n)。
    private boolean cleanSomeSlots(int i, int n) { ... }

    private void rehash() {
        expungeStaleEntries(); // 清理所有过期键
        // 清理完成后仍超过1/2容量则扩容为两倍大小
        if (size >= threshold - threshold / 4)
            resize();
    }
    ```
    **set操作则会执行更多的清理工作，同时可能触发resize操作。**

***

## class **InheritableThreadLocal**\<T\> extends ThreadLocal\<T\>

可继承的ThreadLocal，基于ThreadLocal，解决父子线程之间上下文传递的问题。

-   ```java
    // Thread类
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;

    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
    ```
    InheritableThreadLocal的值，保存于Thread类的ThreadLocalMap类型的**inheritableThreadLocals**属性中。

-   ```java
    // Thread构造方法，inheritThreadLocals默认为true。
    private Thread(ThreadGroup g, Runnable target, String name,
                   long stackSize, AccessControlContext acc,
                   boolean inheritThreadLocals) { // 默认为true
        ...
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        ...
    }

    static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
        return new ThreadLocalMap(parentMap);
    }

    private ThreadLocalMap(ThreadLocalMap parentMap) {
        ...
        table = new Entry[len];
        for (Entry e : parentTable) {
            if (e != null) {
                ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                if (key != null) {
                    // 使用childValue方法复制值，默认为浅拷贝。
                    Object value = key.childValue(e.value);
                    Entry c = new Entry(key, value);
                    ...
                }
            }
        }
    }

    protected T childValue(T parentValue) {
        return parentValue;
    }
    ```
    创建子线程时将父线程的inheritableThreadLocals复制到子线程中，**线程池中不会生效**，因为线程池中的线程并不都是提交任务时创建的，故无法继承来自父线程的上下文。

***
