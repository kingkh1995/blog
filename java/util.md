# [首页](/blog/)

> version: **jdk17**

***

### HashMap

#### 属性

- threshold：容量阈值，为Node数组大小乘以装载因子的值，**只有未初始化时才用于保存需要初始化的数组大小**。
    - HashMap创建时不初始化Node数组，只使用tableSizeFor方法计算出数组大小并保存到threshold属性，首次put操作时才使用该值创建Node数组。

#### 静态方法

- hash(Object key)：哈希值高16位不变，低16位变为高16位和低16位异或的值，目的是用于减少hash碰撞。**支持键为null，返回0。**

- tableSizeFor(int c)：计算数组大小，保证值为2的幂。

- comparableClassFor(Object x)：用于TreeNode，如果当前键实现了Comparable则返回其Class对象，否则返回null。

- tieBreakOrder(Object a, Object b)：用于TreeNode，当hash属性相同且equals方法返回false时，在判断非Comparable类型或compareTo方法比较相等后调用，先比较className，如果相同或者某个参数为null，则使用System.identityHashCode()获取默认哈希值进行比较。**该方法不会返回0，且只具有一致性（即对象没有发生变化则结果不会变化），只能在插入节点时使用，因为不具备对称性则查找节点时无法使用，如果无法比较则只能同时在左子树和右子树中查找**。

#### 方法

- resize()：用户初始化数组和扩容，初始化则默认数组大小为16，装载因子为0.75；若扩容则扩容为两倍，链表会被拆分为两条，红黑树也会被拆成两棵（可能退化为链表）。

- getNode(Object key)：查找键，使用公式 (n-1)&hash（因为数组大小n为2的幂可通过位运算快速取模）计算下标，查找命中要求hash属性相同以及==或equals方法成立。

- putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict)：如果未初始化数组则调用resize方法初始化，如果是插入新值，若链表长度大于8则调用treeifyBin方法将链表升级为红黑树，modCount和size属性加一，若size超过了阈值则触发扩容，最后将evict参数传递给afterNodeInsertion回调方法（在LinkedHashMap中使用），返回null，表示新插入成功。

- treeifyBin()：将链表转为红黑树，如果数组大小小于64则直接扩容，否则遍历链表将Node节点替换为TreeNode节点，替换完成后调用TreeNode的treeify方法构造红黑树。

- removeNode(int hash, Object key, Object value, boolean matchValue, boolean movable)：先获取到需要移除的节点（并没有调用getNode方法，其实是相同的代码），如果是TreeNode则调用removeTreeNode方法，移除完毕后modCount加一，size减一，**注意HashMap只会扩容并不会缩容**。
    - movable参数用于传递给TreeNode的removeTreeNode方法，为false时表示不移动红黑树的其他节点（untreeify和moveRootToFront），**只有在迭代器（HashIterator、LinkedHashIterator）的remove方法中该参数才为false，因为如果允许移动会破坏迭代过程**。

- keySet()：返回一个内部类KeySet对象，为该HashMap对象键集合的视图，支持remove和clear操作。

- values()：返回一个内部类Values对象，为该HashMap对象值集合的视图，支持remove和clear操作。

- entrySet()：返回一个内部类EntrySet对象，为该HashMap对象Entry集合的视图，支持remove和clear操作。

#### TreeNode

继承自LinkedHashMap.Entry，LinkedHashMap.Entry又继承自HashMap.Node，拥有prev和next指针，故红黑树同时也是双向链表。节点键的比较使用hash属性，如果相等则首先判断是否是Comparable类型其次比较className最后比较System.identityHashCode。

- split()：扩容时调用，使用next指针遍历并按照hahs属性拆分成两棵红黑树（**相当于视作链表拆分**），如果新树节点的个数小于6则退化为链表（因为作为链表拆分时统计了节点数），否则构造红黑树。

- untreeify()：红黑树退化，因为同时也是链表，故直接遍历并将TreeNode节点替换为Node节点。

- treeify()：构造红黑树，使用next指针遍历从头开始构造红黑树，构造完成后调用moveRootToFront方法。

- moveRootToFront()：将红黑树根节点移动到链表的头部，并设置到槽上。

- putTreeVal()：插入节点，找到其父节点后，插入链表中其父节点之后，然后插入树中并恢复平衡，最后调用moveRootToFront方法。

- removeTreeNode()：移除节点，首先从链表中删除（处理前后指针），然后从树中删除，通过parent指针找到真正的根节点（**由于存在movable为false的场景，故槽上的节点不一定是根节点**），判断是否需要退化（**判断特定的树形，节点数可能为0-10**），不需要退化则执行红黑树删除节点操作，最后调用moveRootToFront方法。
    - **注意movable为false时（使用迭代器移除场景下）并不会做退化操作和moveRootToFront操作。**
    
#### **链表的长度可能大于8吗？**

可能！数组大小小于64时，链表长度大于8并不会升级为红黑树而是触发扩容，扩容后链表虽然会被拆分为两条，但长度仍然可能大于8。

#### **红黑树的节点数可能小于6个吗？**

可能！移除红黑树节点时，并不是通过统计结点个数来判断是否需要退化，而是通过判断是特定的树型（**由于红黑树的特性可以通过树型判断节点数是否可能过小**），这就导致节点数小于6个的特定树型不会触发退化；其次使用迭代器移除元素时也不会触发红黑树退化操作。

***

### LinkedHashMap

继承自HashMap，同时也是双向链表，支持以插入顺序或访问顺序维护，使用的节点类型为内部类Entry，继承自HashMap.Node，增加了before和after指针。

- afterNodeAccess(Node<K,V> e)：查找元素时，如果双向链表维护方式为访问顺序，那么在节点被访问后，需要调整双向链表，**modCount也会加一**。

***

### ConcurrentHashMap

#### 属性

- volatile Node<K,V>[] nextTable：用于扩容过程中保存新的数组地址。

- volatile int sizeCtl：-1则表示正在初始化；非-1的负数表示正在执行扩容操作，其中高16位为扩容标记，低十六位为当前正在执行扩容操作的线程数+1；如果还未初始化则为需要初始化的数组大小，为0时则使用默认大小16；初始化完成后则为容量（固定为0.75数组大小，等同于HashMap的threshold属性）。

- volatile int transferIndex：扩容处理指针，记录下一个待转移的位置。

- volatile long baseCount：用于统计元素，没有竞争时使用。

- volatile CounterCell[] counterCells：用于统计元素，存在竞争时使用，相关代码改编自LongAdder和Striped64。

#### 构造方法

- ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel)：使用initialCapacity除以loadFactor的值计算数组大小（HashMap是直接使用initialCapacity值）；loadFactor也只在构造方法里被使用，负载因子固定为0.75；concurrencyLevel也只是用来保证initialCapacity必须大于等于concurrencyLevel。

#### 静态方法

- int spread(int h)：计算哈希值，与HashMap的hash方法逻辑相同，但会将符号为置为0以使得结果为非负数，目的是为了避免与MOVED、TREEBIN、RESERVED这三个特定的负数哈希值冲突。**因为是直接使用key.hashcode()作为参数，也意味着不支持键为null。**

- tabAt()、casTabAt()、setTabAt()：使用Unsafe类对Node数组进行读写操作。

#### 方法

- size()：ConcurrentHashMap并没有size属性，实际上是调用了sumCount方法进行统计。

- sumCount()：统计元素，返回long值，累加baseCount属性和counterCells数组的统计值。

- get(Object key)：使用HashMap相同的公式计算出下标，首先判断下标位置节点是否命中，**如未命中且节点为特殊节点（hash属性小于0）则调用节点的find方法查找**，否则遍历链表查找。

- initTable()：循环尝试直到初始化数组成功。如果sizeCtl小于0（**表示其他线程在执行初始化或者扩容操作**）则执行yield方法进行自旋；初始化操作前需要CAS设置sizeCtl为初始化状态（-1）以及再次检查数组，创建的数组大小为原sizeCtl值，如果是使用无参构造方法创建的则原sizeCtl值为0，数组大小则默认为16，初始化完成之后设置sizeCtl为0.75倍数组大小（表示默认容量）。

- tryPresize(int size)：给定一个容量，尝试保证数组至少能容纳**1.5倍**该值，循环执行直到sizeCtl小于0或成功。如果未初始化，则取需要初始化的数组大小（当前sizeCtl值）和要求的最小数组大小之中的大者来初始化数组（初始化逻辑与initTable方法相同）；如已初始化则需要循环多次调用transfer方法扩容，直到数组大小达到要求或循环因并发操作而终止，扩容前需要检查数组地址以及CAS设置sizeCtl为扩容状态（负数并设置扩容线程数为1）。

- addCount(long x, int check)：用于统计元素个数，并发较小时直接CAS更新baseCount属性，失败则更新counterCells数组的某个位置，多次失败则扩容counterCells，counterCells扩容期间还会尝试更新baseCount。统计完成之后，如果当前个数大于sizeCtl则扩容或协助扩容。

- putVal(K key, V value, boolean onlyIfAbsent)：put操作，自旋重试直到成功，与HashMap不同的是键和值均不能为null。首先如未初始化则初始化；其次如果下标位置处不存在节点则直接CAS设置一个新的Node节点；如果存在节点；先根据节点hash属性判断为ForwardingNode，则调用helpTransfer方法尝试协助扩容；如果onlyIfAbsent为true且下标位置节点匹配则直接返回旧值；最后**获取下标位置节点的同步锁后**执行插入操作，同步操作完成后如果需要升级红黑树则升级。
    - 同步插入操作：获取到同步锁之后，检查到下标位置节点未被改变才能执行；如果节点为Node类型（**hahs属性大于等于0**），执行插入链表操作即可，链表长度超过8时会升级为红黑树；如果节点为TreeBin类型，调用TreeBin的putTreeVal方法进行插入；如果节点为ReservationNode类型，则直接抛出IllegalStateException；如果节点为ForwardingNode类型则本次操作结束并进入下一轮循环。

- treeifyBin(Node<K,V>[] tab, int index)：将链表升级为红黑树。如果数组大小小于64则不升级为红黑树而是调用tryPresize方法尝试扩容为两倍大小；下标位置节点为Node类型时才**获取其同步锁**执行升级操作，将Node链表替换为TreeNode链表后，使用TreeNode链表构造一个TreeBin节点并设置到下标位置处。

- helpTransfer(Node<K,V>[] tab, Node<K,V> f)：协助扩容，自旋尝试直到扩容或初始化结束（**sizeCtl为非负数**）。当前正在参与扩容线程数不能超过限制（**扩容时sizeCtl属性后十六位为当前正在执行扩容操作的线程数+1**），CAS设置sizeCtl属性为扩容状态（扩容线程数加一）成功才调用transfer方法。

- transfer(Node<K,V>[] tab, Node<K,V>[] nextTab)：转移节点到新数组，**并不是判断nextTable属性而是判断nextTab参数**，如nextTab参数为null则表示新数组还没创建，创建大小为原数组大小两倍的新数组并立即设置到nextTable属性，以及设置transferIndex值为原数组大小。
    - **转移操作为分块处理**，每次处理的槽个数stride由数组大小和cpu核心数计算得到，默认最小值为16，循环处理直到全部转移完成。每轮转移开始前，需要使用CAS将transferIndex指针前移stride位，再从后往前遍历执行。每轮处理时，如果原数组被处理的位置为空则直接设置为ForwardingNode节点；如果当前位置是ForwardingNode节点，则表示当前区块正在被其他线程转移（其他操作触发），则执行advance寻找下一个需要处理的区块；其他情况下**获取其同步锁**执行转移，如果是链表或红黑树则进行拆分（与HashMap逻辑相同）并在转移完成后将原数组位置处设置为ForwardingNode节点，如果是ReservationNode则抛出IllegalStateException。

- replaceNode(Object key, V value, Object cv)：替换或移除节点，要求cv为null或值等于cv才执行操作，当value为null时则移除结点，自旋重试直到成功。如果下标位置处存在节点，如果根据hash属性判断节点是ForwardingNode则调用helpTransfer方法协助扩容，否则**获取其同步锁**；如果节点hash值大于等于0（节点为Node类型）则遍历查找并替换或移除；如果节点为TreeBin类型则获取红黑树根节点，调用其findTreeNode方法查找节点并替换或移除，移除树节点时调用TreeBin的removeTreeNode方法，如果该方法返回结果为true则执行退化操作后进入下一轮循环；如果节点为ReservationNode则抛出IllegalStateException。

- **compute方法**：重写了默认实现，自旋重试直到成功。如未初始化则调用initTable方法初始化；**如下标位置为空则创建一个ReservationNode，获取其同步锁后，将其设置到下标位置处以表示占据该位置，成功后执行mapping函数并根据函数结果操作节点**；如果下标位置节点为ForwardingNode则调用helpTransfer尝试协助扩容；如果下标位置节点匹配则直接返回结果；其他情况下获取下标位置节点的同步锁执行同步操作（逻辑与putVal方法相似）。

#### Node

与HashMap.Node类基本一致，区别是val属性和next属性为volatile。

- find(int h, Object k)：于get操作中调用，在给定位置处查找键，子类均重写了该方法。

#### ForwardingNode

扩容节点，表示当前位置的节点已被转移到扩容后的新数组中，hash属性默认为MOVED（-1），持有新数组的引用。

- find方法：通过nextTable属性定位到新数组中查找节点，调用find方法。

#### TreeBin

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

#### TreeNode

红黑树节点，与hashMap的TreeNode基本一致，属性均为volatile，并且大部分方法移至TreeBin类中。

- find方法：调用findTreeNode方法。

- findTreeNode(int h, Object k, Class<?> kc)：查找节点，与hashMap的TreeNode类find方法完全相同，先比对hash属性，再使用equals方法判断，如果不等则使用Comparable判断，如果非Comparable类型或compareTo方法比较相等时，则只能在左右子树递归查找。

#### ReservationNode

预留节点，表示当前位置已被compute或computeIfAbsent操作预留，hash属性默认为RESERVED（-3）。

- find方法：返回null。

#### **使用时安全吗？**

需要谨慎使用compute和computeIfAbsent操作，可能会创建ReservationNode节点，会导致其他操作抛出IllegalStateException，为方法API声明之外的异常。

#### **真的是线程安全的吗？**

***