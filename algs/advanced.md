# [首页](/blog/)

> 算法进阶

***

## 散列表

两种解决碰撞的方法：拉链法和线性探测法。

- 拉链法：使用链表存储hash值相同的键。
    - 查找：找到key所在的链表，从链表中顺序查找；
    - 插入：找到key所在的链表，加入链表的头部；
    - 删除：找到key所在的链表，从链表中删除。

- 线性探测法：使用空位来解决冲突，保证使用率在1/8至1/2之间。
    - 查找：找到key所处的位置，未命中则线性向前查找，直到遇到空位，则判断为不存在；
    - 插入：找到key所处的位置，如果冲突则线性向前探测到一个空位来存储冲突的键；
    - 删除：删除该键，并顺序将当前键簇剩余所有键全部重新插入。

***

## 二叉查找树（BST）

一棵二叉查找树是一棵二叉树，且每个结点都大于其左子树的任一结点而小于其右子树的任一结点。

删除任意结点的流程：

1. 为叶子结点直接删除；任一子结点为空，删除该结点，并将其父链接改为指向其非空的子结点。
2. 若子结点均不为空，则找到该结点的后续结点，即其右子树的最小结点。
3. 从右子树删除其后续结点，将后续结点的父链接改为指向后续结点的右子树（因为后续结点为树的最小结点，故其左结点必然为空）。
4. 用后续结点替换掉该结点的位置。

***

## 平衡查找树

### **2-3查找树**

一棵2-3查找树或为一棵空树或由以下结点组成：

- 2-结点，标准二叉查找树中的结点，含有一个键和两个链接
- 3-结点，含有两个键和三个链接

#### **2-3查找树的插入操作**

- 如果插入到2-结点上，直接组成一个3-结点即可；
- 如果插入到3-结点上，先组成一个临时4-结点，然后分裂成三个2-结点，将中间的结点移动到父结点中，并继续向上分解，直到不存在临时的4-结点或者到达了根节点。

### **红黑树**

使用标准的二叉查找树（全部为2-结点）和一些额外信息表示一个2-3树，将一个3-结点替换为由一条左斜的红链接相连的两个2-结点，用黑链接表示结点之间的普通链接。

因为2-3树是完美平衡的（任意空链接到根结点的距离都相同），所以红黑树是**完美黑色平衡**的，任意空连接到根结点的路径上黑链接（黑色结点）数量都相同。

最坏情况下操作是2lgN级别（即左边路径全是3-结点），平均情况下所有操作均是lgN级别。

#### **一、红黑树的插入操作**

在沿着插入点到根路径向上移动的每个结点中完成以下操作即可实现插入：

1. 如果左结点为黑色，右子结点为红色，进行左旋操作；

1. 如果左子结点为红色且其左子结点也为红色，进行右旋操作；
    
1. 如果左右子结点均为红色，进行颜色转换。

最后将根结点设为黑色。

![](/blog/pic/红黑树插入算法.png)

#### **二、自顶向下的2-3-4树的插入算法**

> 2-3-4树允许存在4-结点，使用一条左斜的红链接和一条右斜的红链接以及三个2-结点来表示一个4-结点

插入算法要保证在向下查询路径上分解所有4-结点（***通过颜色转换将一个4-结点变为3个2-结点***），插入结点后向上回溯配平4-结点（**通过旋转将不平衡的4-结点变成一条左斜和一条右斜组成的4-结点**）。

相对于红黑树的插入算法，只需要将颜色转换的代码提前至空判断之后递归插入之前即可（***即在插入结点前将4-结点配平，又因为可以存在4-结点所以插入的最后一步不再需要颜色转换***）。

#### **三、红黑树的删除最小值操作**

删除最小值，沿着左链接向下过程中，保证不存在2-结点：
    
- 3个2-结点可以直接合并为一个4-结点；
- 如果兄弟结点不是2-结点，将兄弟结点最小键的移动到父结点中，将父结点的最小键移动到左子结点中；
- 如果兄弟结点也是2-结点，将左子结点、父结点最小键、兄弟结点合并成一个4-结点。

最后从3-结点或4-结点中删除最小值，然后向上回溯分解所有4-结点。

#### **四、红黑树的删除操作**

使用和删除最小值相同的变换保证向下查找路径中不存在2-结点，如果键在最底层直接删除即可，否则和二叉查找树一样，找到其后继结点，删去该结点并替换被删除结点即可。

![](/blog/pic/红黑树删除算法.png)

#### **五、Java红黑树实现**

实现为2-3-4树，且每个结点均包含父结点的引用，同时允许单个右斜红链接存在。

- 插入操作：新插入的结点都为红结点，自底向上配平4-结点，最多需要两次旋转以及多次颜色变换。

    - 找到插入位置，父结点为黑色则直接插入成功，否则进入循环判断，直到父结点为黑色或者到达了根结点；
    - 循环判断开始，如果父结点和叔叔结点均为红色，则直接颜色转换，当前结点设为爷爷结点，开始下一轮循环；
    - 叔叔结点为黑色，则需要旋转爷爷结点，**这之前需要通过旋转将当前结点倾斜方向与父结点调整为一致同时将当前结点与父结点交换**，这个时候当前结点已被调整到爷爷结点的位置，继续下一轮循环；
    - 循环结束后最后把根结点设为黑色。

- 删除操作：最多需要三次旋转以及多次颜色变换。

    - 先查找被删除结点是否存在，不存在则直接结束；
    - 如果存在左右子树，找到后继结点，即右子树的最小值，并用后继结点替换掉当前结点；
    - 接下来删除当前结点，因为当前结点必然是只有一颗子树或者没有子树（**如果使用后继结点替换过了，后继结点为最小值必然没有左子树**），用空或存在的单棵子树替换其位置；
    - 如果被删除的是黑结点则需要对兄弟结点进行调整（**将兄弟子树路径上黑色链接数量减少1**），自底向上直到恢复完美黑色平衡，循环判断直到当前结点不为黑结点。

***

## 图

### 无向图

非稠密无向图使用邻接表来表示，从图中任意一个顶点出发都存在一条路径到达另一个任意顶点，则此图是连通的。

### 有向图

一个顶点的出度为由该顶点指出的边的总数，一个顶点的入度为指向该顶点的边的总数。有向图中由顶点v能到达顶点w并不意味着由顶点w也能到达顶点v。

#### 拓扑排序

给定一幅有向图，将所有顶点排序，使得所有的有向边均从排在前面的元素指向后面的元素。有向无环图才有拓扑排序，**为所有顶点的逆后序排列**。

- Kosaraju算法：

    1. 求得有向图的反向图的逆后序排列；
    2. 按照上一步得到的排列顺序在有向图中进行深度优先搜索。


### 最小生成树

图的生成树是它的一棵含有所有顶点的无环连通子图，而加权无向图的最小生成树是它的一棵权值之和最小的生成树。

- 树的重要性质：

    - 边的总数为顶点数减去1
    - 用一条边连接任意两个顶点会产生一个新的环
    - 从树中删去一条边将会得到两棵独立的树

#### 贪心算法

根据**切分定理**，找到一种切分，它的横切边均不为黑色，则将它的权重最小的横切边标记为黑色，重复，直到标记了V-1条边为止。

##### **切分定理**

一幅加权图中，给定任意切分，该切分的横切边中权重最小者必然属于图的最小生成树。

- 切分：将图的所有顶点分为两个非空且不重叠的集合。
- 横切边：连接属于两个不同集合的顶点的边。

#### Prim算法

每一步都为一棵生长中的树添加一条边，一开始这颗树只有一个顶点，每次都将下一条连接树中的顶点与不在树中的顶点且权重最小的边加入树中，直到树含有V-1条边。

#### Kruskal算法

按边的权重顺序，将边加入最小生成树中，加入的边不与已加入的边构成环，直到树含有V-1条边。

### 加权有向图

***

## 字符串

***