# [首页](/blog/)

> version: **jdk17**

***

## public class Thread implements Runnable

### **线程状态**

- 操作系统的主要线程状态：
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

### 属性及方法

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

### **FutureTask\<V> implements RunnableFuture\<V>**

- RunnableFuture同时继承了Runnable和Future。用来包装任务后，将自身作为Runable提交，之后当作future使用。

- FutureTask在并发场景下能保证任务只被执行一次（基于CAS）。初始状态为NEW，运行状态只会在set、setException、cancel方法中终止，COMPLETING、INTERRUPTING是任务完成后的瞬时状态。

- 可能的状态转变：
  - NEW -> COMPLETING -> NORMAL
  - NEW -> COMPLETING -> EXCEPTIONAL
  - NEW -> CANCELLED
  - NEW -> INTERRUPTING -> INTERRUPTED

***

## ThreadPool

**有界线程池的问题是，如果任务之间是有依赖性的，则可能造成线程池死锁。**

### **AbstractExecutorService**

ExecutorService默认抽象实现。sumbit方法的实现为使用FutureTask包装任务并提交给execute方法后返回。

### **ThreadPoolExecutor extends AbstractExecutorService**

#### 构造函数

- int corePoolSize：线程池中允许的核心线程数最大值。
    - 核心线程默认一旦创建则不会被销毁，可以通过allowCoreThreadTimeOut方法修改。
    - 提交任务时才会创建核心线程，可以通过prestartCoreThread方法（一个）或prestartAllCoreThreads方法（所有）提前创建。
    - IO密集型配置线程数经验值是2N，CPU密集型配置线程数经验值是N + 1。

- int maximumPoolSize：线程池中允许的线程总数最大值，等于核心线程数 + 非核心线程数。

- long keepAliveTime & TimeUnit unit：非核心线程闲置超时时间及时间单位。

- BlockingQueue workQueue：阻塞队列，**线程池核心线程数量达到最大值后，任务会被提交到阻塞队列，等待被空闲的线程执行。**

- ThreadFactory threadFactory：创建线程的工厂，默认使用DefaultThreadFactory。

- RejectedExecutionHandler handler：拒绝策略。线程池的总数已达到阈值后，对任务采取的策略。默认提供了AbortPolicy、DiscardPolicy、DiscardOldestPolicy和CallerRunsPolicy。
    - **AbortPolicy：默认策略，丢弃任务并抛出RejectedExecutionException异常。**
    - DiscardPolicy：什么都不做，即直接丢弃任务。
    - DiscardOldestPolicy：阻塞队列poll出一个任务，并再次提交任务到线程池，即丢弃最旧任务。
    - CallerRunsPolicy：如线程池未关闭，则直接执行任务的run方法，即由提交任务的线程执行任务。

#### 属性

- **AtomicInteger ctl：使用高3位表示线程池状态，低29位表示Worker总数。**
    - RUNNING：-1，线程池创建后处于RUNNING状态；
    - SHUTDOWN：0，调用shutdown()方法后线程池处于SHUTDOWN状态，不能接受新的任务；
    - STOP：1，调用shutdownNow()方法后线程池处于STOP状态，不能接受新的任务，中断所有线程，阻塞队列所有任务全部丢弃；
    - TIDYING：2，Worker总数为0时，线程池会变为TIDYING状态，然后会执行terminated()方法；
    - TERMINATED：3，TIDYING状态下，执行完terminated()方法后，线程池被设置为TERMINATED状态。

- ReentrantLock mainLock：全局可重入锁。

- Condition termination：使用mainLock创建的Condition，用于支持awaitTermination方法。

- HashSet<Worker> workers：使用HashSet保存工作线程，只有获取到mainLock后才允许操作。

#### 方法

- addWorker(Runnable firstTask, boolean core)：
    1. RUNNING状态允许添加worker，或**SHUTDOWN状态且队列非空时（对应到execute方法回滚入队任务失败且worker已被完全销毁的场景）允许添加空任务worker**；
    2. 自旋尝试CAS增加worker数量，如果是创建核心worker则当前worker总数不能超过corePoolSize，如果是非核心worker则不能超过maximumPoolSize；
    3. 使用firstTask创建一个Worker对象，获取到全局锁后执行操作，**再次检查clt值，要求是RUNNING状态或SHUTDOWN状态且firstTask为空**，继续判断worker持有的线程对象状态是否是初始化状态，否则抛出IllegalThreadStateException；**将worker添加到HashSet后释放锁**；
    4. 调用worker持有的线程对象的start方法，**因为是使用worker作为Runable创建的线程对象，即线程执行时JVM会调用worker的run方法**；
    5. 如果添加worker失败，则执行addWorkerFailed方法，获取到全局锁后执行操作，移除worker并回滚clt值，**最后调用一次tryTerminate方法**。

- getTask()：从阻塞队列中获取任务，自旋操作直到成功，返回null则表示要销毁一个工作线程。
    1. 首先检查clt值，只允许RUNNING状态或SHUTDOWN状态且队列非空，否则worker数量减一并返回null；
    2. 如果worker数量超过了阈值或存在了空闲线程，则使worker数量减一并返回null。
    3. **如果存在非核心线程或允许核心线程过期，则使用带超时时间的poll方法从阻塞队列中获取任务，未获取到则表示线程空闲，进入下一个循环的判断，减少worker数量并退出方法；否则使用take方法阻塞获取任务。**

- runWorker(Worker w)：
    1. 将worker的firstTask属性置为null，并调用一次worker的unlock方法，表示worker可以被打断；
    2. 使用无限循环执行任务，执行firstTask或从workQueue中获取任务，当getTask方法返回null时退出循环，则worker持有的线程退出即被销毁，最终在finally中执行processWorkerExit方法。
    3. **执行任务时，需要先对worker加锁，如果线程池状态大于等于STOP，要保证中断线程，否则清除中断标识，然后执行任务，释放worker的锁。**

- processWorkerExit(processWorkerExit(Worker w, boolean completedAbruptly)：completedAbruptly表示是任务执行过程中抛出异常导致worker终止，则需要减少worker数量；获取全局锁，从HashSet中移除worker对象；调用一次tryTerminate方法；检查clt值，如线程池状态小于STOP，则需要使用addWorker方法重新添加一个空的非核心worker，如果是超时销毁，在worker总数小于corePoolSize时才添加worker。**目的是为了提高并发，因为addWorker也需要获取全局锁**。

- tryTerminate()：**状态为SHUTDOWN同时池和队列都为空，或者状态为STOP同时池为空**时，尝试转换线程池状态为TERMINATED。自旋操作，首先如果池非空，则使用interruptIdleWorkers方法打断一个worker，退出方法；池已经为空了，则获取到全局锁，将状态设置为TIDYING，执行terminated方法（由子类拓展），最后将状态设置为TERMINATED，唤醒在termination条件上等待的所有线程。

#### **ExecutorService接口方法**

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
