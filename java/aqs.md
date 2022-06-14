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

Condition接口的唯一实现类（AQLS的ConditionObject除外）。

- awaitUninterruptibly：无法被打断的锁等待。

### **原子操作**

JDK8使用Unsafe类（sun.misc），JDK9改为使用VarHandle类，JDK14重新使用Unsafe类（jdk.internal.misc）。
> JDK14：*We use jdk.internal Unsafe versions of atomic access methods rather than VarHandles to avoid potential VM bootstrap issues.*

***

## Lock & ReadWriteLock & Condition

- Lock：锁的顶级接口。
- ReadWriteLock：读写锁的的顶级接口，使用readLock和writeLock方法返回读锁和写锁对象。
- Condition：条件的顶级接口，通过Lock的newCondition方法获取。

***

## **ReentrantLock**

可重入的排他锁，支持公平和非公平方式，默认非公平，基于继承内部类Sync实现。

### **Sync**

抽象类，继承自AQS，子类为NonfairSync和FairSync。

- tryLock()：state为0时直接CAS更新state，否则通过AOS的getExclusiveOwnerThread方法判断是否可以重入。

- initialTryLock()：抽象方法，**lock时先调用该方法获取锁失败后才使用AQS的acquire方法**。NonfairSync直接CAS更新state，失败则判断是否能重入，FairSync还要使用AQS的hasQueuedThreads方法判断前面是否有线程等待获取锁。

***

## **ReentrantReadWriteLock**

可重入的读写锁，支持公平和非公平方式，默认也是非公平。写锁为独占模式，读锁为共享模式，**读锁被持有时无法获取写锁，但同一个线程获取了写锁可以继续获取读锁，可能出现‘写饥饿’现象**。

### ReadLock & WriteLock

读锁和写锁，使用ReentrantReadWriteLock的Sync实现，读锁使用共享模式，无法创建Condition，写锁使用排他模式，可以创建Condition。

### **Sync**

抽象类，继承自AQS，子类为NonfairSync和FairSync。

#### 属性

- state：使用高16位表示读锁，低16记录表示写锁。

- readHolds：使用ThreadLocal记录每个线程的读锁加锁次数。
  
- cachedHoldCounter：记录最后一个获取到读锁的线程的读锁加锁次数。

- firstReader & firstReaderHoldCount：记录第一个获取到读锁的线程和其读锁加锁次数。

#### 自身方法

- readerShouldBlock()：抽象方法，判断是否应该被阻止获取读锁。
  - NonfairSync：直接返回apparentlyFirstQueuedIsExclusive()结果，同步队列第一个节点不能是独占模式，**即不能有线程正在等待获取写锁**。
  - FairSync：直接返回hasQueuedPredecessors()结果，即同步队列前面不能有节点等待。

- writerShouldBlock()：抽象方法，判断是否应该被阻止获取写锁。
  - NonfairSync：永远返回false，即任何情况都可以尝试获取写锁。
  - FairSync：也是直接返回hasQueuedPredecessors()结果。

- fullTryAcquireShared()：与tryAcquireShared方法基本相同，添加了共享模式的可重入处理，从readHolds属性中获取当前线程读锁加锁次数。

#### AQS方法实现

- tryAcquire()：state不为0时，如果存在读锁（独占模式资源数为0）或当前线程未持有锁则失败，**即只有当前线程持有写锁时才允许重入**；state为0时，调用writerShouldBlock方法判断能获取读锁后CAS更新state成功表示独占模式获取资源成功。

- tryAcquireShared()：如果存在写锁（独占模式资源数不为0）且锁并不是当前线程持有则失败，**即如果有其他线程持有写锁则不允许获取读锁**；调用readerShouldBlock判断能获取读锁后CAS更新state成功表示共享资源获取成功；之后更新读锁加锁次数的相关记录；最后无论是否成功，调用fullTryAcquireShared方法用于处理**锁重入和CAS失败的场景**。

***

## **StampedLock**

非可重入，读写锁，JDK8新增，**未实现Lock接口，未使用AQS实现**，适用于读多写少的场景，能避免写饥饿现象，**支持写锁，悲观读锁、乐观读锁以及锁互相转换**。

### **属性**

- volatile long state：使用低7位表示悲观读锁，剩余位表示写锁，**只使用第8位表示写锁加锁状态，整体作为更新版本号**。
  - WBIT：10000000，用于判断写锁加锁状态；
  - RBITS：1111111，用于获取悲观读锁计数；
  - RFULL：1111110，悲观读锁计数阈值；
  - ABITS：11111111，用于获取写锁和悲观读锁加锁状态。
  - RSAFE：第7和8位为0，表示可以安全的获取悲观读锁，即未加写锁以及读锁计数未溢出；
  - SBITS：用于获取读锁计数，低7位为0其余为1。

- int readerOverflow：用于保存溢出的悲观读锁次数，因为悲观读锁计数只能支持到126，127用于标记计数溢出。
- volatile Node head & volatile Node tail：同步队列。
  - **Node**：抽象类，子类为写节点WriterNode和子悲观读节点ReaderNode。
  - volatile int status：状态，0初始化、1等待、负数取消；
  - volatile ReaderNode cowaiters：条件等待队列，悲观读节点属性。

### **方法**

- long tryAcquireWrite()：锁空闲时（低8位全为0）尝试获取写锁，成功则返回新的state值，失败返回0。CAS更新state成功后会添加storeStoreFence，防止屏障前后的store指令重排。

- long tryAcquireRead()：写锁未被获取时自旋尝试获取悲观读锁，写锁已被获取则返回0，悲观读锁计数未超过RFULL则直接CAS更新state，否则使用tryIncReaderOverflow方法获取。

- long tryIncReaderOverflow(long s)：CAS更新悲观读锁计数为127，并增加readerOverflow计数。

- long releaseWrite(long s)：释放写锁并返回state值，**写锁计数加一而不是减一，第8位仍然会更新为0，表示未加锁，如果计数溢出则重置**，之后唤醒同步队列下一个节点。

- **long acquireWrite(boolean interruptible, boolean timed, long time)**：尝试获取写锁，与AQS思路acquire方法思路相同，记录自旋次数，更多的尝试自旋而不是挂起线程，被打断则返回1。

- **long acquireRead(boolean interruptible, boolean timed, long time)**：尝试获取悲观读锁。

### **公共API**

- asWriteLock() & asReadWriteLock() & asWriteLock()：返回Lock接口和ReadWriteLock接口的视图，**注意是非可重入锁**。

- long writeLock()：获取写锁并返回state，先假定锁是空闲状态直接尝试CAS更新state一次，失败则使用acquireWrite方法获取。

- long readLock()：获取悲观读锁并返回state，先假定可以安全的获取悲观读锁（第7和8位不为1）使用CAS更新state一次，失败则使用acquireRead方法获取。
  
- long tryOptimisticRead()：获取乐观读锁，只要未加写锁则表示获取乐观读锁成功，并返回更新版本号（后七位置为0），否则返回0表示失败。
  
- boolean validate(long stamp)：验证给定锁状态是否还有效，只要写锁计数未变则表示有效。

- void unlockWrite(long stamp)：释放写锁，参数为加锁时返回的state值，要求state未变更，调用releaseWrite方法。

- void unlockRead(long stamp)：释放悲观读锁，悲观读锁计数减一，如果此时锁空闲了则唤醒同步队列下一个节点。

- tryConvertToWriteLock & tryConvertToReadLock & tryConvertToOptimisticRead：使用加锁时返回的stamp将锁转换为其他锁，**因为不可重入，所以不能同时加读锁和写锁，只能做锁转换**。

***

## **Semaphore**

信号量，支持公平和非公平方式，创建时需要指定资源总数permits。

### **Sync**

抽象类，继承自AQS，使用共享模式获取资源，permits是state的默认值。

- int tryAcquireShared(int acquires)：获取资源
  - NonfairSync：自旋使用CAS更新state直到成功。
  - FairSync：相比NonfairSync，如果使用hasQueuedPredecessors方法判断同步队列存在前置节点则直接返回-1失败。
  
- boolean tryReleaseShared(int releases)：释放资源，自旋使用CAS更新state直到成功。

***

## **CountDownLatch**

闭锁，创建时需要指定count，调用await方法的线程等待，直到其他线程使用countDown方法将count减至0后才继续执行，只能使用一次，count无法重置。

### **Sync**

继承自AQS，使用共享模式获取资源，count是state的默认值。

- int tryAcquireShared(int acquires)：state等于0时返回1，否则返回-1，**即只有当state为0时acquire方法才能执行成功**。
  
- boolean tryReleaseShared(int releases)：自旋CAS操作使state减一，state减为0了才返回true。

### **方法**

- await()：sync.acquireSharedInterruptibly(1)，由于AQS真正获取资源调用的是tryAcquireShared方法，所以会导致阻塞直到state为0，而一旦state为0了，则永远总是会成功。
- countDown()：sync.releaseShared(1)，AQS真正释放资源调用的是tryReleaseShared方法，故只有当state减为1后才视为释放资源成功，并唤醒同步队列中的节点（调用await方法等待中的线程）。

***

## **CyclicBarrier**

循环栅栏，基于ReentrantLock和Condition的等待/通知模式实现，线程使用await方法后阻塞，直到await的次数达到parties值则唤醒所有线程，之后可以重置再次使用。

- final int parties：创建时指定。
- final Runnable barrierCommand：创建时指定，栅栏被绊倒后最后一个调用await的会执行任务。
- Generation generation：当前栅栏状态，只有一个布尔属性broken标识栅栏是否被破坏。
- ReentrantLock lock & Condition trip：栅栏的保护锁及其条件。
- int count：正在等待的parties数量。
- int dowait(boolean timed, long nanos)：**await调用的方法**。获取到锁之后才执行，如果当前栅栏已经被打破则抛出BrokenBarrierException异常；count减一，如果为0了，表示栅栏被打破，执行barrierCommand，然后调用nextGeneration方法后return；如果count未减为0，则等待在条件上，直到被打破、被破坏、被打断或超时。
- nextGeneration()：使用条件唤醒所有线程并重置栅栏。
- void reset()：打破当前栅栏，然后nextGeneration方法重置。

***

## **Phaser**

移相器，升级的栅栏，能维护一个树状的层级关系，阶段将沿着树状结构完成，可以动态地控制每个阶段的任务总量。

- volatile long state：0-15表示还未到达的数量，16-31表示任务总量，32-62表示栅栏的阶段，63位标识栅栏是否被终止。

- register() & bulkRegister()：注册任务，即动态调整任务总量，任务需要全部到达才能继续进入下一阶段。

- arrive()：一个任务到达，并返回达到的阶段。
  
- arriveAndDeregister()：一个任务之后删除掉该任务。

- arriveAndAwaitAdvance()：一个任务达到，并阻塞等待进入下一个阶段。
  
- awaitAdvance(int phase)：等待直到指定阶段。

***

## **Exchanger\<V>**

用于两个线程交换数据，可以视作双向的同步队列，可以重复使用。

***




