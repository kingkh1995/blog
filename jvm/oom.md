## [首页](https://kingkh1995.github.io/blog/)

#### OOM异常种类：

1. java.lang.StackOverflowError
   > 线程栈空间耗尽

1.java.lang.OutOfMemoryError: Java heap space
> 堆内存空间不足

1. java.lang.OutOfMemoryError: GC overhead limit exceeded
   > 垃圾回收上头，进程花费98%以上的时间执行GC，但只恢复了不到2%的内存，且该动作连续重复了5次。
   >> 可能会先触发Java heap space

1. java.lang.OutOfMemoryError-->Metaspace
   > 元空间内存不足，加载的类或方法等太多，元空间位于本地内存，不是虚拟机内存

1. java.lang.OutOfMemoryError: Direct buffer memory
   > 本地内存不足，使用 Direct ByteBuffer 可以直接访问堆外内存，IO操作时就不用在Java堆和本地内存之间来回复制数据

1. java.lang.OutOfMemoryError: unable to create new native thread
   > JVM向操作系统请求创建本地线程失败

1. java.lang.OutOfMemoryError: Requested array size exceeds VM limit
   > 超出了JVM 限制的数组的最大长度

1. java.lang.OutOfMemoryError: Out of swap space
   > 本地交换空间不足，所有可用的虚拟内存已被耗尽，虚拟内存由物理内存和交换空间组成

1. java.lang.OutOfMemoryError：Kill process or sacrifice child
   > 由操作系统触发的，Linux内核允许进程申请的内存总量大于系统可用内存，通过这种“错峰复用”的方式可以更有效的利用系统资源，而当内存不足时，将自动激活OOM Killer，寻找评分低的进程，并将其“杀死”，释放内存资源。
