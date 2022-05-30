# [首页](/blog/)

> timer实现

***

## **~~Timer (jdk)~~**

*不建议使用。*

***

## **ScheduledThreadPoolExecutor (jdk)**

实现基于ThreadPoolExecutor，使用的阻塞队列为DelayedWorkQueue，任务类型为ScheduledFutureTask。

任务提交均直接加入队列，并确保存在运行中的worker，保证了任务一定是被worker从队列中取出执行。

### DelayedWorkQueue

与DealyQueue基本一致，是基于延迟时间排序的二叉堆，并通过ReentrantLock和Condition阻塞worker直到任务能被执行；不同是由于ScheduledFutureTask中记录了其在堆中的索引，故在移除任务时不再需要遍历查找，提高了效率。

***

## **HashedWheelTimer (netty)**

单层时间轮，不支持周期性任务。

- tickDuration：每格的时间，最小为1ms。

- ticksPerWheel：每轮的格数，会调整为2的幂，原因同hashmap是为了快速获取索引位置。

- maxPendingTimeouts：最大等待任务数。

- mask：掩码，为(2^n)-1，使用&运算可以快速得到mod值。

- wheel: HashedWheelTimeout链表数组。

- workerThread：工作线程，每个HashedWheelTimer只有一个。

- taskExecutor：任务执行线程池，默认为ImmediateExecutor，直接在当前线程执行，即交由workerThread执行。

- leakDetector：内存泄漏检查器。

#### *newTimeout方法*

1. pendingTimeoutsCount自增，如果当前等待任务数已超出最大限制则抛出异常；
2. 调用start方法启动工作线程，为幂等方法，即HashedWheelTimer不需要显示启动；
3. 计算剩余延时时间并创建HashedWheelTimeout，添加到队列中，并没有添加到时间轮中。

### HashedWheelTimeout

- deadline：执行时间减去工作线程开始时间。
- remainingRounds：剩余轮数。
- next、prev：前后节点的指针，在格子中以链表方式存储。
- cancel：取消任务，不移除，先加入待取消队列中。
- remove：移除任务，在工作线程中被调用，移除同时pendingTimeouts减1。
- expire：到期后执行任务，提交给任务执行线程池执行。

### Worker

工作线程流程，启动时获取当前纳秒数，记录工作线程启动的时间，之后以每格的时间为速率执行流程：

1. processCancelledTasks：处理取消的任务，从待取消任务队列中取出任务，并执行其remove方法。

2. transferTimeoutsToBuckets：将等待任务队列中的任务取出并加入时间轮中，任务直接添加到格子所属链表的尾端，限制了每次最多处理100000个任务。

3. HashedWheelBucket.expireTimeouts：当前格执行任务，遍历链表，任务到期则移除并执行其expire方法，否则轮数减1。

***

## **TimerWheel (caffeine)**

多层时间轮，BoundedLocalCache如果设置了expireAfter，会创建一个WheelTimer对象用于移除过期键。储存为Sentinal对象（Node链表头节点）二维数组，上层时间轮的一格等于下一层整个时间轮的跨度。

## advance(long currentTimeNanos)

通过传入的currentTimeNanos和上次执行时的nanos，计算出每层时间轮需要处理的格子区间，对区间里的所有节点调用BoundedLocalCache的evictEntry方法做过期移除，如未过期则重新加入时间轮。

***

## **延迟队列VS时间轮算法**

时间轮适合处理大量小任务，因为延迟队列使用了堆，调整堆的时间复杂度为O(logn)，而时间轮算法为O(1)，但空间消耗大。

***