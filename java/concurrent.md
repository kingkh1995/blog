# [首页](/blog/)

> version: **jdk17**

***

## **Atomic**

### AtomicInteger

内部实现为使用Unsafe操作volatile的int变量。

- **Unsafe**：提供了一些绕开JVM的更低层次操作，并不建议使用。JDK9新增了jdk.internal.misc下的Unsafe类，并进行了重新设计，用于替换sun.misc下的原Unsafe类。

- Unsafe.weakCompareAndSetInt：用于根据指定偏移量使用CAS修改对象的变量，**仅保留了被操作的volatile变量的特性，无法保证其他变量的执行顺序和可见性**，默认实现为调用compareAndSet，但添加了@HotSpotIntrinsicCandidate，故允许HotSpot VM修改其实现以提升性能。

### LongAdder 

基于Striped64实现，而Striped64使用VarHandle类执行原子操作。

- **Striped64**：在高并发情况下CAS操作失败概率上升导致原子类效率降低，而Striped64类使用了**热点分离（空间换时间）** 的策略。其操作针对一个基本值和Cell数组，没有竞争时操作base值，存在竞争时不同线程针对不同的cell执行操作，竞争强度上升后会对Cell数组进行扩容，所以在低并发和高并发下都有很好的效率，**缺点是更新操作无法获取到实时的统计值**，只适用于并发统计的场景。

- **VarHandle**：变量句柄，JDK9新增，用于替代Unsafe类，JDK对其安全性和可移植性有保障，允许被用户使用。该类提供了多种对变量的访问模式，只能通过MethodHandles类的内部类Lookup创建。

### AtomicStampedReference

相比于AtomicReference，解决了ABA问题，操作的变量为私有内部类Pair，其为reference变量捆绑了版本号stamp变量，也是基于VarHandle类实现。

***

## **CopyOnWriteArrayList**

线程安全的List，COW写时复制思想设计，读写分离，适用于读多写少的并发场景。使用volatile和一个对象锁实现，读操作是直接读取效率高，写操作需要获取到对象的同步锁，写操作并不修改原数组，而是创建新数组，完成后使用新数组替换原数组。

***

## CopyOnWriteArraySet

线程安全的Set，使用CopyOnWriteArrayList实现。

- **获取线程安全的Set更推荐的方式：**
    - 使用ConcurrentHashMap的静态方法newKeySet()创建。
    - 使用Collections工具类的newSetFromMap方法包装ConcurrentHashMap。
    - ***也可以使用ConcurrentSkipListSet（使用ConcurrentSkipListMap实现）。***

***

## **ConcurrentHashMap**

### 属性

- volatile Node<K,V>[] nextTable：用于扩容过程中保存新的数组地址。

- volatile int sizeCtl：-1则表示正在初始化；非-1的负数表示正在执行扩容操作，其中高16位为扩容标记，低十六位为当前正在执行扩容操作的线程数+1；如果还未初始化则为需要初始化的数组大小，为0时则使用默认大小16；初始化完成后则为容量（固定为0.75数组大小，等同于HashMap的threshold属性）。

- volatile int transferIndex：扩容处理指针，记录下一个待转移的位置。

- volatile long baseCount：用于统计元素，没有竞争时使用。

- volatile CounterCell[] counterCells：用于统计元素，存在竞争时使用，相关代码改编自LongAdder和Striped64。

### 构造方法

- ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel)：使用initialCapacity除以loadFactor的值计算数组大小（HashMap是直接使用initialCapacity值）；loadFactor也只在构造方法里被使用，负载因子固定为0.75；concurrencyLevel也只是用来保证initialCapacity必须大于等于concurrencyLevel。

### 静态方法

- int spread(int h)：计算哈希值，与HashMap的hash方法逻辑相同，但会将符号为置为0以使得结果为非负数，目的是为了避免与MOVED、TREEBIN、RESERVED这三个特定的负数哈希值冲突。**因为是直接使用key.hashcode()作为参数，也意味着不支持键为null。**

- tabAt()、casTabAt()、setTabAt()：使用Unsafe类对Node数组进行读写操作。

- newKeySet()：JDK8新增，用于创建线程安全的Set，返回值为KeySetView<K,Boolean>类型，支持添加操作（添加时值默认为Boolean.TRUE）。

### 方法

- size()：ConcurrentHashMap并没有size属性，实际上是调用了sumCount方法进行统计。

- sumCount()：统计元素，返回long值，累加baseCount属性和counterCells数组的统计值。

- get(Object key)：使用HashMap相同的公式计算出下标，首先判断下标位置节点是否命中，**如未命中且节点为特殊节点（hash属性小于0）则调用节点的find方法查找**，否则遍历链表查找。

- initTable()：循环尝试直到初始化数组成功。如果sizeCtl小于0（**表示其他线程在执行初始化或者扩容操作**）则执行yield方法进行自旋；初始化操作前需要CAS设置sizeCtl为初始化状态（-1）以及再次检查数组，创建的数组大小为原sizeCtl值，如果是使用无参构造方法创建的则原sizeCtl值为0，数组大小则默认为16，初始化完成之后设置sizeCtl为0.75倍数组大小（表示默认容量）。

- tryPresize(int size)：给定一个容量，尝试保证数组至少能容纳**1.5倍**该值，循环执行直到sizeCtl小于0或成功。如果未初始化，则取需要初始化的数组大小（当前sizeCtl值）和要求的最小数组大小之中的大者来初始化数组（初始化逻辑与initTable方法相同）；如已初始化则需要循环多次调用transfer方法扩容，直到数组大小达到要求或循环因并发操作而终止，扩容前需要检查数组地址以及CAS设置sizeCtl为扩容状态（负数并设置扩容线程数为1）。

- addCount(long x, int check)：用于统计元素个数，并发较小时直接CAS更新baseCount属性，失败则更新counterCells数组的某个位置，多次失败则扩容counterCells，counterCells扩容期间还会尝试更新baseCount。统计完成之后，如果当前个数大于sizeCtl则扩容或协助扩容。

- putVal(K key, V value, boolean onlyIfAbsent)：put操作，自旋重试直到成功，与HashMap不同的是键和值均不能为null。首先如未初始化则初始化；其次如果下标位置处不存在节点则直接CAS设置一个新的Node节点；如果存在节点；先根据节点hash属性判断为ForwardingNode，则调用helpTransfer方法尝试协助扩容并跳转到新数组中；如果onlyIfAbsent为true且下标位置节点匹配则直接返回旧值；最后**获取下标位置节点的同步锁后**执行插入操作，同步操作完成后如果需要升级红黑树则升级。
    - 同步插入操作：获取到同步锁之后，检查到下标位置节点未被改变才能执行；如果节点为Node类型（**hahs属性大于等于0**），执行插入链表操作即可，链表长度超过8时会升级为红黑树；如果节点为TreeBin类型，调用TreeBin的putTreeVal方法进行插入；如果节点为ReservationNode类型，则直接抛出IllegalStateException；如果节点为ForwardingNode类型则本次操作结束并进入下一轮循环（即尝试扩容）。

- treeifyBin(Node<K,V>[] tab, int index)：将链表升级为红黑树。如果数组大小小于64则不升级为红黑树而是调用tryPresize方法尝试扩容为两倍大小；下标位置节点为Node类型时才**获取其同步锁**执行升级操作，将Node链表替换为TreeNode链表后，使用TreeNode链表构造一个TreeBin节点并设置到下标位置处。

- helpTransfer(Node<K,V>[] tab, Node<K,V> f)：协助扩容并返回新数组地址。自旋尝试直到扩容结束或参与扩容线程数超过限制（**扩容时sizeCtl属性后十六位为当前正在执行扩容操作的线程数+1**），当CAS更新sizeCtl属性成功（使扩容线程数加一）才调用transfer方法。

- transfer(Node<K,V>[] tab, Node<K,V>[] nextTab)：转移节点到新数组，**并不是判断nextTable属性而是判断nextTab参数**，如nextTab参数为null则表示新数组还没创建，创建大小为原数组大小两倍的新数组并立即设置到nextTable属性，以及设置transferIndex值为原数组大小；**每个节点转移完成后将原位置处设置为ForwardingNode节点**，扩容完成后CAS修改sizeCtl值使扩容线程数减一，如果是最后一个扩容线程则使用新数组替换原数组。
    - **转移操作为分块处理**，每次处理的槽个数stride由数组大小和cpu核心数计算得到，默认最小值为16，循环处理直到全部转移完成。每轮转移开始前，需要使用CAS将transferIndex指针前移stride位，再从后往前遍历执行。每轮处理时，如果原数组被处理的位置为空则直接设置为ForwardingNode节点；如果当前位置是ForwardingNode节点，则表示当前区块已经或正在被其他线程处理，则执行advance寻找下一个需要处理的区块；其他情况下**获取其同步锁**执行转移，如果是链表或红黑树则进行拆分（与HashMap逻辑相同）并在转移完成后将原数组位置处设置为ForwardingNode节点，如果是ReservationNode则抛出IllegalStateException。

- replaceNode(Object key, V value, Object cv)：替换或移除节点，要求cv为null或值等于cv才执行操作，当value为null时则移除结点，自旋重试直到成功。如果下标位置处存在节点，如果根据hash属性判断节点是ForwardingNode则调用helpTransfer方法协助扩容，否则**获取其同步锁**；如果节点hash值大于等于0（节点为Node类型）则遍历查找并替换或移除；如果节点为TreeBin类型则获取红黑树根节点，调用其findTreeNode方法查找节点并替换或移除，移除树节点时调用TreeBin的removeTreeNode方法，如果该方法返回结果为true则执行退化操作后进入下一轮循环；如果节点为ReservationNode则抛出IllegalStateException。

- **compute方法**：重写了默认实现，自旋重试直到成功。如未初始化则调用initTable方法初始化；**如下标位置为空则创建一个ReservationNode，获取其同步锁后，将其设置到下标位置处以表示占据该位置，成功后执行mapping函数并根据函数结果操作节点**；如果下标位置节点为ForwardingNode则调用helpTransfer尝试协助扩容；如果下标位置节点匹配则直接返回结果；其他情况下获取下标位置节点的同步锁执行同步操作（逻辑与putVal方法相似）。

### Node

与HashMap.Node类基本一致，区别是val属性和next属性为volatile。

- find(int h, Object k)：于get操作中调用，在给定位置处查找键，子类均重写了该方法。

### ForwardingNode

扩容节点，表示当前位置的节点已被转移到扩容后的新数组中，hash属性默认为MOVED（-1），持有新数组的引用。

- find方法：通过nextTable属性定位到新数组中查找节点，调用find方法。

### TreeBin

红黑树代理节点，维护树根节点和链表头节点，key、val、next属性均为null，hash属性默认为TREEBIN（-2）。

- TreeNode<K,V> root：红黑树根节点，**非volatile**。

- volatile TreeNode<K,V> first：链表头节点。

- volatile Thread waiter：正在等待获取锁的线程。

- volatile int lockState：锁状态，使用volatile和CAS操作实现读写锁，包含WRITER（写锁）、WAITER（等待获取锁）、READER（读锁）三个特殊状态，0表示未加锁。

- TreeBin(TreeNode<K,V> b)：构造方法，使用TreeNode链表构造红黑树代理类，红黑树构造方式与HashMap一致，但是构造完成后不需要调整链表，因为使用了first属性维护链表的头节点。

- lockRoot()：阻塞获取当前对象的写锁。

- putTreeVal(int h, K k, V v)：查找或添加节点，如果需要恢复平衡，执行恢复平衡操作需要调用lockRoot方法加锁。

- removeTreeNode(TreeNode<K,V> p)：移除给定的红黑树节点，返回值为当前红黑树是否需要退化。使用与HashMap相同的方式判断是否需要退化，如需退化则直接返回true，否则调用lockRoot方法加锁后移除树节点。

- find方法：自旋直到获取到读锁后，调用根节点的findTreeNode方法查找。

### TreeNode

红黑树节点，与hashMap的TreeNode基本一致，属性均为volatile，并且大部分方法移至TreeBin类中。

- find方法：调用findTreeNode方法。

- findTreeNode(int h, Object k, Class<?> kc)：查找节点，与hashMap的TreeNode类find方法完全相同，先比对hash属性，再使用equals方法判断，如果不等则使用Comparable判断，如果非Comparable类型或compareTo方法比较相等时，则只能在左右子树递归查找。

### ReservationNode

预留节点，表示当前位置已被compute或computeIfAbsent操作预留，hash属性默认为RESERVED（-3）。

- find方法：返回null。

### **使用时安全吗？**

需要谨慎使用compute和computeIfAbsent操作，可能会创建ReservationNode节点，会导致其他操作抛出IllegalStateException，为方法API声明之外的异常。

### **为什么效率高？**

- 支持的并发级别即为数组大小，相比于使用segment粒度更小了，从使用ReentrantLock锁多个槽变为使用同步锁锁单个槽。
- 扩容操作支持并发协助，且不阻塞操作，get操作直接通过ForwardingNode跳转到新数组中，put操作会先尝试协助扩容再跳转到新数组中。

***

## ConcurrentLinkedQueue （TODO）

线程安全的队列

***

### ConcurrentSkipListMap（TODO）

跳跃表

***

## **BlockingQueue extends Queue**

阻塞队列，线程安全的队列，主要用于生产者-消费者模式，不允许接受null元素。有四种操作模式，抛出异常和立即返回结果来自于Queue接口，**put和take操作为阻塞等待，带参数的offer和poll操作为等待指定时间后返回结果**。

### ArrayBlockingQueue

数组实现的有界阻塞队列，只记录队首和队尾的索引，不会产生和销毁额外对象，GC压力小。只使用了一个ReentrantLock，可以设置是否使用公平锁，默认非公平锁。

### LinkedBlockingQueue

链表实现，可有界可无界（容量为Integer.MAX_VALUE），需要创建和销毁Node对象，GC压力大。put和take各使用了一个非公平的ReentrantLock，并发效率更高。

### DelayQueue

无界的延时队列，元素必须实现Delayed接口，使用以延时时间维护的PriorityQueue保存元素，只有元素延迟时间已到才允许被获取。只使用了一个非公平的ReentrantLock，**但使用leader属性保存了第一个阻塞等待获取元素的线程，take操作等同于是公平的**。

### DelayedWorkQueue

ScheduledThreadPoolExecutor使用的队列，与DelayQueue设计相同。不同之处是自己使用数组实现了堆，**堆内元素是ScheduledFutureTask，保存了其在堆中的索引，则remove操作不再需要遍历堆查找到元素**。

### PriorityBlockingQueue

基于优先级的无界阻塞队列，使用Object数组实现，只使用了一个非公平的ReentrantLock。

### SynchronousQueue (TODO)

特殊的阻塞队列，**不保存元素**，支持公平和非公平的方式。每个put必须等待一个take，反之亦然，都是使用内部类Transfer的transfer方法实现。

- TransferQueue：公平模式下使用，队列方式。
- TransferStack：非公平模式下使用，栈方式。

***