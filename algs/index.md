# [首页](/blog/)
> 算法基础

*** 

## union-find算法

- quick-find算法

    使用数组保存每一个触点所属连通分量的id。

    - find操作：只需要访问数组一次。
    - union操作：需要遍历数组，修改连通分量下的所有。触点

    ```java
    public int find(int p) {
		return id[p];
	}

	public void union(int p, int q) {
		int pID = find(p);
		int qID = find(q);
		if (pID == qID) {
			return;
		}
		for (int i = 0; i < id.length; i++) {
			if (id[i] == pID) {
				id[i] = qID;
			}
		}
		count--;
	}
    ```

- quick-union算法

    使用数组保存每个触点所属连通分量的根触点。

    - union操作：只需要修改当前触点即可。
    - find操作：需要向上迭代以找到根触点，最坏情况下触点组成的树可能退化成了链表。

    ```java
    public int find(int p) {
		while (p != id[p]) {
			p = id[p];
		}
		return p;
	}

	public void union(int p, int q) {
		int pRoot = find(p);
		int qRoot = find(q);
		if (pRoot == qRoot) {
			return;
		}
		id[pRoot] = qRoot;
		count--;
	}
    ```

- 加权的quick-union算法

    对比quick-union算法，额外使用数组保存每个根触点的树的大小，以保证union操作只将小树连接到大树上。

    - find操作：使用加权的quick-union算法后，N个触点构造的树的最大深度为logN，故find操作成本增长的数量级为logN。

    ```java
    // find方法同quick-union
    public void union(int p, int q) {
		int i = find(p);
		int j = find(q);
		if (i == j) {
			return;
		}
		if (sz[i] < sz[j]) {
			id[i] = j;
			sz[j] += sz[i];
		} else {
			id[j] = i;
			sz[i] += sz[j];
		}
		count--;
	}
    ```
- ****拓展***：根据高度加权的quick-union算法*

    ```java
    // 只有两个高度相同的树连接在一起，树的高度才会加1
    // 利用归纳法可知，组成高度H的树，需要至少2^(n-1)个点
    // 故N个触点的树高度不会超过logN。
    public void union(int p, int q) {
		int i = find(p);
		int j = find(q);
		if (i == j) {
			return;
		}
		if (sz[i] < sz[j]) {
			id[i] = j;
		} else {
			id[j] = i;
            // 高度相同则加1 否则不变
			sz[i] = Math.max(sz[i], sz[j]+1);
		}
		count--;
	}
    ```

- 路径压缩的加权quick-union算法算法

    在find操作向上迭代过程中，将遇到的所有触点都直接连接到根触点上，理论上可以得到几乎完全扁平化的树，是最优的解法。

    ```java
    // union方法选择quick-union算法或加权quick-union算法
    public int find(int p) {
		int root = p;
		while (root != id[root]) {
			root = id[root];
		}
		while (id[p] != root) {
			int temp = p;
			p = id[p];
			id[temp] = root;
		}
		return root;
	}
    ```

***