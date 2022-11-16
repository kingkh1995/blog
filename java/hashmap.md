# [首页](/blog/)

> **HashMap & ConcurrentHashMap**

***

## class **HashMap**\<K,V\> extends AbstractMap\<K,V\> implements Map\<K,V\>, Cloneable, Serializable

### 属性

```java
transient Node<K,V>[] table; // Node数组，默认大小16，需要为2的幂次方。

transient int size; // 大小

final float loadFactor; // 装载因子，允许自定义，默认为0.75。

int threshold; // 容量阈值，为tableSize * loadFactor。

transient int modCount; // 并发Fail-Fast机制

transient Set<Map.Entry<K,V>> entrySet; // EntrySet代理对象缓存
```

### 静态方法

-   ```java
    // 支持key为null，哈希值为0。
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
    ```
    重新计算哈希值，将高16位与低16位做异或运算，**因为高16位的随机性更强**。

-   ```java
    // 如果键的类型实现了Comparable，则返回其Class对象，否则返回null。
    static Class<?> comparableClassFor(Object x) { ... }

    // 转为Comparable并使用compareTo方法比较
    static int compareComparables(Class<?> kc, Object k, Object x) {
        return (x == null || x.getClass() != kc ? 0 : ((Comparable)k).compareTo(x));
    }
    ```
    用于比较Comparable类型的键。

-   ```java
    static final int tableSizeFor(int cap) {
        int n = -1 >>> Integer.numberOfLeadingZeros(cap - 1);
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
    ```
    使用给定的容量计算tableSize，**保证值为2的幂次方**。

- tieBreakOrder(Object a, Object b)：用于TreeNode，当hash属性相同且equals方法返回false时，在判断非Comparable类型或compareTo方法比较相等后调用，先比较className，如果相同或者某个参数为null，则使用System.identityHashCode()获取默认哈希值进行比较。**该方法不会返回0，且只具有一致性（即对象没有发生变化则结果不会变化），只能在插入节点时使用，因为不具备对称性则查找节点时无法使用，如果无法比较则只能同时在左子树和右子树中查找**。

### 实例方法

-   ```java
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        // 1、计算tableSize，并创建新table；
        // 初始化：使用threshold，创建时如指定了initialCapacity，则其值为tableSize；未指定则为0，使用默认大小16；
        // 扩容：容量扩大一倍。
        ...
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        // 2、扩容则执行转移。
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) { // 处理原table的每一个槽
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null; // for GC
                    if (e.next == null) // 只有一个节点直接移动
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode) // 拆分红黑树
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // 将Node链表拆分为两条，使用尾插法。
                        Node<K,V> loHead = null, loTail = null; // lo链表设置到原位置i
                        Node<K,V> hiHead = null, hiTail = null; // hi链表设置到i + oldCap位置
                        ...
                    }
                }
            }
        }
        return newTab;
    }

    // TreeNode
    final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
        TreeNode<K,V> b = this;
        // 同Node链表，将TreeNode链表也拆分为两条
        TreeNode<K,V> loHead = null, loTail = null;
        TreeNode<K,V> hiHead = null, hiTail = null;
        int lc = 0, hc = 0; // 拆分过程中统计链表长度
        ...
        // 处理TreeNode链表并设置槽上
        if (loHead != null) {
            if (lc <= UNTREEIFY_THRESHOLD) // 小于等于6则转化为Node链表
                tab[index] = loHead.untreeify(map);
            else { // 大于6则转化为红黑树
                tab[index] = loHead;
                if (hiHead != null) // (else is already treeified)
                    loHead.treeify(tab);
            }
        }
        if (hiHead != null) {
            ...
        }
    }
    ```
    扩容，也用于初始化table。

### **Node操作**

-   ```java
    final Node<K,V> getNode(Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n, hash; K k;
        // 1、使用&运算快速取余确定键的槽位；
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & (hash = hash(key))]) != null) {
            // 2、首先判断哈希槽上的节点，即链表头节点；
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            // 3、红黑树查找或链表遍历查找。
            if ((e = first.next) != null) {
                if (first instanceof TreeNode) // 为红黑树
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do { // 否则为普通链表，则遍历查找。
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
    ```
    get操作，确定槽位后在哈希桶内查找，区分链表或红黑树场景。


-   ```java
    static final int TREEIFY_THRESHOLD = 8; // 链表升级红黑树阈值

    final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length; // 懒加载，首次put时使用resize方法初始化table。
        // 1、确定槽位，如果为空，则直接插入；
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k; // e为待替换的节点，为空则表示是插入。
            // 2、槽位非空，首先判断头节点；
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode) // 3、执行红黑树插入；
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else { // 3、执行链表插入；
                for (int binCount = 0; ; ++binCount) { // 链表遍历过程统计节点个数
                    if ((e = p.next) == null) { // 未查找到节点，插入链表尾部。
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // 链表长度达到红黑树升级阈值
                            treeifyBin(tab, hash); // 链表升级红黑树（并不一定会升级）
                        break;
                    }
                    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                        break; // 查找到节点则执行替换操作
                    p = e;
                }
            }
            // 4、判断是否是替换操作；
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null) // 允许替换操作才替换
                    e.value = value;
                afterNodeAccess(e); // 钩子方法，供LinkedHashMap使用。
                return oldValue; // 返回旧值
            }
        }
        // 5、插入操作则size增加。
        ++modCount; // 替换操作modCount不增加
        if (++size > threshold) // 超过阈值则扩容
            resize();
        afterNodeInsertion(evict); // 钩子方法，供LinkedHashMap使用。
        return null;
    }

    static final int MIN_TREEIFY_CAPACITY = 64; // 最小升级红黑树容量

    // 链表升级红黑树
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        // 容量小于64时并不升级为红黑树，而是执行一次扩容。
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            // 将Node链表替换为TreeNode链表
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                ...
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab); // 调用头节点的treeify方法，将链表结构转化为红黑树结构。
        }
    }
    ```
    **put操作，value允许为null，如果替换则返回原值，插入则返回null。**

-   ```java
    final Node<K,V> removeNode(int hash, Object key, Object value, boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            // 同getNode操作，查找到要移除的节点；
            ...
            if (node != null && (!matchValue || (v = node.value) == value || // matchValue表示仅值匹配时才移除
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode) // 红黑树移除
                    // movable为false时，不允许移动其他节点，用于迭代器移除，防止因移动其他节点而破坏迭代过程。
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p) // 链表移除
                    tab[index] = node.next;
                else
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node); // 钩子方法，供LinkedHashMap使用。
                return node; // 返回被移除的节点
            }
        }
        return null; // 未查找到则返回null
    }
    ```
    remove操作，movable参数用于迭代器移除，**因为红黑树移除操作默认会改变链表中节点的位置**，而在迭代器是将红黑树作为链表进行遍历，故迭代器移除时不能移动红黑树中节点在链表中的位置。

### final class **TreeNode**\<K,V\> extends LinkedHashMap.Entry\<K,V\>

红黑树，为2-3-4树，叶子节点可以为4节点，包含parent、left、right、prev以及next（Node）指针。

#### 实例方法

-   ```java
    final TreeNode<K,V> getTreeNode(int h, Object k) {
        // 找到根节点，并调用其find方法查找。
        return ((parent != null) ? root() : this).find(h, k, null);
    }

    final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
        TreeNode<K,V> p = this;
        do {
            int ph, dir; K pk;
            TreeNode<K,V> pl = p.left, pr = p.right, q;
            // 1、先比较hash值
            if ((ph = p.hash) > h)
                p = pl;
            else if (ph < h)
                p = pr;
            // 2、hash值相同则使用equals比较
            else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                return p;
            // 3、equals比较不等则继续在子树中查找
            // 4、某一子树为空则直接在另外非空的子树中查找
            else if (pl == null)
                p = pr;
            else if (pr == null)
                p = pl;
            // 5、子树均非空，再尝试进行比较
            else if ((kc != null ||
                        (kc = comparableClassFor(k)) != null) &&
                        (dir = compareComparables(kc, k, pk)) != 0)
                p = (dir < 0) ? pl : pr;
            // 6、无法比较或比较结果为0，即无法确定顺序，则只能在左右子树内都查找
            else if ((q = pr.find(h, k, kc)) != null) // 右子树递归调用find方法
                return q;
            else // 左子树在当前find方法内迭代
                p = pl;
        } while (p != null);
        return null;
    }
    ```
    查找节点，红黑树内首先使用hash值排序，其次尝试使用compareTo，最后使用tieBreakOrder方法确定顺序。

-   ```java
    final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab, int h, K k, V v) {
        Class<?> kc = null;
        boolean searched = false;
        TreeNode<K,V> root = (parent != null) ? root() : this; // 使用parent指针确定到根节点；
        for (TreeNode<K,V> p = root;;) { // 确定插入的位置
            int dir, ph; K pk;
            if ((ph = p.hash) > h) // 1、先比较hash值
                dir = -1;
            else if (ph < h)
                dir = 1;
            else if ((pk = p.key) == k || (k != null && k.equals(pk))) // 2、判断是否equals
                return p;
            // 3、尝试进行比较以确定顺序
            else if ((kc == null &&
                        (kc = comparableClassFor(k)) == null) ||
                        (dir = compareComparables(kc, k, pk)) == 0) {
                // 4、无法确定顺序则使用find方法在左右子树中查找节点
                if (!searched) {
                    TreeNode<K,V> q, ch;
                    searched = true;
                    if (((ch = p.left) != null && (q = ch.find(h, k, kc)) != null) ||
                        ((ch = p.right) != null && (q = ch.find(h, k, kc)) != null))
                        return q; // 查找到节点则返回结果
                }
                // 5、无法查找到节点，则使用tieBreakOrder方法确定出一个插入顺序（非0结果）
                // 由于tieBreakOrder方法并不具有对称性，故不能用于查找时确定顺序，只能在左右子树中都查找节点。
                dir = tieBreakOrder(k, pk);
            }
            TreeNode<K,V> xp = p;
            // 6、确定了插入位置后插入节点
            if ((p = (dir <= 0) ? p.left : p.right) == null) {
                ...
                // 7、插入节点并平衡，最后调整链表。
                moveRootToFront(tab, balanceInsertion(root, x));
                return null;
            }
        }
    }
    ```
    插入红黑树节点或返回要被替换的节点。

    ```java
    final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab, boolean movable) {
        int n;
        if (tab == null || (n = tab.length) == 0)
            return;
        int index = (n - 1) & hash;
        TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
        // 先从链表结构中移除节点
        ...
        if (first == null) // 链表已经为空了，直接return。
            return;
        if (root.parent != null) // 确定红黑树根节点
            root = root.root();
        // 通过树结构判断红黑树节点数是否过少（节点数可能为0-10），如果过少则退化为Node链表即可。
        if (root == null
            || (movable
                && (root.right == null
                    || (rl = root.left) == null
                    || rl.left == null))) {
            tab[index] = first.untreeify(map);  // too small
            return;
        }
        // 不允许调整链表结构或者不需要退化为链表时从红黑树中移除节点
        TreeNode<K,V> p = this, pl = left, pr = right, replacement;
        // 找到当前节点的后继节点（右子树的最小节点），并使用其替换当前节点的位置。
        ...
        // 使用后继节点替换当前节点
        if (replacement != p) {
            TreeNode<K,V> pp = replacement.parent = p.parent;
            if (pp == null)
                (root = replacement).red = false;
            else if (p == pp.left)
                pp.left = replacement;
            else
                pp.right = replacement;
            p.left = p.right = p.parent = null;
        }
        // 从右子树中删除后继节点并恢复平衡
        TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);
        ...
        // 调整链表结构
        if (movable)
            moveRootToFront(tab, r);
    }
    ```
    移除当前节点，先从链表中移除，再从红黑树中移除，movable为false时不会调整链表顺序。

-   ```java
    // 将TreeNode链表转化为红黑树结构
    final void treeify(Node<K,V>[] tab) {
        TreeNode<K,V> root = null;
        // 遍历TreeNode链表
        for (TreeNode<K,V> x = this, next; x != null; x = next) {
            next = (TreeNode<K,V>)x.next;
            x.left = x.right = null;
            // 依次插入每一个节点到红黑树，且不修改链表指针。
            ...
        }
        // 调整根节点为头节点
        moveRootToFront(tab, root);
    }

    // 将红黑树结构转化为Node链表
    final Node<K,V> untreeify(HashMap<K,V> map) {
        Node<K,V> hd = null, tl = null;
        // 遍历TreeNode链表
        for (Node<K,V> q = this; q != null; q = q.next) {
            // 将TreeNode替换为Node
            Node<K,V> p = map.replacementNode(q, null);
            ...
        }
        return hd;
    }
    ```
    链表和红黑树结构的互相转化。

#### 静态方法

-   ```java
    static int tieBreakOrder(Object a, Object b) {
        int d;
        if (a == null || b == null ||
            (d = a.getClass().getName().
                compareTo(b.getClass().getName())) == 0)
            // 不具有对称性，因为identityHashCode()方法的返回值仍然有概率是相同的。
            d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
                    -1 : 1);
        return d;
    }
    ```
    当两个键hash相同、equals返回false、且无法比较或比较相同时，确定出一个比较结果，不会返回0。

    ```java
    static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {
        int n;
        if (root != null && tab != null && (n = tab.length) > 0) {
            int index = (n - 1) & root.hash;
            TreeNode<K,V> first = (TreeNode<K,V>)tab[index];
            if (root != first) { // 根节点非链表头节点则将根节点移动到链表头部
                
                ...
            }
            assert checkInvariants(root);
        }
    }
    ```
    将红黑树的根节点调整为链表的头节点，因为TreeNode继承自Node，即红黑树同时也是链表；**但红黑树的根节点并不一定是链表的头节点**，因为迭代器移除时为了不修改链表顺序并没有调用调整方法。

***

## class **ConcurrentHashMap**\<K,V\> extends AbstractMap\<K,V\> implements ConcurrentMap\<K,V\>, Serializable

ConcurrentHashMap为线程安全的HashMap，同样是使用拉链法，键和值都不支持为null，不支持设置loadFactor。

### 属性

- volatile Node<K,V>[] nextTable：用于扩容过程中保存新的数组地址。

- volatile int sizeCtl：-1则表示正在初始化；非-1的负数表示正在执行扩容操作，其中高16位为扩容标记，低十六位为当前正在执行扩容操作的线程数+1；如果还未初始化则为需要初始化的数组大小，为0时则使用默认大小16；初始化完成后则为容量（固定为0.75数组大小，等同于HashMap的threshold属性）。

- volatile int transferIndex：扩容处理指针，记录下一个待转移的位置。

- volatile long baseCount：用于统计元素，没有竞争时使用。

- volatile CounterCell[] counterCells：用于统计元素，存在竞争时使用，相关代码改编自LongAdder和Striped64。

### 静态方法

- int spread(int h)：计算哈希值，与HashMap的hash方法逻辑相同，但会将符号为置为0以使得结果为非负数，目的是为了避免与MOVED、TREEBIN、RESERVED这三个特定的负数哈希值冲突。**因为是直接使用key.hashcode()作为参数，也意味着不支持键为null。**

- tabAt()、casTabAt()、setTabAt()：使用Unsafe类对Node数组进行读写操作。

- newKeySet()：JDK8新增，用于创建线程安全的Set，返回值为KeySetView<K,Boolean>类型，支持添加操作（添加时值默认为Boolean.TRUE）。

### 方法

- size() & mappingCount()：都是调用sumCount方法，size方法需要将long转为int，**建议使用mappingCount方法**。

- sumCount()：统计元素，返回long值，累加baseCount属性和counterCells数组的统计值。

- **jdk8新增带parallelismThreshold参数的foreach（全部执行操作）、reduce（全部整合为一个结果）、search（执行操作直到searchFunction返回非空的结果）**，parallelismThreshold参数表示并发阈值，如果预估大小（sumCount）小于该值则会单线程操作，否则使用ForkJoinPool并行操作。
    > 故parallelismThreshold参数为1则可以开启最大并行执行，为Long.MAX_VALUE则顺序执行。

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

扩容转发节点，表示当前位置的节点已被转移到扩容后的新数组中，hash属性默认为MOVED（-1），持有新数组的引用。

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

保留加锁节点，表示当前位置已被compute或computeIfAbsent操作预留，hash属性默认为RESERVED（-3）。

- find方法：返回null。

***

## **问题**

### **tableSize为什么需要是2的幂次方？**

1. hashcode % tableSize == hashcode & (tableSize - 1)，哈希槽位的确定可以用一次&运算来替代取余运算；
2. 扩容时哈希槽上的链表只需要被拆分为两条，一条设置到当前位置i，另一条设置到i + tableSize位置。
    
### **链表的长度可能大于8吗？**

可能！数组大小小于64时，链表长度大于8并不会升级为红黑树而是触发扩容，扩容后链表虽然会被拆分为两条，但长度仍然可能大于8。

### **红黑树的节点数可能小于6个吗？**

可能！移除红黑树节点时，并不是通过统计结点个数来判断是否需要退化，而是通过判断是特定的树型（**由于红黑树的特性可以通过树型判断节点数是否可能过小**），这就导致节点数小于6个的特定树型不会触发退化；其次使用迭代器移除元素时也不会触发红黑树退化操作。

### **使用时安全吗？**

需要谨慎使用compute和computeIfAbsent操作，可能会创建ReservationNode节点，会导致其他操作抛出IllegalStateException，为方法API声明之外的异常。

### **为什么效率高？**

- 支持的并发级别即为数组大小，相比于使用segment粒度更小了，从使用ReentrantLock锁多个槽变为使用同步锁锁单个槽。
- 扩容操作支持并发协助，且不阻塞操作，get操作直接通过ForwardingNode跳转到新数组中，put操作会先尝试协助扩容再跳转到新数组中。

***