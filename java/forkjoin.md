# [首页](/blog/)

> version: **jdk17**

> **Fork/Join(TODO)**

***

## **ForkJoinPool**

继承自AbstractExecutorService，工作窃取线程池，即当某个线程执行完自己的任务后，从另一个线程的双端队列中窃取任务来执行。

### 构造方法

- int parallelism：并行度，默认值为CPU逻辑处理器个数（**公共池为N-1，因为还存在一个主线程，为了减少线程的切换。**），并不表示worker最大值；
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