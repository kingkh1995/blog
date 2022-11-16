# [首页](/blog/)

> version: **jdk17**

***

## ArrayList

动态数组，非线程安全，支持随机访问（实现了RandomAccess），默认容量10，每次扩容为1.5倍。

- subList()：返回私有内部类SubList对象，所有操作全部被代理到ArrayList对象上。

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