# [首页](/blog/)
> 算法基础

*** 

## union-find算法

- quick-find算法

    使用数组保存每一个触点所属连通分量的id。

    - find操作：只需要访问数组一次。
    - union操作：需要遍历数组，修改连通分量下的所有触点。

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

    - union操作：只需要修改当前触点。
    - find操作：需要向上迭代以找到根触点，最坏情况下树退化成了链表。

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
    // 只有两个高度相同的树连接在一起，树的高度才会加1;
    // 利用归纳法可知，组成高度h的树，需要至少2^(h-1)个触点;
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

- 路径压缩的加权quick-union算法

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

## 基础排序算法

### 一、选择排序

最简单的算法，需要~(N^2)/2次比较和N次交换。

- *运行时间与输入无关*
- *数据移动是所有算法中最少的*

### 二、插入排序

每次都将当前元素插入到左侧已排序完成的数组并保证插入之后左侧数组依然有序，最坏情况下（逆序数组）比较次数和交换次数都是~(N^2)/2次。

- *交换的次数等于逆序数量*
- *适合部分有序的数组排序*

```java
public static void sort(int[] arr) {
    for (int i = 1; i < arr.length; i++) {
        for (int j = i; j > 0 && arr[j] < arr[j - 1]; j--) {
            // 出现逆序则交换相邻元素
            ArrayUtils.swap(arr, j, j - 1);
        }
    }
}
```

```java
// 优化解法：循环找到应该所处的位置，并将所有大于当前元素的元素后移一格
public static void sort(int[] arr) {
    for (int i = 1; i < arr.length; i++) {
        int temp = arr[i];
        for (int j = i; j > 0 && temp < arr[j - 1]; j--) {
            arr[j] = arr[j-1];
        }
    }
}
```

### 三、希尔排序

使用插入排序对间隔为h的子序列进行排序，不断减少h直到h为1。

- *对比初级排序算法,数组越大优势越大*
- *使用序列 1、4、13、40、121...平均性能更好*

```java
public static void sort(int[] arr) {
    int length = arr.length;
    int h = 1;
    while (h < length / 3) {
        h = 3 * h + 1;
    }
    for (; h >= 1; h /= 3) {
        // 将数组变为h有序
        for (int i = h; i < length; i++) {
            for (int j = i; j >= h && arr[j] < arr[j - h]; j -= h) {
                ArrayUtils.swap(arr, j, j - h);
            }
        }
    }
}
```

### 四、冒泡排序

从左至右不断交换相邻的逆序元素，每一轮循环都将最大值上浮到右侧，并且如果某一轮循环未发生交换，即代表数组已经有序。

```java
public static void sort(int[] arr) {
    int length = arr.length;
    boolean isSorted = false;
    for (int i = length - 1; !isSorted && i > 0; i--) {
        isSorted = true;
        for (int j = 0; j < i; j++) {
            if(arr[j] > arr[j+1]){
                isSorted = false;
                ArrayUtils.swap(arr, j, j + 1);
            }
        }
    }
}
```

### 五、归并排序

***

## 快速排序

***

## 堆排序

***