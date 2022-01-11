# [首页](/blog/)

> 高性能缓存

***

## BoundedLocalCache

有界本地缓存，使用了W-TinyLFU算法，为了更好的应对突发稀疏流量设计。

- 使用SLRU（分段的LRU）算法将缓存空间分为Window Cache和Main Cache，Main Cache又分为protected和probation，**probation的数据命中后会晋升到protected**。Window Cache默认占1%空间，Main Cache中的probation默认占20%，使用中会动态调整Window Cache的大小。

- 使用类似布隆过滤器的算法统计缓存的近似频率。

### FrequencySketch

- table：long类型一维数组，长度与缓存maximumSize相关，且为了能快速取模调整为2的幂。**限制了统计数的最大值为15，故一个long储存了16个统计数。**

- sampleSize：保新机制阈值，默认为10倍maximumSize。

- size：计数器，达到sampleSize后，调用reset方法使得所有统计数减半，这就是缓存的保新机制，用于剔除掉过往频率很高的缓存。

- frequency方法：

    利用hash值后两位获取到随机数（0，4，8，12）确定分段的起始位置，依次使用四种不同的哈希函数确定出四个槽位，虽然槽位可能相同，但分段是从起始位置递增的，故避免了获取到重复的值，四个分段上统计值的最小值即为缓存数据的近似频率。

- increment方法：
    
    缓存命中时调用，与frequency方法相同得到四个分段后增加统计数，统计数限制最高为15，只有当统计数成功增加后，计数器才自增，并判断是否需要触发保新机制。

### maintenance(Runnable task)

- 缓存维护方法，读写等操作之后调用，为了同步方法。

- 流程为依次为：清理ReadBuffer、WriteBuffer，执行提交的维护任务，清理keyReferences、valueReferences，缓存过期，缓存淘汰，动态调整缓存空间。

#### ReadBuffer、WriteBuffer

- 由于缓存的读写伴随着一系列且需要同步执行的后续工作，为了提高并发处理能力，读写操作通过ReadBuffer、WriteBuffer记录下操作。

- 读操作会在维护阶段执行后续工作，ReadBuffer写入允许丢失。BoundedBuffer为ReadBuffer默认实现，使用了类似于Striped64（LongAdder）热点分离的方式提高了并发写入效率。

- 写操作则不允许WriteBuffer写入失败，并立即执行后续工作，消费WriteBuffer为同步方法，故WriteBuffer为MPSC队列（适用于多个生产者和一个消费者的场景），故提升了缓存高并发写能力。

#### 缓存淘汰策略

新的数据会直接加入window区，之后在维护阶段判断window区如果超出限制，会使用LRU将淘汰出来的数据candidate放入probation区，如果probation区也超出限制，那么也使用LRU选出要淘汰的数据victim，然后通过FrequencySketch统计的频率决定淘汰哪一个。

#### 缓存过期

expireVariableEntries使用WheelTimer多层时间轮实现，expireAfterAccessEntries和expireAfterWriteEntries是使用AccessOrderDeque双端队列实现。

#### 缓存空间动态调整

动态调整window区和main区的比例以使缓存表现最佳的命中率，只有当总请求量达到了FrequencySketch的sampleSize才进行调整，通过本次命中率与上次命中率的差值决定window区增减的大小，且缓存的整体大小不变。

***