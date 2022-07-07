# [首页](/blog/)

> version: **jdk17**

***

## ArrayList

动态数组，非线程安全，支持随机访问（实现了RandomAccess），默认容量10，每次扩容为1.5倍。

- subList()：返回私有内部类SubList对象，所有操作全部被代理到ArrayList对象上。

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

## HashSet

通过HashMap实现，Entry的值默认为常量（Object对象）。

***

## LinkedHashSet

继承自HashSet，使用HashSet中专门为其提供的构造方法：
```java
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```

***

## **EnumMap**

使用**固定大小为枚举类型中值的个数的数组分开保存枚举键和值**，使用枚举值的ordinal确定下标，键不允许为null，但值允许为null。

***

## **EnumSet**

抽象类，有RegularEnumSet和JumboEnumSet两种内部实现，只能使用静态工厂方法创建对象。**每个对象都持有一个包含了全部枚举值且按ordinal排序的数组的引用，该数组被赋予了常量的语义会被所有对象共享**。RegularEnumSet和JumboEnumSet都是使用位运算提高效率，**将每个long类型变量视作长度64的布尔数组，用来标识枚举值是否存在**。枚举类型中值的个数小于等于64时实现为RegularEnumSet，其持有一个long类型变量，size操作实现为调用Long.bitCount()方法；大于64时实现为JumboEnumSet，其持有long类型数组，操作时会统计size。

***

## **IdentityHashMap**

- **在比较键和值时使用引用相等性代替对象相等性**，违反了Map的规范了，只适用于特殊的场景。

- **使用线性探测法而非拉链法解决冲突**，默认大小为32，使用率默认为2/3（size等于1/3数组大小，但一个mapping占据两个槽，故使用率为2/3）。

- **使用Object数组而不是Entry数组**，重新散列使键的哈希值为偶数，故键保存在偶数槽中，值保存在键的下一个位置。

- **允许空键和空值**，空键会被替换为常量，因为数组偶数槽为空表示键不存在。

- **使用迭代器移除前，会拷贝一份原数组用于后续迭代**，并从原数组中移除元素，因为线性探测法移除需要将当前键簇剩余所有键全部重新插入，故会破坏迭代。

***

## WeakHashMap

只使用拉链法实现，其Entry类继承自WeakReference，使用WeakReference保存键，当键被GC回收则视作键值对被回收，get、put、resize等操作都会附带移除已被回收键值对。支持空键，使用常量NULL_KEY替换空键，故空键永远不会被回收。

***

## TreeMap

红黑树（2-3-4树），实现NavigableMap接口，非线程安全。如果键非Comparable类型且构造时未指定Comparator，则运行时会抛出ClassCastException（不存在Comparator会直接转换为Comparable），**注意是通过Comparable或在Comparator去重**，迭代时以中序遍历的方式迭代。

### *NavigableMap*

可导航Map接口，继承自SortedMap接口，支持floor（小于等于）、lower（严格小于）、celling（大于等于）、higher（严格大于）。

***

## TreeSet

实现NavigableSet接口，通过TreeMap实现。

***

## Queue

- add：新元素入队尾，容量不足则抛出异常。
- remove：队首元素出队，为空则抛出异常。
- element：查看队首元素，为空则抛出异常。
- offer：新元素入队尾，容量不足返回false。
- poll：队首元素出队，为空则返回null。
- peek：查看队首元素，为空则返回null。

***

## PriorityQueue

无界最小堆，要求元素是Comparable类型或者构造时指定Comparator，否则运行时会抛出转换异常，内部使用了Object数组，默认初始容量11。

- grow：如果容量不足则扩容，小于64时则扩容为两倍，否则增长50%。
- remove：移除任意元素操作，首先需要遍历找到元素位置，如果为队尾元素则直接移除即不破坏堆的性质，否则使用队尾元素替换，先下沉，如元素未下沉则上浮。
    - 如果能成功下沉则**被移除元素所在的子堆**和**以被移除元素作为堆顶的子堆**都恢复了堆的性质；但**队尾元素可能并不位于以被移除元素作为堆顶的子堆**，故队尾元素可能比被移除元素小而无法下沉，这样就破坏了被移除元素所在的子堆的性质，所以碰到无法下沉的情景时需要上浮以恢复整个堆的性质。

***

## Deque\[deck\]

继承自Queue接口，双向队列，来自Queue的方法语义不变，get方法与element方法语义相同；可作为栈使用，队首作为栈顶，push时容量不足则抛出异常，pop时栈为空则抛出异常。

***

## LinkedList

双向链表，非线程安全，继承自AbstractSequentialList（表示只支持顺序访问），实现了Deque接口。

***

## ArrayDeque

基于数组实现的Deque，非线程安全。可指定初始容量，在不需要扩容的场景下效率更高，因为相比于LinkedList其不需要创建对象。

***

## **Arrays**

- asList()：数组转List，返回私有内部类ArrayList，**直接将数组代理为List**，支持set和replace，不支持add和remove。

- hash()：使用除留余数法计算数组哈希值。

- equals()：数组对比，遍历比较每个元素。如果是对象数组，还可以使用Comparator比较。
    > *数组并没有重写equals方法，但集合类的抽象父类重写了equals方法。*

- deepEquals()：等同于Objects的deepEquals方法。对象数组深度对比，如果元素为对象数组，则转换为数组递归深度对比。

- copyOf()、copyOfRange()：数组复制，调用System类的本地静态方法arraycopy实现，为**值复制浅拷贝**。

- sort()：排序数组，如果是基本数据类型使用DualPivotQuicksort，如果是对象则使用ComparableTimSort（Comparable类型）和TimSort（指定Comparator）。
    - DualPivotQuicksort：双指针快速排序，即三向切分快排。
    - **TimSort**：**稳定的、自适应的、迭代的归并排序，融合了归并算法和二分插入排序算法**。如果数组长度小于32则直接使用二分插入排序；否则将数组分为一个个run，每一个run都是连续上升或下降的子区间；确定出一个minrun（16~32），如果run的大小小于该值时则使用插入排序合并；确定好run后会放入栈内，最终使run的数量刚好为2的幂或稍小于2的幂；之后合并run，合并操作会使用尽量小的内存空间和GALLOP模式来加速合并。

- parallelSort()：并行排序数组，只针对int、long、float、double和对象数组，要求ForkJoinPool的并行度（COMMON_PARALLELISM）大于1，对象数组大小要大于8192，基本数据类型数组大小要大于4096。

- parallelPrefix()：**并行的前缀操作**，如数组\[1,1,1,1\]和操作{(left,right) -> left+right}，返回结果\[1,2,3,4\]。

- spliterator()：为数组创建拆分迭代器（特性为有序和不可变），用于创建流。

***

## RandomGenerator

JDK17新增，随机数生成器的公共接口，新增了一系列生成随机数流的方法，以及of(String name)工厂方法（使用SPI的方式加载随机数生成器的实现）。

### Random

普通伪随机数生成器，使用当前seed值计算得到新的seed值并更新，因为seed为AtomicLong类型，故在高并发下效率低，**不应该再使用**。

### **ThreadLocalRandom**

线程隔离的伪随机数生成器，**为单例模式**，不支持设置种子，与ThreadLocal相似，内部是操作当前Thread对象的属性（通过Unsafe）。

#### **使用的Thread属性**

- threadLocalRandomSeed：种子，用于计算伪随机数。
- threadLocalRandomProbe：线程探针哈希值，**用于提供可变的线程哈希值**，非0，使用0表示当前线程ThreadLocalRandom还未初始化。
- threadLocalRandomSecondarySeed：第二种子。

#### **静态方法**

- current()：唯一的静态工厂方法，返回单例对象。如果当前线程对象的threadLocalRandomProbe属性为0则初始化（通过Unsafe设置当前线程对象的threadLocalRandomProbe属性和threadLocalRandomSeed属性）。

- advanceProbe(int probe)：获取下一个线程探针哈希值，**在ConcurrentHashMap的fullAddCount方法和ForkJoinPool的submissionQueue方法中使用探针哈希值来确定线程对应的下标**，如冲突则使用该方法更新哈希值并再次尝试。

- nextSecondarySeed()：获取第二种子值，如果为0则初始化，只在ConcurrentSkipListMap中使用。

#### **实例方法**

- nextSeed()：伪随机算法，使用旧种子计算出新种子并更新种子值，通过Unsafe操作当前线程对象的threadLocalRandomSeed属性。

- setSeed(long seed)：无任何作用，为了兼容父类构造方法，只允许被父类构造方法调用一次，否则抛出UnsupportedOperationException。

### SplittableRandom

可拆分的随机数生成器，用于使用ForkJoinPool和并行流的 场景，通过split方法可以从原实例得到一个新的实例，在流被分割后不同实例之间互不影响。

### SecureRandom

强随机数生成器，用于有安全性要求的场景。

***

## Base64

二进制到文本的编码方式，使用大小64的字符表（数字、字母、+和/）将每六位二进制映射为字符序列，在尾部添加=作为填充字符（每个=表示填充了00）。

- getEncoder()：返回标准Base64编码器，编码后的文本不会包含换行符。
- getUrlEncoder()：返回URL和文件名安全的Base64编码器，相比于getEncoder()，字符表中+字符被替换为-，/被替换为_。
- getMimeEncoder()：返回适用于MIME类型的Base64编码器，相比于getEncoder()，编码后的文本每76个字符后会添加一个换行符。
- *withoutPadding()*：实例方法，返回此编码器的等效编码器，但是不在尾部添加填充字符。

***