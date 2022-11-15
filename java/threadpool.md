# [首页](/blog/)

> version: **jdk17**

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
- 任务取消及移除：**任务的取消是通过FutureTask保证**，任务提交后即不会从线程池移除（ScheduledThreadPoolExecutor可设置removeOnCancel），需要调用remove方法移除任务或purge方法移除所有被取消的任务。

### 属性

#### 构造方法属性

```java
// 核心线程数，默认添加任务时才创建，可通过prestartCoreThread、ensurePrestart及prestartAllCoreThreads提前创建。
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
                w.lock(); // 获取到任务后阻塞加锁
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
                    task = null; // benefit for GC
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
    worker启动后执行，只有getTask方法返回null时worker才会退出，**每次获取到任务后才锁定worker**。

-   ```java
    // tryTerminate方法中调用时onlyOne为true
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
    用于打断**空闲的**worker，因为需要tryLock成功；空闲表示worker在获取任务中，此时worker会释放锁；打断只是设置了worker的中断标识，worker仍然会继续执行任务。

### **ExecutorService方法实现**

-   ```java
    public void execute(Runnable command) {
        ...
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true)) // 添加核心worker
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get(); // 入队成功后recheck状态
            if (!isRunning(recheck) && remove(command)) // 非RUNNING则移除已入队任务
                reject(command); // 移除成功则拒绝任务
            else if (workerCountOf(recheck) == 0) // 无须移除任务或任务移除失败场景且worker数为0
                addWorker(null, false); // 添加空任务worker处理入队的任务
        }
        else if (!addWorker(command, false))  // 添加非核心worker
            reject(command); // 添加worker失败则拒绝任务
    }
    ```
    执行任务流程：
    1. 运行线程数小于corePoolSize则创建核心线程；
    2. 创建核心线程失败则任务入队；
    3. 任务入队失败则创建非核心线程，创建失败则拒绝任务。

-   ```java
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock(); // 阻塞加锁
        try {
            checkShutdownAccess();
            advanceRunState(SHUTDOWN); // 自旋直到CAS成功更新状态为SHUTDOWN
            interruptIdleWorkers(); // 打断所有空闲的worker，需要tryLock成功。
            onShutdown(); // 执行自定义钩子方法
        } finally {
            mainLock.unlock();
        }
        tryTerminate(); // 尝试终止
    }
    ```
    shutdown不再允许接受任务，但未清理队列，仅打断空闲的线程，执行中及队列内的任务仍将执行完成；允许添加worker，以执行队列内剩余的任务。

-   ```java
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock(); // 阻塞加锁
        try {
            checkShutdownAccess();
            advanceRunState(STOP); // 自旋直到CAS成功更新状态为STOP
            interruptWorkers(); // 打断所有worker，不需要tryLock。
            tasks = drainQueue(); // 移除队列内所有任务
        } finally {
            mainLock.unlock();
        }
        tryTerminate(); // 尝试终止
        return tasks;
    }
    ```
    shutdownNow也不再允许接受任务，并清理队列，打断所有线程。

***

## Executors

-   ```java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
    ```
    FixedThreadPool: 固定线程池，使用无界的LinkedBlockingQueue，适用于**负载较重场景**。

-   ```java
    public static ExecutorService newWorkStealingPool(int parallelism) {
        return new ForkJoinPool
            (parallelism, // 默认Runtime.getRuntime().availableProcessors()
             ForkJoinPool.defaultForkJoinWorkerThreadFactory,
             null, true); // 执行模式为FIFO
    }
    ```
    WorkStealingPool: 使用ForkJoinPool，适合处理**计算密集型任务**，能充分利用CPU性能，一定程度上可替代FixedThreadPool。

-   ```java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>()));
    }
    ```
    SingleThreadExecutor: 等同于线程数为1的FixedThreadPool，适用于**串行执行场景**。

-   ```java
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE, // 全部为非核心线程
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
    ```
    CachedThreadPool：无界线程池，使用同步阻塞队列（不存储任务），适合处理**大量短时间任务**。

    ```java
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
    ```
    ScheduledThreadPool: 固定长度线程池，使用DelayedWorkQueue，用于处理定时及周期性任务。

### ScheduledThreadPoolExecutor

#### private class ScheduledFutureTask\<V\> extends FutureTask\<V\> implements RunnableScheduledFuture<V>

ScheduledThreadPoolExecutor使用的FutureTask，使用heapIndex记录了task在DelayedWorkQueue中的索引，**将查找效率从O(logn)提高到O(1)**。

#### class DelayedWorkQueue extends AbstractQueue\<Runnable\> implements BlockingQueue\<Runnable\>

ScheduledThreadPoolExecutor使用的BlockingQueue，**为基于延时时间的优先队列（实现类似DelayQueue）**，存储任务使用RunnableScheduledFuture<?>[]（非PriorityQueue），故查找效率为O(1)。

***