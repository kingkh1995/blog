# [首页](/blog/)

> version: **jdk17**

***

## public class Thread implements Runnable

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

## Callable

```java
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```
不是继承自Runnable，能返回结果或者抛出异常。

***

## public interface RunnableFuture<V> extends Runnable, Future<V>

RunnableFuture同时继承了Runnable和Future，用于包装任务，将自身作为Runable提交，之后再作为Future使用。

***

## public class FutureTask<V> implements RunnableFuture<V>

-   ```java
    private volatile int state;

    private Callable<V> callable;

    private volatile Thread runner;

    // 保存执行结果或执行时异常，异常会被包装为ExecutionException
    private Object outcome; // non-volatile, protected by state reads/writes
    ```

-   ```java
    public void run() {
        // 要求当前状态为NEW，并且使用VarHandle设置当前线程到runner属性成功。
        if (state != NEW || !RUNNER.compareAndSet(this, null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) { // CAS成功后，再次判断状态。
                V result;
                boolean ran;
                try {
                    result = c.call(); // 在执行过程中状态仍然为NEW
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    // NEW -> COMPLETING -> NORMAL
                    setException(ex);
                }
                // NEW -> COMPLETING -> EXCEPTIONAL
                if (ran)
                    set(result);
            }
        } finally {
            runner = null;
            int s = state;
            // 如果被cancel操作中断，yield直到操作完成。
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
    ```

-   ```java
    public boolean cancel(boolean mayInterruptIfRunning) {
        // mayInterruptIfRunning = false: NEW -> CANCELLED
        if (!(state == NEW && STATE.compareAndSet
                (this, NEW, mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    // mayInterruptIfRunning = true: NEW -> INTERRUPTING -> INTERRUPTED
                    STATE.setRelease(this, INTERRUPTED);
                }
            }
        } finally {
            finishCompletion();
        }
        return true;
    }
    ```
    **NEW状态只表示任务未执行完成**，mayInterruptIfRunning为true时，会执行运行线程的interrupt方法尝试打断。

***

## public abstract class AbstractExecutorService implements ExecutorService

ExecutorService的抽象实现，默认将提交给线程池的任务包装为FutureTask。

***

## public class ThreadPoolExecutor extends AbstractExecutorService

- 有界与无界：无界线程池的问题是可能会导致内存溢出，而有界线程池的问题是如果任务之间是有依赖性的，则可能造成线程池死锁。
- 最佳线程数：CPU密集型应用建议为N+1，N是为了减少线程的切换，+1是为了防止线程意外终止，而导致CPU资源被浪费；IO密集型应用，则需要结合实际场景进行设置，即要防止创建过多线程，也要保证应用的处理能力。

### 属性

#### 构造方法属性

```java
// 核心线程数
private volatile int corePoolSize;

// 最大线程数，必须大于corePoolSize。
private volatile int maximumPoolSize;

// 阻塞队列，无法被修改。
private final BlockingQueue<Runnable> workQueue;

// 空闲线程存活时间，单位纳秒。
private volatile long keepAliveTime;

// 线程工厂，默认Executors.defaultThreadFactory()。
private volatile ThreadFactory threadFactory;

// 拒绝策略，默认AbortPolicy。
private volatile RejectedExecutionHandler handler;
```

#### 其他属性

-   ```java
    // 高3位表示线程池状态，低29位表示worker数。
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

    // 前三位为0，后29位为1，通过&运算获取runState和workerCount值。
    private static final int COUNT_MASK = (1 << COUNT_BITS) - 1;

    // 使用runState和workerCount计算ctl值
    private static int ctlOf(int rs, int wc) { return rs | wc; }
    ```
    使用ctl记录状态和工作线程数，能保证原子性，状态判断只需要比较值，修改工作线程数则直接加减。
    - **RUNNING**：-1，线程池创建后处于RUNNING状态；
    - **SHUTDOWN**：0，调用线程池**shutdown方法**后处于SHUTDOWN状态，不再接受新的任务，执行中及阻塞队列内的任务会被执行完成；
    - **STOP**：1，调用线程池**shutdownNow方法**后处于STOP状态，不再接受新的任务，并会中断所有worker，丢弃阻塞队列内所有任务；
    - **TIDYING**：2，SHUTDOWN及STOP状态下，当worker数为0时线程池会转变为TIDYING状态，然后执行**terminated方法**；
    - **TERMINATED**：3，TIDYING状态下，执行完**terminated方法**后，线程池转变为TERMINATED状态，为终态。

-   ```java
    // 全局锁，主要用于维护workers集合。
    private final ReentrantLock mainLock = new ReentrantLock();

    // 用于awaitTermination
    private final Condition termination = mainLock.newCondition();

    // 记录线程池达到过的最大线程数，通过mainLock访问。
    private int largestPoolSize;

    // 线程池完成的任务总数，通过mainLock访问。
    private long completedTaskCount;
    ```

-   ```java
    private final HashSet<Worker> workers = new HashSet<>();
    ```
    Worker集合，使用mainLock保证其线程安全性。

-   ```java
    private volatile boolean allowCoreThreadTimeOut;
    ```
    是否允许空闲核心线程超时终止，**默认false**，只能通过allowCoreThreadTimeOut方法修改。
    > 适合只在特定时间段处理任务的线程池，可节约机器资源。

### interface RejectedExecutionHandler

```java
void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
```

-   ```java
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException(...);
    }
    ```
    AbortPolicy：默认策略

-   ```java
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            r.run(); // 提交任务的线程执行run方法
        }
    }
    ```
    CallerRunsPolicy: 提交任务的线程执行任务

-   ```java
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            e.getQueue().poll();
            e.execute(r); // 可能再次触发reject
        }
    }
    ```
    DiscardOldestPolicy: 丢弃队首任务

-   ```java
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    }
    ```
    DiscardPolicy: 丢弃任务

### private final class **Worker** extends AbstractQueuedSynchronizer implements Runnable

-   ```java
    // 工作线程
    final Thread thread;

    // 创建时提交的任务，可能为null。
    Runnable firstTask;

    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    public void run() {
        runWorker(this);
    }
    ```
    工作线程运行时执行ThreadPoolExecutor的runWorker方法。

-   ```java
    public void lock()        { acquire(1); }

    public boolean tryLock()  { return tryAcquire(1); }

    protected boolean tryAcquire(int unused) {
        // state = 0时才能成功
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }
    ```
    **只是使用了AQS状态管理的功能**，0表示未锁，-1（初始态）和1表示锁定；lock方法只在runWorker方法中调用，而runWorker方法是Worker的run方法调用，即只会是单线程场景。

### 实例方法

-   ```java
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry: // 标记外层循环
        for (int c = ctl.get();;) {
            // 允许的场景：1、RUNNING；2、SHUTDOWN且队列非空允许添加空任务worker。
            if (runStateAtLeast(c, SHUTDOWN)
                && (runStateAtLeast(c, STOP) || firstTask != null || workQueue.isEmpty()))
                return false;
            for (;;) { // 内层循环
                // 添加核心worker则worker数不能超过corePoolSize，非核心worker则不能超过maximumPoolSize。
                if (workerCountOf(c)
                    >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry; // CAS增加worker数成功则退出外层循环
                c = ctl.get();  // Re-read ctl
                if (runStateAtLeast(c, SHUTDOWN))
                    continue retry; // 非RUNNING则跳出内层循环进入外层循环再次判断是否允许添加worker
                // CAS失败且状态仍为RUNNING则重试内层循环
            }
        }
        // CAS增加woker数成功后创建woker
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock(); // 阻塞加锁
                try {
                    int c = ctl.get();
                    // 再次校验：1、RUNNING；2、SHUTDOWN且添加空任务worker
                    if (isRunning(c) ||
                        (runStateLessThan(c, STOP) && firstTask == null)) {
                        if (t.getState() != Thread.State.NEW) // 校验线程状态
                            throw new IllegalThreadStateException();
                        workers.add(w); // 添加到workers集合
                        workerAdded = true;
                        ...
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start(); // 启动worker，最终执行runWorker方法。
                    workerStarted = true;
                }
            }
        } finally {
            // 若启动失败抛出异常，则在finally中减少worker数并移除worker
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
    ```
    用于创建worker，若线程启动失败会抛出异常。

-   ```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // 将worker状态从-1修改为0，允许被打断。
        boolean completedAbruptly = true; // 表示worker是否因为异常退出
        try {
            // 无限循环，先执行firstTask，再通过getTask()获取任务执行。
            while (task != null || (task = getTask()) != null) {
                w.lock(); // 阻塞加锁
                // STOP状态以上保证设置worker的线程中断标识为true
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) 
                     && !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task); // 可以自定义的执行前钩子方法
                    try {
                        task.run();
                        afterExecute(task, null); // 可以自定义的执行后钩子方法
                    } catch (Throwable ex) {
                        afterExecute(task, ex); // 可以自定义的执行后钩子方法
                        throw ex;
                    }
                } finally {
                    task = null;
                    w.unlock();
                }
            }
            // 退出循环，getTask方法未返回任务，线程的run方法结束，线程退出。
            completedAbruptly = false;
        } finally {
            // 执行worker销毁工作
            processWorkerExit(w, completedAbruptly);
        }
    }

    // 获取任务，返回null则表示worker退出。
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?
        for (;;) {
            int c = ctl.get();
            // 所有worker退出的场景：1、SHUTDOWN且队列为空；2、STOP以上。
            if (runStateAtLeast(c, SHUTDOWN)
                && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
                decrementWorkerCount(); // worker正常结束worker数减一
                return null;
            }
            int wc = workerCountOf(c);
            // 当前worker允许退出条件：allowCoreThreadTimeOut为true或worker数大于corePoolSize
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            // 退出判断：1、worker数大于maximumPoolSize；2、允许退出且空闲超时。
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue; // CAS更新worker数失败则重试循环
            }
            // 不需要退出时从阻塞队列获取任务
            try {
                // 当前worker允许退出则使用带超时时间的poll方法，不允许退出则使用阻塞的take方法。
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r; // 获取到任务则return
                timedOut = true; // 未获取到任务则设置当前worker空闲超时
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }

    // 执行worker退出
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // 如异常退出worker数减一，正常退出在getTask方法中减一。
            decrementWorkerCount();
        ... // 移除worker，执行清理工作。
        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            // 正常退出判断是否需要添加worker
            if (!completedAbruptly) {
                // allowCoreThreadTimeOut为false时，worker数至少为corePoolSize。
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                // workQueue非空时至少要存在一个worker
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            // 异常退出时必然再次添加worker
            addWorker(null, false);
        }
    }
    ```
    worker启动后执行。

-   ```java
    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock(); // 阻塞加锁
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt(); // 打断
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne) // 为true时仅尝试一次
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }
    ```
    用于打断worker，tryTerminate方法中调用时onlyOne为true。

### **ExecutorService方法实现**

- execute(Runnable command)：
    1. 如果当前worker数小于corePoolSize，则调用addWorker方法创建核心worker，成功则直接返回。
    2. 线程池为RUNNING状态，才允许提交任务到阻塞队列，等待被worker获取；**如果入队成功则重新检查clt值，如线程池已不再是RUNNING状态则尝试回滚，从队列中移除任务并执行拒绝策略；如果此时线程池worker数为0（可能是回滚失败或状态未变不需要回滚），则使用addWorker方法添加一个空的非核心的worker，用以处理已入队的任务**。
    3. 任务入队失败，则尝试使用addWorker方法创建非核心worker，失败则执行拒绝策略。

- shutdown()：获取全局锁，修改状态为SHUTDOWN，使用interruptIdleWorkers方法打断所有worker，释放锁，最后执行一次tryTerminate方法。**interruptIdleWorkers方法对worker加锁成功才打断其线程，即会等待worker执行完当前任务，但runWorker方法会清除中断标识，意味着worker会继续从阻塞队列中获取任务并执行。**

- shutdownNow()：获取全局锁，修改状态为STOP，使用interruptWorkers方法打断所有worker，释放锁，最后执行一次tryTerminate方法，最后返回阻塞队列中剩余未被执行任务。**interruptWorkers方法不获取全局锁，也不对worker加锁，即会直接打断所有worker，然后在runWorker方法中所有worker都会终止。**

- awaitTermination(long timeout, TimeUnit unit)：获取全局锁，等待线程池状态变为TERMINATED后，被termination条件唤醒（来自tryTerminate方法），或达到超时时间。

#### **Worker extends AbstractQueuedSynchronizer implements Runnable**

工作线程，使用AQS独占模式，**0表示未锁，1表示锁定**。创建成功后先设置state为-1，表示此阶段不允许被打断，直到执行runWorker方法时，通过unlock方法将state设置为0。run方法实现为使用自身作为参数调用ThreadPoolExecutor的runWorker方法。

### Executors

- newCachedThreadPool()：new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS, new SynchronousQueue<Runnable>())，无界的线程池，适合执行大量的短时间任务。

- newFixedThreadPool(int nThreads)：corePoolSize = maximumPoolSize = nThreadsnew，使用无界的LinkedBlockingQueue，只会创建核心线程，适合处理负载较重的场景。

- newScheduledThreadPool(int corePoolSize)：使用DelayedWorkQueue，只会创建核心线程，用于执行延时任务。

- newWorkStealingPool(int parallelism)：返回ForkJoinPool，默认并发度为处理器个数，**适合计算密集型场景，不要用于执行阻塞型任务**。

***

## [ThreadLocal](/blog/adtl)

***

## **Fork/Join(TODO)**

## **ForkJoinPool**

继承自AbstractExecutorService，工作窃取线程池，即当某个线程执行完自己的任务后，从另一个线程的双端队列中窃取任务来执行。

### 构造方法

- int parallelism：并行度，默认值为CPU逻辑处理器个数（**公共池为N-1**），并不表示worker最大值；
- ForkJoinWorkerThreadFactory factory：worker创建工厂；
- boolean asyncMode：实际上是工作队列执行模式，true为FIFO（队列），false为LIFO（栈），默认false；
- corePoolSize：核心线程数，通常情况下等于并行度。
- maximumPoolSize：允许的最大线程数量，最大为32567（**公共池为256**）；
- maximumPoolSize：最小线程数，可以为0（**公共池为1**）。

### **属性**

- volatile int mode：记录并行度、运行状态和队列模式。

- WorkQueue[] queues：工作队列数组。

- volatile long ctl：分为四段记录，RC：没有入队的worker数量减去目标并行度；TC：总worker数量减去目标并行度；SS：队首等待的线程的版本计数和状态；ID：阻塞栈栈顶的poolIndex。

### **任务提交**

调用externalSubmit方法，如果线程为当前线程池的ForkJoinWorkerThread则直接push到非共享模式的队列中，否则使用externalPush，根据线程探针哈希值找到工作队列数组中未绑定ForkJoinWorkerThread的工作队列加入，最后都会调用signalWork方法通知ForkJoinWorkerThread执行，如果数量不足则创建ForkJoinWorkerThread。

### **调度过程**

工作队列数组大小为2的幂，初始化时为空数组，任务提交后在偶数槽添加一个没有绑定工作线程的工作队列，然后创建工作线程，将其工作队列放置在奇数槽上，然后扫描工作队列数组，找到一个可以窃取的工作队列，获取到ForkJoinTask后执行，而ForkJoinTask的fork方法又会将任务添加到当前工作线程的工作队列中，最终ForkJoinTask中的join方法又会导致当前工作线程阻塞等待结果，因为可能被其他工作线程窃取走了任务。

### **WorkQueue**

工作队列，为ForkJoinTask数组，由ForkJoinWorkerThread持有。

### **ForkJoinWorkerThread**

ForkJoinPool中的worker，持有ForkJoinPool和WorkQueue，run方法将WorkQueue注册到ForkJoinPool，再调用ForkJoinPool执行WorkQueue的任务。

## **ForkJoinTask**

抽象类，Fork/Join框架的核心，一般需要继承其抽象子类RecursiveAction和RecursiveTask，然后实现compute方法。核心是将任务拆分为小任务，然后小任务通过fork方法提交到ForkJoinPool异步执行，最后使用join方法等待小任务执行完成后合并计算结果。

- volatile int status：状态，0初始化，负数DONE，正数为未完成状态，高16位标识ABNORMAL和THROWN，低16位由子类自定义（用来表示并行度）。
- volatile Aux aux：等待完成或抛出了异常的节点。

- fork()：异步提交任务，如果当前线程为ForkJoinWorkerThread则提交到其workQueue，否则提交任务到公共ForkJoinPool的externalQueue。

- join()：非完成状态时，等待所有任务完成，不会抛出受检查异常。

- get()：等待所有任务完成，会抛出受检查异常。

- invoke()：执行所有任务，同步等待执行完成。

- **innt awaitDone(ForkJoinPool pool, boolean ran, boolean interruptible, boolean timed, long nanos)**：核心方法，用于join、get、invoke，帮助and或or的waits完成。

***
