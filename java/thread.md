# [首页](/blog/)

> version: **jdk17**

***

## Thread implements Runnable

### 静态方法

- nextThreadID()：静态同步方法，用于生成线程ID，**基于静态long类型属性threadSeqNumber自增实现**，并没有使用原子类型。

- currentThread()：静态本地方法，获取当前线程对象的引用。

- yield()：静态本地方法，表示当前线程愿意让出对当前CPU的占用，重新进入竞争CPU的状态。

- sleep()：静态本地方法，抛出InterruptedException异常，让当前线程停止执行并让出对CPU的占用，当指定的时间到了之后会自动恢复运行状态，睡眠期间不会释放对象锁资源。

- interrupted()：获取当前线程的中断标识后清除标识。

### 实例方法

- start()：同步方法，用于启动线程，启动后虚拟机会调用run方法，再次调用会抛出IllegalThreadStateException异常。

- join()：同步方法，当前线程调用另一个线程的join方法，当前线程会一直等待直到另一个线程死亡才继续，要求另一个线程必须是启动状态。
    - ***内部通过Object的wait方法实现，使当前线程等待直到另一个线程退出后当前线程才会被唤醒。***

- interrupt()：并不是打断线程，只是将中断标识设置为true，只有当线程处于等待状态时，线程才会检查中断标识，如果为true会抛出InterruptedException异常。

### 线程状态

- **操作系统主要线程状态**：
    - ready：就绪状态，线程正在等待使用CPU，经调度程序调用之后可进入running状态；
    - running：执行状态，线程正在使用CPU；
    - waiting：等待状态，线程经过等待事件的调用或者正在等待其他资源（如I/O）。

- **java线程状态**：
    - NEW：线程创建成功，还未调用start方法；
    - RUNNABLE：线程在jvm中运行，**包含了操作系统线程的ready和running两个状态**；
    - BLOCKED：阻塞状态，正在等待锁的释放；
    - WAITING：等待状态，需要被唤醒才能进入RUNNABLE状态；
    - TIMED_WAITING：超时等待状态，线程等待一个具体的时间后会被自动唤醒；
    - TERMINATED：终止状态，线程已执行完毕。

### ThreadGroup

每个Thread必然存在于一个ThreadGroup中，ThreadGroup是一个标准的向下引用的树状结构，是防止"上级"线程被"下级"线程引用而无法有效地被GC回收，可以指定1~10的优先级，java中默认为5，线程的优先级最终还是由操作系统决定，高优先级的线程会有更高的几率得到执行，线程优先级不能大于其所在线程组的最大优先级。

***

## FutureTask<V> implements RunnableFuture<V>
RunnableFuture用于在只能使用Runnable的场景下提供Callable的功能，作为Runnable提交后，再将自身作为Future使用。

- 可能的状态转变：
  - NEW -> COMPLETING -> NORMAL
  - NEW -> COMPLETING -> EXCEPTIONAL
  - NEW -> CANCELLED
  - NEW -> INTERRUPTING -> INTERRUPTED

***

## ThreadPool

***

## [ThreadLocal](/blog/adtl)

***
