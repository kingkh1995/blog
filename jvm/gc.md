## [首页](/blog/)

> 垃圾回收

***

## 垃圾回收算法


***

## G1 GC

- Young GC：【-XX:MaxGCPauseMillis】配置表示每次YGC/MixedGC的期望最长停顿时间，会按此配置执行回收；
- Mixed GC：同时回收新生代加老年代区域，【-XX:InitiatingHeapOccupancyPercent】配置表示当老年代占用比例超过该值后，执行Mixed GC替代Young GC，默认45%；
- Old GC：实际上没有Full GC的概念，因为是基于复制算法，当 Young GC/Mixed GC 执行时无法找到空闲的region时就会失败，**一旦失败就会立刻停止系统程序切换到 G1 之外的 Serial Old GC 来收集整个堆（包括 Young、Old、Metaspace）**，其性能等同于串行收集器。

***

## ZGC

-XX:+UseZGC

***