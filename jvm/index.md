# [首页](/blog/)

> <<深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）>>

***

## 内存结构

***

## 类加载

***

## OutOfMemoryError

### java.lang.StackOverflowError：线程栈空间耗尽

最常见的就是由无限递归导致（如两个对象循环依赖然后调用toString方法），如确实需要，可以通过JVM启动参数（-Xss）适当增大增加线程栈内存空间。

### Java heap space：堆内存空间不足

最常见的错误。首先需要检测是否出现内存泄漏；然后判断是否是创建了超大对象（或数组），检查该逻辑是否合理；如果是突发流量，则填谷削峰、限流降级；如果确实是业务压力增大，则通过JVM启动参数（-Xmx）调大堆内存，或新增资源。

### GC overhead limit exceeded：垃圾回收上头

进程花费98%以上的时间执行GC，但只恢复了不到2%的内存，且该动作连续重复了5次，**可能会先触发Java heap space**。使用和Java heap space相同的方案排查问题。

### Direct buffer memory：本地内存不足

使用 DirectBuffer（ByteBuffer等）可以直接访问堆外内存以实现高速IO，省去了在Java堆和本地内存之间来回复制数据的消耗。首先也是需要检查是否存在内存泄漏，否则通过启动参数（-XX:MaxDirectMemorySize）增加可分配的堆外内存上限值，或增加服务器内存。

### Unable to create new native thread：JVM向操作系统请求创建本地线程失败

可能是线程数超过了操作系统最大线程数限制，需要检查创建的线程数量是否合理；或本地内存不足，则增加内存即可。

### Metaspace：元空间内存不足

**Metaspace 是方法区在 HotSpot 中的实现，其位于本地内存，不是虚拟机内存**。加载的类数目太多或体积太大导致，运行时会生成大量动态类的应用场景下可能出现，可以调整为不限制元空间大小或增加其初始空间。

### Requested array size exceeds VM limit：超出了JVM限制的数组的最大长度

一般会比Integer.MAX_VALUE略小，一般很难出现，需要检查代码。

### Out of swap space：本地交换空间不足

表示所有可用的虚拟内存已被耗尽（**虚拟内存由物理内存和交换空间组成**），一般很难出现。

### Kill process or sacrifice child：被操作系统‘杀死’

Linux内核允许进程申请的内存总量大于系统可用内存，通过这种“错峰复用”的方式可以更有效的利用系统资源，而当内存不足时，将自动激活OOM Killer，寻找评分低的进程，并将其“杀死”，释放内存资源。升级服务器配置或隔离部署，或优化OOM Killer。

***
