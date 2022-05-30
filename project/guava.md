# [首页](/blog/)

> **Guava**

***

## RateLimiter

- 延迟计算：并不是定时生成令牌，而是在请求的时候计算令牌数量（同步方法）。
- 预消费：如果当前允许获取令牌，则直接获取并返回，令牌数量不足而产生的等待时间，由之后的请求承担。
- 稳定模式（SmoothBursty）：令牌生成速度恒定。
- 渐进模式（SmoothWarmingUp）：令牌生成速度缓慢提升直到维持在一个稳定值。储存令牌数超过阈值后，令牌生成速度逐渐减缓，即令牌消耗的越快则生成越快。

## SmoothRateLimiter

- 属性：
    - storedPermits：当前桶里有多少令牌。
    - maxPermits：桶可以最大存储多少令牌，稳定模式下为一秒内能产生的令牌数。
    - stableIntervalMicros：稳定期生成一个令牌的间隔，单位微秒。
    - nextFreeTicketMicros：下一个请求允许获取到令牌的微秒数。
- 方法：
    - reserveEarliestAvailable：消耗令牌，同步执行，先调用resync，再计算出待补充令牌生成时间，重新设置nextFreeTicketMicros，注意返回的结果为resync之后的nextFreeTicketMicros。
    - resync：补充令牌，判断nextFreeTicketMicros，如果过期则计算并设置storedPermits，并将nextFreeTicketMicros设置为当前时间，即当前请求无需等待可立即获取到令牌。


### *限流算法的对比*

- 漏桶算法：请求先进入桶内，以恒定的速率传输请求数据。适用于系统调用的第三方服务处理能力有限的情况，服务器只是用来调控流量，故突然流量出现时只能拒绝。

- 令牌桶：以一定的速率生成令牌，请求从桶中获取到令牌才能被处理。适用于系统自身处理能力有限的情况，在出现突发流量时，可以一定程度的处理超过限制的请求。

### *RedissonRateLimiter*

使用滑动窗口限流方式，使用String记录可用令牌数，使用Zset记录请求，每次获取令牌时使用Zrangebyscore命令查出滑动窗口外的请求并删除，同时恢复令牌数量。

***

## BloomFilter

以时间换空间，使用了双重散列的探查方式，只做了一次hash计算，将低位视为hash1，高位视为hash2，计算公式为：

    (hash1 + i * hash2) % TABLE_SIZE。

- bits：bit数组，LockFreeBitArray类型，使用了AtomicLongArray和LongAdder。
- numHashFunctions：hash次数，实际上是双重散列的探查次数。

### *RedissonBloomFilter*

基于redis的bitmap数据类型实现，原理与Guava BloomFilter相同。

### *计数布隆过滤器*

改为使用数组统计每个槽位被命中的次数，可以删除元素，只需把计数减1即可，缺点就是空间占用大大增加。

### *布谷鸟哈希*

通过多个哈希函数计算出多个位置，将元素放置到任一位置的空置席位上，如果不存在空置席位则挤走某个元素，被挤走的元素，继续按此方式落位到一个新的空置席位上，直到所有元素均落位成功，或该挤兑行为重复次数达到一定阈值后触发扩容，扩容行为与hashmap相似。

#### *布谷鸟过滤器*

每个位置上设置4个席位，席位上存储的是指纹信息，只使用两个哈希函数，第二个哈希值由指纹和第一个哈希值计算得出。

- 空间性能高于布隆过滤器，因为只会确定两个位置，内存寻址次数少于布隆过滤器。
- 能够删除元素，但是也会存在误判，因为保存的只是指纹信息。

***

## EventBus

***

## Striped

用于并发场景下创建锁对象。

***

## Monitor

对象监视器，用于替代ReentrantLock。

***