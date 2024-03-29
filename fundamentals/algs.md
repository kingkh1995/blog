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
        // sz[]以该触点为根触点的连通分量的触点总数
		if (sz[i] < sz[j]) {
			id[i] = j;
			sz[j] += sz[i];
		} else {
			id[j] = i;
			sz[i] += sz[j];
		}
		count--; // 连通分量的总数
	}
    ```

- ***拓展***：根据高度加权的quick-union算法*

    ```java
    // 只有两个高度相同的树连接在一起，树的高度才会加1
    // 故利用归纳法可知：组成高度h的树，需要至少2^(h-1)个触点
    // 即N个触点的树高度不会超过logN。
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
        // 找到根触点
		int root = p;
		while (root != id[root]) {
			root = id[root];
		}
        // 重新迭代修改路径上所有触点
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

最简单的算法，每次遍历选出最小元素按顺序放到左侧，需要~(N^2)/2次比较和~N次交换。

- *运行时间与输入无关。*
- *数据移动是所有算法中最少的。*

```java
public static void sort(int[] arr) {
    int length = arr.length;
    for (int i = 0; i < length; ++i) {
        int min = i;
        for (int j = i + 1; j < length; ++j) {
            if (arr[j] < arr[min]) {
                min = j;
            }
        }
        swap(arr, i, min);
    }
}
```

### 二、冒泡排序

从左至右不断交换相邻的逆序元素，每一轮循环都将最大值上浮到右侧，并且如果某一轮循环未发生交换，即代表数组已经有序。

```java
public static void sort(int[] arr) {
    int length = arr.length;
    boolean isSorted = false;
    for (int i = length - 1; !isSorted && i > 0; --i) {
        isSorted = true;
        for (int j = 0; j < i; ++j) {
            // 与后一个元素进行比较，如果是逆序则交换。
            if (arr[j] > arr[j + 1]) {
                isSorted = false;
                swap(arr, j, j + 1);
            }
        }
    }
}
```

### 三、插入排序

每次都将当前元素插入到左侧已排序完成的数组并保证插入之后左侧数组依然有序，最坏情况下（逆序数组）比较次数和交换次数都是~(N^2)/2次。

- *交换的次数等于逆序数量；*
- *适合部分有序的数组排序；*
- *三种初级排序算法中平均性能最好。*

```java
public static void sort(int[] arr) {
    // 从第二个元素开始，将第一个元素视为有序数组
    for (int i = 1; i < arr.length; ++i) {
        for (int j = i; j > 0 && arr[j - 1] > arr[j]; --j) {
            // 与前一个元素进行比较，如果是逆序则交换。
            swap(arr, j - 1, j);
        }
    }
}
```

```java
// 优化解法：遍历找到应该目标位置，并将所有大于当前元素的元素后移一格。
public static void sort(int[] arr) {
    for (int i = 1; i < arr.length; ++i) {
        int j = i, temp = arr[j];
        for (; j > 0 && arr[j - 1] > temp; --j) {
            arr[j] = arr[j - 1];
        }
        arr[j] = temp;
    }
}
```

### 四、希尔排序

使用插入排序对间隔为h的子序列进行排序，不断减少h直到h为1。

- *对比初级排序算法，在大规模数组下优势很大，且是不稳定的。*
- *使用序列 1、4、13、40、121...平均性能更好。*

```java
public static void sort(int[] arr) {
    int length = arr.length;
    int h = 1;
    while (h < length / 3) {
        h = 3 * h + 1;
    }
    // 每趟排序类似于插入排序，直到间隔逐渐缩小直到为1，则等同于插入排序。
    for (; h >= 1; h /= 3) { 
        // 将数组变为h有序
        for (int i = h; i < length; ++i) {
            for (int j = i; j >= h && arr[j] < arr[j - h]; j -= h) {
                swap(arr, j, j - h);
            }
        }
    }
}
```

### 五、归并排序

使用分治思想，对一个数组，递归的将它分成两半分别排序，然后将结果归并。

- *能保证**对任意长度为N的数组排序**所需时间都和NlogN成正比。*
- *需要的额外空间也和N成正比。*

```java
// merge方法 左数组：[lo, mid] 右数组：[mid + 1, hi]
// 可以针对merge优化，如果arr[mid] <= arr[mid + 1]，则不需要merge了。
void merge(int[] arr, int lo, int mid, int hi)

// 自顶向下的归并排序（递归实现）
private void sort(int[] arr, int lo, int hi) {
    if (hi <= lo) {
        return;
    }
    int mid = lo + ((hi - lo) >> 1);
    sort(arr, lo, mid);
    sort(arr, mid + 1, hi);
	merge(arr, lo, mid, hi);
}

// 自底向上的归并排序（迭代实现）
private void sort(int[] arr) {
    int n = arr.length;
	// 区间大小从1开始 每次乘以2
    for (int sz = 1; sz < n; sz <<= 1) {
		// 循环两两合并
        for (int lo = 0; lo + sz < n; lo += sz + sz) {
            merge(arr, lo, lo + sz - 1, Math.min(lo + sz + sz - 1, n - 1));
        }
    }
}
```

***

## 快速排序

一种分治的排序算法，递归的将数组分为两个子数组，将两部分独立的排序。

- *最好的情况是每次切分都将元素对半分，时间复杂度为NlogN级别。*
- *最坏的情况就是每次选择最小的元素进行切分，这种情况下需要N^2/2次比较。*

### 1、基本算法

```java
private static void sort(int[] arr, int lo, int hi) {
    if (hi <= lo) {
        return;
    }
    int j = partition(arr, lo, hi);
    sort(arr, lo, j - 1);
    sort(arr, j + 1, hi);
}

// 双指针
private static int partition1(int[] arr, int lo, int hi) {
    // 默认将lo位置的元素选作切分元素
    int pivot = arr[lo];
    int i = lo, j = hi + 1;
    while (true) {
        // 从左边开始找到第一个大于等于pivot的元素
        while (arr[++i] < pivot) {
            if (i == hi) {
                break;
            }
        }
        // 从右边开始找到第一个小于等于pivot的元素 
        while (arr[--j] > pivot) {
            // 可以去掉，因为pivot默认为lo的值，必然会在lo处终止循环
            if (j == lo) {
                break;
            }
        }
        if (i >= j) {
            break;
        }
        // 如果循环未结束则交换i和j
        swap(arr, i, j);
    }
    // 循环终止后则交换j和切分元素
    // 因为循环最后一趟未交换i和j，所以j最终的值必然是小于等于切分元素
    swap(arr, lo, j);
    return j;
}

// 单指针
private static int partition2(int[] arr, int lo, int hi) {
    int pivot = arr[lo], ge = hi + 1; // [ge, hi]区间为大于等于pivot的元素
    // 从右边开始找到所有大于pivot的元素然后交换到[ge, hi]区间
    for (int i = hi; i > lo; --i) {
      if (arr[i] > pivot) {
        swap(arr, i, --ge);
      }
    }
    swap(arr, lo, --ge); // 最后移动pivot
    return ge;
}
```

### 2、算法改进

- 切换到插入排序
> 对于小数组，快速排序比插入排序慢，在排序小数组时使用插入排序。

- 三取样切分
> 使用中位数作为切分值效率最高，但是代价过高，为了找到比较好的切分值，折中方式是取三个元素并将大小居中的元素作为切分值。

- 三向切分
> 对于有大量重复元素的数组，将数组切分为三部分，分别是小于、等于和大于切分元素的数组元素，**对有大量重复元素的随机数组排序时间复杂度是线性的**。

```java
// 单指针三向切分
private static void sort(int[] arr, int lo, int hi) {
    if (hi <= lo){
        return;
    }
    int pivot = arr[lo];
    int lt = lo, i = lo + 1, gt = hi;
    // 未排序区间：[i, gt]
    while (i <= gt) {
        int cmp = arr[i] - pivot;
        if (cmp < 0) {
            // 小于则lt右移，i右移
            swap(arr, i++, lt++);
        } else if (cmp > 0) {
            // 大于则gt左移
            swap(arr, i, gt--);
        } else {
            // 等于则i右移
            i++;
        }
    } 
    // 小于：[lo, lt-1] 等于：[lt, gt]  大于：[gt+1, hi]
    sort(arr, lo, lt - 1);
    sort(arr, gt + 1, hi);
}
```

***

## 堆

优先队列的实现，最大堆的每个结点总是小于等于其父结点，最小堆的每个结点总是大于等于其父结点。

### 二叉堆

二叉堆是一颗完全二叉树，所以可以使用数组来表示，根结点索引为1（不使用0）；任意一个结点索引为k，其子结点为2k和2k+1，其父结点为k/2。

**插入元素和删除堆顶元素时间复杂度都是对数级别。**

- 插入元素：将新元素放到最尾端，让其上浮到合适的的位置。
- 删除堆顶元素：删去最顶端的元素，并将最尾端的元素放到最顶端，让其下沉到合适的位置。

### 堆排序

使用堆的性质，先使用所有元素构建一个最大堆，再重复调用删除最大元素操作，使元素按顺序删去，达到排序的目的。

- *下沉排序过程和选择排序相似，只是比较的次数少了一些。*

```java
// 下沉排序，不需要上浮操作
public static void sort(int[] arr) {
    // 构建最大堆，从最后一个非叶子结点开始往堆顶顺序使用sink方法
    int n = arr.length - 1;
    // 如果一个结点的子结点已经是堆了，那么在该结点上使用sink操作也会变成一个堆
    for (int k = n / 2; k >= 1; k--) {
        sink(arr, k, n);
    }
    // 开始排序，将堆顶的最大元素删除
    // 按顺序将后面的元素和堆顶元素交换位置并下沉
    while (n > 1) {
        swap(arr, 1, n--);
        sink(arr, 1, n);
    }
}

// 在1-n区间元素构成的最大堆中将k位置的元素下沉
private static void sink(int[] arr, int k, int n) {
    while (k << 1 <= n) {
        int j = k << 1;
        // 与子结点中大的结点比较
        if (j < n && arr[j + 1] > arr[j]) {
            j++;
        }
        if (arr[k] >= arr[j]) {
            break;
        }
        swap(arr, j, k);
        k = j;
    }
}
```

***