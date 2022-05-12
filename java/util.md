# [首页](/blog/)

> version: **jdk17**

***

## ArrayList

动态数组，非线程安全，默认容量10，每次扩容为1.5倍。

***

## **CopyOnWriteArrayList**

线程安全的List，使用COW思想实现，适用于读多写少的并发场景。

***

## **HashMap**

### 属性

- threshold：容量阈值，为Node数组大小乘以装载因子的值，**只有未初始化时才用于保存需要初始化的数组大小**。
    - HashMap创建时不初始化Node数组，只使用tableSizeFor方法计算出数组大小并保存到threshold属性，首次put操作时才使用该值创建Node数组。

### 静态方法

- hash(Object key)：哈希值高16位不变，低16位变为高16位和低16位异或的值，目的是用于减少hash碰撞。**支持键为null，返回0。**

- tableSizeFor(int c)：计算数组大小，保证值为2的幂。

- comparableClassFor(Object x)：用于TreeNode，如果当前键实现了Comparable则返回其Class对象，否则返回null。

- tieBreakOrder(Object a, Object b)：用于TreeNode，当hash属性相同且equals方法返回false时，在判断非Comparable类型或compareTo方法比较相等后调用，先比较className，如果相同或者某个参数为null，则使用System.identityHashCode()获取默认哈希值进行比较。**该方法不会返回0，且只具有一致性（即对象没有发生变化则结果不会变化），只能在插入节点时使用，因为不具备对称性则查找节点时无法使用，如果无法比较则只能同时在左子树和右子树中查找**。

### 方法

- resize()：用户初始化数组和扩容，初始化则默认数组大小为16，装载因子为0.75；若扩容则扩容为两倍，链表会被拆分为两条，红黑树也会被拆成两棵（可能退化为链表）。

- getNode(Object key)：查找键，使用公式 (n-1)&hash（因为数组大小n为2的幂可通过位运算快速取模）计算下标，查找命中要求hash属性相同以及==或equals方法成立。

- putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict)：如果未初始化数组则调用resize方法初始化，如果是插入新值，若链表长度大于8则调用treeifyBin方法将链表升级为红黑树，modCount和size属性加一，若size超过了阈值则触发扩容，最后将evict参数传递给afterNodeInsertion回调方法（在LinkedHashMap中使用），返回null，表示新插入成功。

- treeifyBin()：将链表转为红黑树，如果数组大小小于64则直接扩容，否则遍历链表将Node节点替换为TreeNode节点，替换完成后调用TreeNode的treeify方法构造红黑树。

- removeNode(int hash, Object key, Object value, boolean matchValue, boolean movable)：先获取到需要移除的节点（并没有调用getNode方法，其实是相同的代码），如果是TreeNode则调用removeTreeNode方法，移除完毕后modCount加一，size减一，**注意HashMap只会扩容并不会缩容**。
    - movable参数用于传递给TreeNode的removeTreeNode方法，为false时表示不移动红黑树的其他节点（untreeify和moveRootToFront），**只有在迭代器（HashIterator、LinkedHashIterator）的remove方法中该参数才为false，因为如果允许移动会破坏迭代过程**。

- keySet()：返回一个内部类KeySet对象，为该HashMap对象键集合的视图，支持remove和clear操作。

- values()：返回一个内部类Values对象，为该HashMap对象值集合的视图，支持remove和clear操作。

- entrySet()：返回一个内部类EntrySet对象，为该HashMap对象Entry集合的视图，支持remove和clear操作。

### TreeNode

继承自LinkedHashMap.Entry，LinkedHashMap.Entry又继承自HashMap.Node，拥有prev和next指针，故红黑树同时也是双向链表。节点键的比较使用hash属性，如果相等则首先判断是否是Comparable类型其次比较className最后比较System.identityHashCode。

- split()：扩容时调用，使用next指针遍历并按照hahs属性拆分成两棵红黑树（**相当于视作链表拆分**），如果新树节点的个数小于6则退化为链表（因为作为链表拆分时统计了节点数），否则构造红黑树。

- untreeify()：红黑树退化，因为同时也是链表，故直接遍历并将TreeNode节点替换为Node节点。

- treeify()：构造红黑树，使用next指针遍历从头开始构造红黑树，构造完成后调用moveRootToFront方法。

- moveRootToFront()：将红黑树根节点移动到链表的头部，并设置到槽上。

- putTreeVal()：插入节点，找到其父节点后，插入链表中其父节点之后，然后插入树中并恢复平衡，最后调用moveRootToFront方法。

- removeTreeNode()：移除节点，首先从链表中删除（处理前后指针），然后从树中删除，通过parent指针找到真正的根节点（**由于存在movable为false的场景，故槽上的节点不一定是根节点**），判断是否需要退化（**判断特定的树形，节点数可能为0-10**），不需要退化则执行红黑树删除节点操作，最后调用moveRootToFront方法。
    - **注意movable为false时（使用迭代器移除场景下）并不会做退化操作和moveRootToFront操作。**
    
### **链表的长度可能大于8吗？**

可能！数组大小小于64时，链表长度大于8并不会升级为红黑树而是触发扩容，扩容后链表虽然会被拆分为两条，但长度仍然可能大于8。

### **红黑树的节点数可能小于6个吗？**

可能！移除红黑树节点时，并不是通过统计结点个数来判断是否需要退化，而是通过判断是特定的树型（**由于红黑树的特性可以通过树型判断节点数是否可能过小**），这就导致节点数小于6个的特定树型不会触发退化；其次使用迭代器移除元素时也不会触发红黑树退化操作。

***

## LinkedHashMap

继承自HashMap，同时维护了一条双向链表，支持插入顺序或访问顺序，使用的节点类型为内部类Entry，继承自HashMap.Node，增加了before和after指针。

- afterNodeAccess(Node<K,V> e)：查找元素时，如果双向链表维护方式为访问顺序，那么在节点被访问后，需要调整双向链表，**modCount也会加一**。

***

## IdentityHashMap

- **在比较键和值时使用引用相等性代替对象相等性**，违反了Map的规范了，只适用于特殊的场景。

- **使用线性探测法而非拉链法解决冲突**，默认大小为32，使用率默认为2/3（size等于1/3数组大小，但一个mapping占据两个槽，故使用率为2/3）。

- **使用Object数组而不是Entry数组**，重新散列使键的哈希值为偶数，故键保存在偶数槽中，值保存在键的下一个位置。

- **允许空键和空值**，空键会被替换为常量，因为数组偶数槽为空表示键不存在。

- **使用迭代器移除前，会拷贝一份原数组用于后续迭代**，并从原数组中移除元素，因为线性探测法移除需要将当前键簇剩余所有键全部重新插入，故会破坏迭代。

***

## WeakHashMap

与HashMap实现基本一致但并无红黑树，其Entry类继承自WeakReference，即使用WeakReference保存键，当键被GC回收后，视作键值对已被回收，get、put、resize等操作都会附带移除已被回收键值对。

***

## HashSet

通过HashMap实现，Entry的值默认为常量（Object对象）。

***

## CopyOnWriteArraySet

线程安全的Set，使用CopyOnWriteArrayList实现。


- **获取线程安全的Set更推荐的方式：**
    - 使用ConcurrentHashMap的静态方法newKeySet()创建。
    - 使用Collections工具类的newSetFromMap方法包装ConcurrentHashMap。

***

## Queue

- add：新元素入队尾，容量不足则抛出异常。
- remove：队首元素出队，为空则抛出异常。
- element：查看队首元素，为空则抛出异常。
- offer：新元素入队尾，容量不足返回false。
- poll：队首元素出队，为空则返回null。
- peek：查看队首元素，为空则返回null。

***

## Deque(deck) extends Queue

继承自Queue，双向队列，first视作队首，last为队尾；来自Queue的方法语义不变，get方法与element方法语义相同；可作为栈使用，push方法入队首，容量不足则抛出异常，pop方法出队首，为空则抛出异常。

***

## LinkedList

双向链表，容量无限，继承自AbstractSequentialList（顺序访问List），实现了Deque接口，非线程安全。

***