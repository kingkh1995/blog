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
    查找节点，可能会递归调用。

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
    插入红黑树节点或返回要被替换的节点，红黑树内首先使用hash值排序，其次尝试使用compareTo，最后使用tieBreakOrder方法确定顺序。

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

ConcurrentHashMap为线程安全的HashMap，同样是使用拉链法，键和值都不支持为null；相比于JDK8之前的版本，不再使用分段锁，**而是在单个槽位上同步**，不再支持设置loadFactor和concurrencyLevel。

### 属性

```java

transient volatile Node<K,V>[] table; // 当前哈希表，使用Unsafe安全操作。

private transient volatile Node<K,V>[] nextTable; // 新哈希表，扩容过程使用

// 1、创建时为哈希表初始容量，0则使用默认容量16
// 2、初始化时为-1
// 3、扩容时小于-1，高16位为扩容标记，低16位值为辅助扩容的线程数+1（1表示扩容已结束）
// 4、其他阶段为扩容阈值（HashMap的threshold属性），loadFactor固定为0.75
private transient volatile int sizeCtl;

// 并发转移的下一个区段的起点，从右往左移动，这样只需要与0比较即可，0表示所有槽位已被转移完成。
private transient volatile int transferIndex;

// 用于计数，实现同LongAdder。
private transient volatile long baseCount; // 计数基准值
private transient volatile int cellsBusy; // 计数单元初始化及扩容时使用
private transient volatile CounterCell[] counterCells; // 竞争场景计数单元

// 视图，为代理对象。
private transient KeySetView<K,V> keySet;
private transient ValuesView<K,V> values;
private transient EntrySetView<K,V> entrySet;
```

### 静态方法

-   ```java
    static final int HASH_BITS = 0x7fffffff; // 掩码，最高位为0其余位位1，用于获取正数哈希值。

    static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }
    ```
    扰动函数，重新计算哈希值，同HashMap（将高16位与低16位做异或运算），最后使用掩码去除符号位，**因为负数哈希值用于标识节点类型**。

-   ```java
    private static final int RESIZE_STAMP_BITS = 16;

    // n为tableSize，因为均为2的幂次方，故前置0的个数肯定不同（最大32），再将结果的低16位的最高位设为1。
    static final int resizeStamp(int n) {
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
    }
    
    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS; // 16

    // 将低16位左移至高16位，即最高位为1，则结果为负数。
    int rs = resizeStamp(n) << RESIZE_STAMP_SHIFT;
    ```
    用于生成扩容标记（**扩容阶段sizeCtl的高16位**），要求为负数，因为正数用于记录阈值，同时要求不同的tableSize下扩容标记不同。

-   ```java
    // 返回值类型为Set<K>
    public static <K> KeySetView<K,Boolean> newKeySet(int initialCapacity) {
        return new KeySetView<K,Boolean>(new ConcurrentHashMap<K,Boolean>(initialCapacity), Boolean.TRUE);
    }
    ```
    **返回线程安全的HashSet**，为ConcurrentHashMap的视图对象，支持插入操作，值默认为Boolean.TRUE。

### **Nodes**

-   ```java
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val; 
        volatile Node<K,V> next;

        ...

        // Virtualized support for map.get(); overridden in subclasses.
        Node<K,V> find(int h, Object k) {
            Node<K,V> e = this;
            if (k != null) {
                do { // 遍历链表查找
                    K ek;
                    if (e.hash == h && ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                } while ((e = e.next) != null);
            }
            return null;
        }
    }
    ```
    Node：与HashMap.Node的区别是，val和next是volatile修饰的。

-   ```java
    static final int MOVED = -1; // 固定hash

    static final class ForwardingNode<K,V> extends Node<K,V> {
        final Node<K,V>[] nextTable; // 记录新哈希表的地址
        ForwardingNode(Node<K,V>[] tab) {
            super(MOVED, null, null); // 固定hash
            this.nextTable = tab;
        }

        Node<K,V> find(int h, Object k) {
            // loop to avoid arbitrarily deep recursion on forwarding nodes
            outer: for (Node<K,V>[] tab = nextTable;;) { // 在新哈希表中查找
                Node<K,V> e; int n;
                if (k == null || tab == null || (n = tab.length) == 0 || (e = tabAt(tab, (n - 1) & h)) == null)
                    return null;
                for (;;) { // 内层循环，遍历槽位上的Node链表查找。
                    int eh; K ek;
                    if ((eh = e.hash) == h && ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                    if (eh < 0) { // 为特殊节点
                        if (e instanceof ForwardingNode) { // 转移节点，即又一次发生了扩容。
                            tab = ((ForwardingNode<K,V>)e).nextTable; // 跳转到新哈希表
                            continue outer; // 退出内层循环进入外层循环，相当于迭代。
                        }
                        else // 其他则调用对应节点的find方法查找
                            return e.find(h, k);
                    }
                    if ((e = e.next) == null) // 指针后移
                        return null;
                }
            }
        }
    }
    ```
    ForwardingNode：转移节点，扩容中使用，表示旧哈希表内该槽位上的节点已被移动到新哈希表内。

-   ```java
    static final class TreeNode<K,V> extends Node<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;

        TreeNode(int hash, K key, V val, Node<K,V> next,
                 TreeNode<K,V> parent) {
            super(hash, key, val, next);
            this.parent = parent;
        }

        Node<K,V> find(int h, Object k) {
            return findTreeNode(h, k, null); // 查找逻辑完全等同于HashMap
        }
    }
    ```
    TreeNode：红黑树节点，结构及方法与HashMap的TreeNode的完全相同，**因为是由TreeBin去保证其线程安全性**。

-   ```java
    static final int TREEBIN = -2; // 固定hash

    static final class TreeBin<K,V> extends Node<K,V> {
        TreeNode<K,V> root; // 红黑树根节点，非volatile，通过加锁保证原子性。
        volatile TreeNode<K,V> first; // 链表头节点

        // 额外的读写锁机制
        volatile Thread waiter;
        volatile int lockState; // 高30位不全为0表示读锁
        static final int WRITER = 1; // set while holding write lock
        static final int WAITER = 2; // set when waiting for write lock
        static final int READER = 4; // increment value for setting read lock
        ...

        // 构造方法，为TreeNode链表生成TreeBin节点，设置在哈希表槽位上。
        TreeBin(TreeNode<K,V> b) {
            super(TREEBIN, null, null); // 固定hash
            this.first = b; // 设置链表头节点
            TreeNode<K,V> r = null;
            // 遍历TreeNode链表，依次插入节点以构造红黑树，同HashMap
            for (TreeNode<K,V> x = b, next; x != null; x = next) {
                ...
            }
            this.root = r; // 设置红黑树根节点
            // 不同于HashMap，不调整链表结构
            assert checkInvariants(root);
        }

        // 查找，能加读锁则红黑树方式查找，否则作为链表查找。
        final Node<K,V> find(int h, Object k) {
            if (k != null) {
                for (Node<K,V> e = first; e != null; ) {
                    int s; K ek;
                    if (((s = lockState) & (WAITER|WRITER)) != 0) { // 1、如果是WAITER或WRITER状态，则作为链表遍历查找
                        if (e.hash == h && ((ek = e.key) == k || (ek != null && k.equals(ek))))
                            return e;
                        e = e.next;
                    } else if (U.compareAndSetInt(this, LOCKSTATE, s, s + READER)) { // 2、否则加读锁，红黑树查找。
                        TreeNode<K,V> r, p;
                        try {
                            p = ((r = root) == null ? null : r.findTreeNode(h, k, null));
                        } finally {
                            // 3、释放读锁并通知等待的线程waiter
                            ...
                        }
                        return p;
                    }
                }
            }
            return null;
        }

        // 插入或替换
        final TreeNode<K,V> putTreeVal(int h, K k, V v) {
            Class<?> kc = null;
            boolean searched = false;
            for (TreeNode<K,V> p = root;;) {
                int dir, ph; K pk;
                // 同HashMap，找到匹配的节点替换，或者确定到要插入的位置
                ...
                // 如果是插入，需要加写锁后，才允许插入并更新root。
                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    TreeNode<K,V> x, f = first;
                    first = x = new TreeNode<K,V>(h, k, v, f, xp);
                    ...
                    else {
                        lockRoot(); // 阻塞获取写锁
                        try {
                            root = balanceInsertion(root, x); // 插入节点恢复平衡，最后更新root。
                        } finally {
                            unlockRoot(); // 释放写锁
                        }
                    }
                    break;
                }
            }
            assert checkInvariants(root);
            return null;
        }

        ...
    }
    ```
    TreeBin：红黑树代理节点，设置在槽位上，代理红黑树的操作，以保证红黑树操作的线程安全性；有额外的读写锁机制，find操作会加读锁，若无法加读锁则遍历查找，put和remove操作都会加写锁。

-   ```java
    static final int RESERVED = -3;  // 固定hash

    static final class ReservationNode<K,V> extends Node<K,V> {
        ReservationNode() {
            super(RESERVED, null, null);
        }

        Node<K,V> find(int h, Object k) { // 直接返回null
            return null;
        }
    }
    ```
    ReservationNode：保留节点，compute和computeIfAbsent操作中使用，表示该节点已被其他线程占用。

### Map方法

-   ```java
    // Map method
    public int size() { // 返回值转为int
        long n = sumCount();
        return ((n < 0L) ? 0 : (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE : (int)n);
    }

    // ConcurrentHashMap method
    public long mappingCount() {
        long n = sumCount();
        return (n < 0L) ? 0L : n; // ignore transient negative values
    }

    final long sumCount() { // 同LongAdder
        CounterCell[] cs = counterCells;
        long sum = baseCount;
        if (cs != null) {
            for (CounterCell c : cs)
                if (c != null)
                    sum += c.value;
        }
        return sum;
    }
    ```
    计数，同LongAdder计数，**更建议使用mappingCount方法**。

-   ```java
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode()); // 1、重新计算哈希值
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) { // 2、使用&运算快速取余确定槽位（同HashMap）
            if ((eh = e.hash) == h) { // 3、判断哈希槽上的节点
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0) // 4、非Node节点（节点hash属性为负）则调用其find方法查找
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) { // 5、为Node节点则继续遍历链表查找
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
    ```
    查找操作**不需要加锁**，使用与HashMap相同的方式确定槽位并查找。

-   ```java
    public V put(K key, V value) {
        return putVal(key, value, false);
    }

    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException(); // key和value均不能为null
        int hash = spread(key.hashCode());
        int binCount = 0; // 统计链表节点数
        for (Node<K,V>[] tab = table;;) { // 循环尝试直到成功
            Node<K,V> f; int n, i, fh; K fk; V fv;
            if (tab == null || (n = tab.length) == 0) // 1、未初始化table则初始化；
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { // 2、槽位为空直接设置；
                if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                    break;                   // CAS成功则退出循环
            }
            else if ((fh = f.hash) == MOVED) // 3、判断为转移节点则辅助扩容；
                tab = helpTransfer(tab, f);
            else if (onlyIfAbsent // 4、判断槽位上的节点，如果查找命中且不允许替换则直接返回原值；
                     && fh == hash
                     && ((fk = f.key) == key || (fk != null && key.equals(fk)))
                     && (fv = f.val) != null)
                return fv;
            else { // 5、执行插入或替换
                V oldVal = null;
                synchronized (f) { // 1）获取槽上节点的同步锁；
                    if (tabAt(tab, i) == f) { // 2）rechek，防止已被修改；
                        if (fh >= 0) { // 3）hash大于0则为Node节点，插入链表尾部或替换，并统计节点数；
                            ...
                        }
                        else if (f instanceof TreeBin) { // 3）为TreeBin节点，调用putTreeVal方法插入；
                            Node<K,V> p;
                            binCount = 2; // 节点至少为2，一个代理节点和一个根节点
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                                ...
                            }
                        }
                        else if (f instanceof ReservationNode) // 4）为保留节点直接抛出异常
                            throw new IllegalStateException("Recursive update");
                    }
                }
                if (binCount != 0) { // 根据Node链表节点数判断是否需要升级为红黑树（同HashMap）
                    if (binCount >= TREEIFY_THRESHOLD) 
                        treeifyBin(tab, i); // 升级为红黑树，并创建一个TreeBin节点，设置到槽位上。
                    if (oldVal != null) // 替换操作直接return
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount); // 6、插入操作，增肌计数，并判断是否需要扩容或辅助扩容。
        return null;
    }
    ```
    插入或替换，**仅需要对槽位上的节点加同步锁**，并会辅助扩容。

-   ```java
    public V remove(Object key) {
        return replaceNode(key, null, null);
    }

    // 替换节点，remove及replace操作使用，value为null表示移除，cv非null表示需要匹配原值。
    final V replaceNode(Object key, V value, Object cv) {
        int hash = spread(key.hashCode());
        for (Node<K,V>[] tab = table;;) { // 循环尝试直到成功
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0 ||
                (f = tabAt(tab, i = (n - 1) & hash)) == null)
                break; // 1、未初始化或槽为空直接退出
            else if ((fh = f.hash) == MOVED) // 2、辅助扩容后再次进入循环尝试
                tab = helpTransfer(tab, f);
            else { // 3、执行移除或替换
                V oldVal = null;
                boolean validated = false; // 标识
                synchronized (f) { // 1）获取同步锁
                    if (tabAt(tab, i) == f) { // recheck
                        if (fh >= 0) {  // 2）hash大于0则为Node节点，遍历链表查找并移除；
                            validated = true;
                            ...
                        }
                        else if (f instanceof TreeBin) { // 3）为TreeBin节点，调用putTreeVal方法；
                            validated = true;
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> r, p;
                            if ((r = t.root) != null &&
                                (p = r.findTreeNode(hash, key, null)) != null) { // 调用TreeNode的findTreeNode查找到节点
                                V pv = p.val;
                                if (cv == null || cv == pv || (pv != null && cv.equals(pv))) {
                                    oldVal = pv;
                                    if (value != null) // value非null则替换
                                        p.val = value;
                                    else if (t.removeTreeNode(p)) // value为null则移除
                                        // removeTreeNode方法返回true表示红黑树需要退化链表
                                        setTabAt(tab, i, untreeify(t.first));
                                }
                            }
                        }
                        else if (f instanceof ReservationNode) // 4）为保留节点直接抛出异常
                            throw new IllegalStateException("Recursive update");
                    }
                }
                if (validated) { // 4、移除操作则计数减1
                    if (oldVal != null) {
                        if (value == null)
                            addCount(-1L, -1);
                        return oldVal;
                    }
                    break;
                }
            }
        }
        return null;
    }
    ```
    移除操作也**仅需要对槽位上的节点加同步锁**，也会辅助扩容。

-   ```java
    public void clear() {
        long delta = 0L; // negative number of deletions
        int i = 0;
        Node<K,V>[] tab = table;
        while (tab != null && i < tab.length) {
            int fh;
            Node<K,V> f = tabAt(tab, i);
            if (f == null) // 槽为空指针后移
                ++i;
            else if ((fh = f.hash) == MOVED) {
                tab = helpTransfer(tab, f); // 先辅助扩容
                i = 0; // 扩容完成后重置指针，此时已经替换为新的哈希表，需要重新清理。
            }
            else {
                synchronized (f) { // 获取同步锁
                    if (tabAt(tab, i) == f) { // recheck
                        // 将节点作为链表遍历统计节点数
                        ...
                        setTabAt(tab, i++, null); // 移除槽上节点
                    }
                }
            }
        }
        if (delta != 0L) // 更新计数
            addCount(delta, -1);
    }
    ```
    清空操作，依次获取每个槽位的同步锁并移除节点，**如在扩容中会先辅助扩容完成**。

-   ```java
    public V compute(K key,
                     BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
        ...
        for (Node<K,V>[] tab = table;;) { // 循环尝试直到成功
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0) // 初始化table
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & h)) == null) { // 槽为空
                Node<K,V> r = new ReservationNode<K,V>(); // 创建保留节点
                synchronized (r) { // 获取同步锁
                    if (casTabAt(tab, i, null, r)) { // CAS设置保留节点到槽上，目的是阻塞其他操作该槽的线程。
                        binCount = 1;
                        Node<K,V> node = null;
                        try {
                            if ((val = remappingFunction.apply(key, null)) != null) { // 计算
                                delta = 1;
                                node = new Node<K,V>(h, key, val); // 使用计算的结果创建新节点
                            }
                        } finally {
                            setTabAt(tab, i, node); // 无论计算是否成功，最终都会移除保留节点。
                        }
                    }
                }
                if (binCount != 0) // 插入成功则退出循环
                    break;
            }
            // 槽位非空时，等同于put操作。
            ...
        }
        ...
        return val;
    }
    ```
    原子计算操作compute和computeIfAbsent在槽为空时，会先在槽位上设置ReservationNode节点，用来阻塞其他操作该槽的线程。

    ```java
    // foreach
    public void forEach(long parallelismThreshold, BiConsumer<? super K,? super V> action) {
        if (action == null) throw new NullPointerException();
        new ForEachMappingTask<K,V>(null, batchFor(parallelismThreshold), 0, 0, table, action).invoke();
    }

    // search，查找直到searchFunction返回非null。
    public <U> U search(long parallelismThreshold, BiFunction<? super K, ? super V, ? extends U> searchFunction) { ... }

    // reduce，整合全部为一个结果
    public <U> U reduce(long parallelismThreshold,
                        BiFunction<? super K, ? super V, ? extends U> transformer,
                        BiFunction<? super U, ? super U, ? extends U> reducer) { ... }

    ...

    // 计算并发线程数
    final int batchFor(long b) {
        long n;
        if (b == Long.MAX_VALUE || (n = sumCount()) <= 1L || n < b)
            return 0;
        int sp = ForkJoinPool.getCommonPoolParallelism() << 2; // COMMON_PARALLELISM / 4
        return (b <= 0L || (n /= b) >= sp) ? sp : (int)n;
    }
    ```
    JDK8新增的**批量操作API**，parallelismThreshold设置为1则可以开启最大并行执行，为Long.MAX_VALUE则单线程执行。

### 初始化 & 扩容

-   ```java
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) { // 循环直到table初始化完成
            if ((sc = sizeCtl) < 0) // sizeCtl小于即正在初始化或扩容中
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) { // 非负则执行初始化，先CAS设置sizeCtl为-1。
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY; // 等于0时使用默认容量16
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n]; // 创建数组
                        table = tab = nt; // 设置值
                        sc = n - (n >>> 2); // 计算阈值，loadFacotor固定为0.75
                    }
                } finally {
                    sizeCtl = sc; // 更新sizeCtl为阈值
                }
                break;
            }
        }
        return tab;
    }
    ```
    初始化哈希表，**初始化过程中设置sizeCtl为-1**，初始化完成后设置sizeCtl为容量阈值。

-   ```java
    private static final int MIN_TRANSFER_STRIDE = 16; // 转移区段最小长度

    // nextTab为null表示启动扩容，非null则协助扩容
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        // 计算转移区段长度
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        if (nextTab == null) { // 启动扩容，则初始化新哈希表，为原哈希表长度的两倍。
            ...
            transferIndex = n; // 初始值为原哈希表长度
        }
        int nextn = nextTab.length;
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab); // 创建转移节点
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) { // 指针左移或抢占下一个区段
                int nextIndex, nextBound;
                if (--i >= bound || finishing) // i指针从区段右端往左端移动
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) { // 区段已被抢占完成
                    i = -1; // -1表示退出执行
                    advance = false;
                }
                // CAS修改transferIndex值，减少一个区段长度，成功则表示抢占到该区段。
                else if (U.compareAndSetInt(this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ? nextIndex - stride : 0))) {
                    bound = nextBound; // 设置为区段终点
                    i = nextIndex - 1; // 设置为区段起点
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) { 
                int sc;
                if (finishing) { // 当前线程为最后一个结束的线程
                    nextTable = null; 
                    table = nextTab; // 新哈希表替换旧哈希表
                    sizeCtl = (n << 1) - (n >>> 1); // 设置为容量阈值
                    return; // 结束
                }
                if (U.compareAndSetInt(this, SIZECTL, sc = sizeCtl, sc - 1)) { // CAS设置线程数减一
                    // 判断当前线程是否是最后一个结束的线程
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT) // sc是修改前的值，sizeCtl低16等于2表示只剩一个线程。
                        return; // 非最后一个线程直接退出
                    finishing = advance = true; // 修改标记，当前线程为最后一个扩容中的线程。
                    i = n; // 即让最后一个线程重新扫描一遍原哈希表
                }
            }
            else if ((f = tabAt(tab, i)) == null) // 当前槽位为空，直接设置转移节点。
                advance = casTabAt(tab, i, null, fwd); // CAS成功则可以advance
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                synchronized (f) { // 获取同步锁
                    if (tabAt(tab, i) == f) { // recheck
                        // 1、为普通Node链表，则拆分为两条
                        // 2、为TreeBin则将红黑树链表拆为两条，统计节点数后，进行升级或者退化
                        // 3、为ReservationNode，则抛出异常
                        // 4、为ForwardingNode，进入下一轮循环后跳过当前槽
                        ...
                    }
                }
            }
        }
    }
    ```
    addCount、tryPresize、helpTransfer三个函数会触发扩容操作，允许多个线程一起扩容，将哈希表分为多个区块，**线程每次抢占一个区段后才执行转移槽位**，每个槽位处理完成后，在原哈希表槽位上设置一个转移节点。

-   ```java
    // check: if <0, don't check resize, if <= 1 only check if uncontended
    private final void addCount(long x, int check) {
        // 计数统计，同LongAdder代码，使用baseCount和cs。
        CounterCell[] cs; long b, s;
        ...
        if (check >= 0) { // check resize
            Node<K,V>[] tab, nt; int n, sc;
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) { // 循环尝试
                int rs = resizeStamp(n) << RESIZE_STAMP_SHIFT; // 计算出扩容标记
                if (sc < 0) { // 为负数表示正在扩容则辅助扩容
                    if (sc == rs + MAX_RESIZERS ||  // 扩容线程数超过阈值，因为只使用低16位记录线程数
                        sc == rs + 1 || // 低16位为1，表示扩容已经完成
                        (nt = nextTable) == null || // 新哈希表还未创建，即还没开始扩容。
                        transferIndex <= 0) // 扩容区段已经被分配完成，更多的线程无法再参与扩容
                        break;
                    if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1)) // 允许协助扩容，CAS更新扩容线程数加一
                        transfer(tab, nt); // 协助扩容
                } // 非负数表示超过了阈值则需要扩容
                else if (U.compareAndSetInt(this, SIZECTL, sc, rs + 2)) // sizeCtl低16位设置为2，即一个线程正在扩容
                    transfer(tab, null); // 启动扩容
                s = sumCount(); // 更新计数，下轮循环重新判断条件
            }
        }
    }
    ```
    用于更新计数，更新完成后判断是否需要扩容。

-   ```java
    // 协助扩容
    final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) { // 要求线程是从转移节点进入
            int rs = resizeStamp(tab.length) << RESIZE_STAMP_SHIFT; // 计算扩容标记
            while (nextTab == nextTable && table == tab && (sc = sizeCtl) < 0) { // recheck
                if (sc == rs + MAX_RESIZERS || // 扩容线程数不能超过阈值，因为只使用低16位记录线程数
                    sc == rs + 1 || // 低16位为1，扩容已经完成
                    transferIndex <= 0) // 扩容区段已经被分配完成，更多的线程无法再参与扩容
                    break;
                if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1)) { // 允许协助扩容，CAS更新扩容线程数加一
                    transfer(tab, nextTab); // 开始协助扩容
                    break;
                }
            }
            return nextTab; // 如发生了扩容则返回新哈希表
        }
        return table; // 状态不正确，返回原哈希表
    }
    ```
    协助扩容，put、remove、clear等写操作遇到转移节点后触发。

-   ```java
    private final void tryPresize(int size) {
        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
            tableSizeFor(size + (size >>> 1) + 1); // 要求最低容量能满足1.5倍给定大小
        int sc;
        while ((sc = sizeCtl) >= 0) { // 仅正常状态
            Node<K,V>[] tab = table; int n;
            if (tab == null || (n = tab.length) == 0) { // 未初始化则直接按计算的最低容量初始化
                ...
            }
            else if (c <= sc || n >= MAXIMUM_CAPACITY) // 当前哈希表容量已得到保证则退出
                break;
            else if (tab == table) { // recheck
                // 扩容一次，扩容完成后sizeCtl变为正数，则会再次进入循环，如果容量未满足则会再次扩容。
                int rs = resizeStamp(n);
                if (U.compareAndSetInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null); // 启动扩容
            }
        }
    }
    ```
    保证哈希表容量足够，putAll方法使用，**会一直尝试扩容直到容量达到要求**。

***

## **问题**

### **tableSize为什么需要是2的幂次方？**

1. hashcode % tableSize == hashcode & (tableSize - 1)，哈希槽的确定可以用一次&运算来替代取余运算；
2. 扩容时哈希槽上的链表只需要被拆分为两条，一条设置到当前位置i，另一条设置到i + tableSize位置。
    
### **链表的长度可能大于8吗？**

可能！**哈希表长度小于64时并不会将链表升级为红黑树而是扩容一次**，而扩容后链表虽然会被拆分为两条，但长度仍然可能大于8。

### **红黑树的节点数可能小于6个吗？**

可能！**只有拆分红黑树时才是通过统计结点个数来判断是否需要退化**，节点移除操作是通过特定的树型进行判断（红黑树的特性决定其可以通过树型判断节点数是否可能过小），也可能导致某些节点数小于6个的树型不会触发退化；**同时使用迭代器移除时也不会触发红黑树退化（为了不破坏迭代过程）**。

### **ConcurrentHashMap效率高在哪里？**

1. 相比于分段锁，加锁粒度更小，从使用ReentrantLock锁多个槽变为使用同步锁锁单个槽；
2. 支持辅助扩容，且不阻塞get操作。

### **不加锁的读操作为什么是线程安全的？**

1. 如果是普通Node节点则直接遍历查找；**如果查找中发生了升级**，因为红黑树升级并不会破坏原链表顺序故不会造成影响；如果发生了插入，插入是尾插法不会影响；如果发生了移除，移除并不会修改next指针，遍历仍可以继续；如果发生了替换，只会更新节点的值，不会造成影响。
2. 如果是扩容节点，会通过转移节点进入最新的哈希表内查找，由于已经设置了转移节点，即该槽位已经转移完成，故不会影响查询操作。
3. 如果是TreeBin节点，通过其读写锁机制保证。
4. 如果是保留节点，表示当前计算还未完成，且当前槽之前为空，直接返回null。

### **保留节点的问题？**

如果计算操作为阻塞型操作，则会导致其他线程因为等待该保留节点的同步锁而阻塞，**因为put等其他操作是先获取到同步锁之后才判断节点是否是保留节点，所以无法做到快速失败**。

### **TreeBin为什么还需要额外的加锁机制？**

因为读操作是不会对槽加锁的，但红黑树写操作会改变红黑树结构，为了读写操作的互不影响，需要额外的读写锁机制。

***

[ConcurrentHashMap解析](https://zhuanlan.zhihu.com/p/257027309)

***