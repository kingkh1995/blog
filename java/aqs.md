# [首页](/blog/)

> version: **jdk17**

***

## **AQS**

抽象队列同步器，是一个用于构建同步器的框架，底层使用了一个先进先出的同步队列和一个int类型的状态。为模板方法设计模式，提供了一系列方法用于获取和释放同步状态，子类需要实现tryAcquire、tryAcquireShared、tryRelease、tryReleaseShared、isHeldExclusively方法。

### **state**

volatile的，资源的标识，使用该变量的值来表示同步器的状态，定义了compareAndSetState方法对其进行CAS更新。


### **Node**

同步队列，为先进先出的双端队列，分为独享模式和共享模式，独享模式只允许一个线程获取到资源，而共享模式允许多个。

- JDK14之前：
    - waitStatus：节点等待状态标识，0（初始状态）、CANCELLED（1表示该节点已被取消）、SIGNAL（-1表示其后继节点需要被唤醒）、CONDITION（-2表示该节点位于条件等待队列中）、PROPAGATE（-3表示共享模式下无条件向后传播releaseShared状态，是为了解决共享锁并发释放引起线程挂起的问题）
    - nextWaiter：条件等待队列，如果是共享模式则是一个常量值。

- **JDK14**：变为抽象类，子类为ExclusiveNode、SharedNode、ConditionNode（独有nextWaiter属性），取消waitStatus改为status。
    - status：节点状态，WAITING（**等待状态才可以被挂起或唤醒**）、COND（位于条件等待队列中）、CANCELLED（负数）

### 获取和释放资源

#### JDK14之前

- acquire：首先尝试使用tryAcquire方法获取资源，失败则使用addWaiter方法往同步队列中加入一个独享节点后，再使用acquireQueued方法自旋处理队列直到获取资源成功。
    - acquireQueued：自旋处理直到加入节点的前置节点为头节点且再次调用tryAcquire方法获取资源成功，**成功后将当前节点设置为头节点（同步队列的头节点是虚节点或已出队节点）**，否则调用shouldParkAfterFailedAcquire判断是否需要阻塞当前节点（使用LockSupport的park方法挂起线程）。
    - **shouldParkAfterFailedAcquire**：判断前置节点的waitStatus，**只有为SIGNAL才阻塞当前节点（表示当前节点只需等待前置节点释放资源后将其唤醒然后再次尝试获取资源即可）**；如果大于0（取消状态）则从后往前移除所有连续的取消节点后；其他情况（初始状态或PROPAGATE）则更新前置节点waitStatus为SIGNAL。

- acquireShared：尝试使用tryAcquireShared获取资源，如果返回值小于0调用doAcquireShared阻塞获取资源。
    - doAcquireShared：首先往同步队列中加入一个共享节点，之后自旋处理队列直到获取资源成功。处理逻辑与acquireQueued一致，当加入节点的前置节点为头节点时调用tryAcquireShared获取资源，成功后将当前节点设置为头节点，**在剩余资源数大于0、头节点状态为空、头节点非取消状态情景下，如果加入节点不存在后继节点、或后继节点仍然为共享节点**， 则需要调用doReleaseShared方法唤醒同步队列中节点以获取资源。

- release：使用tryRelease方法释放资源，**释放成功后不移除同步队列头节点**，头节点非空且非初始状态时则使用unparkSuccessor方法处理同步队列。
    - **unparkSuccessor**：首先如果头节点waitStatus小于0则先尝试将其设置为0（**并发释放资源时阻止其他线程进入该方法**），然后唤醒头节点的第一个非取消状态的后续节点（节点竞争资源失败且处于阻塞状态，使用LockSupport的unpark方法唤醒线程）。如果头节点的后继节点不存在或为取消状态，则**从同步队列尾端往前查找到最后一个非取消状态的节点（出于线程安全考虑）**。

- releaseShared：使用tryReleaseShared方法释放资源，释放成功后不移除同步队列头节点，调用doReleaseShared方法唤醒后续节点。
    - **doReleaseShared**：**循环尝试对同步队列头节点使用unparkSuccessor方法直到头节点不再变化**，即同步队列中的节点不再能获取到资源，因为如果等待的节点获取到资源则头节点会改变。

#### **JDK14**

- acquire & acquireShared：首先尝试调用tryAcquire或tryAcquireShared获取资源，失败则调用核心的acquire方法，参数node为null。

- **final int acquire(Node node, int arg, boolean shared, boolean interruptible, boolean timed, long time)**：获取资源核心方法，自旋操作直到获取到资源或等待超时。
    - spins、postSpins：**记录自旋次数，目的是尽可能的使用自旋而不是将线程挂起**。每次线程被挂起时postSpins都会增加，并将spins重置为postSpins，每自旋一次spins减一直到为0时则再次挂起线程。
    - 首先判断当前节点的前置节点是否为头节点，否的场景下，如果前置节点是取消状态（status小于0）则调用cleanQueue方法清理队列；
    - 继续处理，如果前置节点已经为头节点或者前置节点为null（节点还没加入队列），则调用tryAcquire或tryAcquireShared获取资源，如获取资源成功且当前节点已经入队，则当前节点设为头节点，**共享模式时还需要对当前节点调用signalNextIfShared方法（与signalNext方法相同但是只唤醒共享节点）**；
    - 继续处理，如未则初始化节则初始化；如未入队则入队，使用CAS设置到tail；如前置节点已经是头节点了且spins非0（表示在上一步中抢占资源仍然是失败且线程已被挂起过），尝试使用Thread.onSpinWait()让出CPU；如节点状态为0则直接设置为WAITING；其他情况下则设置postSpins和spins然后挂起线程，如果超时或需要中断则退出循环并取消当前节点，**自动苏醒或被唤醒后清空节点状态（设置为0，阻止被其他线程并发操作）**。

- release & releaseShared：使用tryRelease或tryReleaseShared释放资源成功后，对头节点调用signalNext方法通知后续节点。

- **void signalNext(Node h)**：如果后继节点存在且非初始化状态，则修改状态并唤醒，修改状态使用getAndUnsetStatus(WAITING)（如果状态为0或1则设置为0，如果为2则仍为2，如果为负数则仍然为负数）。

- **hasQueuedPredecessors()**：是否在同步队列中有前驱节点，即之前是否有线程在等待获取资源。调用getFirstQueuedThread方法，从当前tail往前遍历一遍同步队列，查找到最后一个非空的waiter值。

#### 区别

JDK14之前，通过头节点的waitStatus判断是否需要阻塞，而JDK14之后不需要判断头节点的status，同时独占和共享模式的实现被集成到了一起。

### **ConditionObject(TODO)**

-

### **原子操作**

JDK8使用Unsafe类（sun.misc），JDK9改为使用VarHandle类，JDK14重新使用Unsafe类（jdk.internal.misc）。
> JDK14：*We use jdk.internal Unsafe versions of atomic access methods rather than VarHandles to avoid potential VM bootstrap issues.*

***

## ReentrantLock

***