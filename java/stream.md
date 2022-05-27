# [首页](/blog/)

> version: **jdk17**

***

## Stream

### 实例方法

- limit()：短路有状态中间操作，**截取流中前n个元素**，可终止无限生成流。
    > **如在limit操作前使用了filter操作导致生成的元素个数永远无法满足，则会因为无限生成流无法自行终止而导致一直执行。**

- unordered()：中间操作，**用于消除并行流中保证有序操作的限制，并不是使流无序**。
    > 使用了unordered操作后的并行流，findFirst操作与findAny操作的效果相同。

- dropWhile()：JDK9增加，有状态中间操作，丢弃元素直到元素不满足条件后终止丢弃。

- takeWhile()：JDK9增加，有状态中间操作，获取元素直到元素不满足条件后终止获取，**可用于终止无限流**。

- mapMulti(BiConsumer<? super T, ? super Consumer<R>> mapper)：JDK16增加，中间操作，将一个元素替换为0-n个元素。
    > **委托给flatMap操作实现，使用缓存区保存新元素，并使用Spliterator创建流，开销小于使用直接创建新流。**

    ```java
    // 结果集为 [1,2,2,3,3,3]
    Stream.of(1, 2, 3)
        .mapMulti(
            (value, consummer) -> {
              for (int i = 0; i < value; ++i) {
                // consummer为SpinedBuffer对象
                // 初始缓冲区大小16，每次执行都会将元素放入其缓冲区中。
                consummer.accept(value);
              }
            })
        .toList();
    ```

- toList()：JDK16增加，中断操作，收集为List，使用toArray方法返回的数组创建ArrayList然后包装成unmodifiableList。
    > **Collectors.toList()收集为ArrayList，toList()多了一次数组复制。**

### 静态方法

- builder()：使用Builder模式创建流。

- generate(Supplier<? extends T> s)：无限生成流，使用Supplier生成，**一定要使用终止操作终止**。

- iterate(T seed, Predicate<? super T> hasNext, UnaryOperator<T> next)：生成流，每个元素使用上一个元素生成，如不设置终止条件则只能使用终止操作终止。

***

## IntStream & LongStream

### 静态方法

- range() & rangeClose()：按区块生成元素，相比于iterate支持并行。

### 实例方法

- boxed()：中间操作，返回包装对象的流。

***