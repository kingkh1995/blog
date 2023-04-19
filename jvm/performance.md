# [首页](/blog/)

> 监控&调优

***

## **JVM通用参数**

- -Xms：初始堆大小；
- -Xmx：最大堆大小，一般设置为等于Xms，即不支持堆内存动态调整；
- -Xmn：新生代的内存大小；
- -Xss：每个线程的堆栈大小，默认值为0，表示使用系统默认值，64位系统中默认值为1M；
- -XX:MetaspaceSize：元空间的初始空间大小，达到该值就会触发Full GC进行类型卸载，同时收集器会对该值进行调整；
- -XX:MaxMetaspaceSize：设置元空间最大值，默认值为-1，表示不限制；
- -XX:SoftRefLRUPolicyMSPerMB：表示JVM可以忍受多久软引用不被回收，**这个参数不代表软引用能存在的最大时间**，建议设置为2000 - 5000ms。如果是0则每次都会把软引用回收掉释放内存，使用公式（clock - timestamp <= freespace * SoftRefLRUPolicyMSPerMB）判断每个软引用对象是否可以不回收，clock为上次垃圾回收的时间，timestamp为软引用对象上次被调用的时间，**这个公式表示空闲空间越小能够忍耐的软引用对象空闲的时间会越短**。


***

## **垃圾收集器参数**

|参数|描述|
|:--|:--|
|**UseConcMarkSweepGC**|使用ParNew + CMS + Serial Old的收集器组合|
|SurvivorRatio|新生代中Eden区域与Survivor区域的容量比值，默认值为8，即8:1:1|
|PretenureSizeThreshold|直接进入老年代的对象大小|
|MaxTenuringThreshold|晋升到老年代的对象年龄|
|HandlePromotionFailure|是否允许空间分配担保失败|
|CMSInitiatingOccupancyFration|设置CMS在老年代空间使用多少后触发垃圾收集，默认68%|
|**UseG1GC**|使用G1收集器，JDK9之后服务端模式的默认值|
|G1HeapRegionSize|设置Region大小，非最终值|
|MaxGCPauseMills|设置G1收集过程目标时间，默认200ms，非硬性条件|
|MaxTenuringThreshold|晋升到老年代的对象年龄|
|MaxTenuringThreshold|晋升到老年代的对象年龄|
|G1NewSizePercent|G1新生代最小值，默认5%|
|G1MaxNewSizePercent|G1新生代最大值，默认60%|
|G1MaxNewSizePercent|G1新生代最大值，默认60%|
|G1ReservePercent|G1老年代为分配担保预留的空间比例，默认10%|
|InitiatingHeapOccupancyPercent|设置触发全局并发标记的老年代在Java堆占用率阈值，默认值45%|
|**ParallelGCThreads**|用户线程冻结期间（STW阶段）并行执行的收集器线程数，默认值是CPU核心数|
|**ConGCThreads**|并发整理、并发标记（非STW阶段）的执行线程数，默认值由ParallelGCThread计算得出，且小于其值|

***

## **垃圾收集器日志参数**

**JDK9之后日志相关参数全部归纳到“-Xlog”上，垃圾收集器的标签为gc。**

|描述|JDK9之前参数|JDK9之后参数|
|:--|:--|:--|
|查看GC基本信息|-XX:+PrintGC|-Xlog:gc|
|查看GC详细信息|-XX:+PrintGCDetails|-Xlog:gc*|
|查看GC前后的堆、方法区可用高容量变化|-XX:+PrintHeapAtGC|-Xlog:gc+heap=debug|
|查看GC过程中用户线程并发时间|-XX:+PrintGCApplicationConcurrentTime|-Xlog:safepoint|
|查看GC过程中用户线程停顿时间|-XX:+PrintGCApplicationStoppedTime|-Xlog:safepoint|
|查看收集器Ergonomics机制自动调节的相关信息|-XX:+PrintAdaptiveSizePolicy|-Xlog:gc+ergo*=trace|
|查看熬过收集后剩余对象的年龄分布信息|-XX:+PrintTenuringDistribution|-Xlog:gc+age=trace|

***

## **故障处理工具**

### **jps**

列出正在运行的虚拟机进程，并显示虚拟机执行主类名称，以及这些进程的**本地虚拟机唯一ID**（LVMID）。

```
jps [options] [hostid]
```

|选项|作用|
|:--|:--|
|-q|只输出LMVID，省略主类的名称|
|-m|输出虚拟机进程启动时传递给主类main()函数的参数|
|-l|输出主类的全名，如果进程执行的是JAR包，则输出其路径|
|-v|输出虚拟机进程启动时的JVM参数|

### **jstat**

监视虚拟机各种运行状态信息，可以显示虚拟机进程中的类加载、内存、垃圾收集、即时编译等运行时数据。

```
jstat [ option vmid [interval [count]] ]
# interval和count代表查询间隔和次数，如果省略这两个参数则表示只查询一次。
```

|选项|作用|
|:--|:--|
|-class|监视类加载、卸载数量、总空间、类装载耗时等|
|-compiler|输出即时编译器的行为信息|
|-printcompilation|输出已被即时编译的方法|
|-gc|监视Java堆状况，包括各区容量、已用空间、垃圾收集时间等|
|-gccapacity|与-gc基本相同，但输出主要关注Java堆各个区域使用到的**最大、最小空间**|
|-gcutil|与-gc基本相同，但输出主要关注已使用空间占总空间的**百分比**|
|-gccause|与-gcutil一样，但是会额外输出**导致上一次垃圾收集产生的原因**|
|-gcnew|监视新生代的垃圾收集情况|
|-gcnewcapacity|与-gcnew基本相同，但输出关注使用到的最大、最小空间|
|-gcold|监视老年代的垃圾收集情况|
|-gcoldcapacity|与-gcold基本相同，但输出关注使用到的最大、最小空间|
|-gcmetacapacity|输出元空间使用到的最大、最小空间|

### **jinfo**

实时查看和调整虚拟机各项参数。

```
jinfo [option] pid
```

- -flag \<name\>：查询指定名称的参数；
- -flag [+|-]\<name\>：打开或关闭指定参数；
- -flag \<name\>=\<value\>：设置指定参数；
- -flags：打印所有参数；
- -sysprops：打印虚拟机进程的System.getProperties()的内容。

### **jstack**

用于生成虚拟机当前时刻的线程快照，即每一条线程正在执行的方法堆栈的集合。用于定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间挂起等。

```
jstack [option] vmid
```

|选项|作用|
|:--|:--|
|-F|强制输出线程堆栈|
|-l|额外输出锁的相关信息|
|-m|如果调用到本地方法，可以显示C/C++的堆栈|

### **jmap**

用于生成堆转储快照（dump文件），也可以查询Java堆相关的各类信息。dump文件可以使用jhat、Visual VM等图形化工具进行分析。

```
jmap [option] vmid
```

|选项|作用|
|:--|:--|
|-heap|显示堆详细信息，如使用的收集器、参数配置、分代状况等|
|-histo|显示队中对象统计信息，包括类、实例数量、合计容量|
|-clstats|显示元空间中类加载器的统计信息|
|-dump|生成dump文件|
|-F|强制生成dump快照，不支持live参数|

```
#dump命令格式，live参数表示只dump存活对象，parallel参数表示并行遍历堆的线程数。
-dump:[live,]format=b,file=<fileName>,[parallel=<number>]
```

### **jcmd**（功能整合类终端）

JDK7开始提供，是集成式多功能工具箱。

|基础工具|JCMD|
|:--|:--|
|jsp -lm|jcmd|
|jmap -dump \<pid\>|jcmd \<pid\> GC.heap_dump|
|jmap -histo \<pid\>|jcmd \<pid\> GC.class_histogram|
|jstack \<pid\>|jcmd \<pid\> Thread.print|
|jinfo -sysprops \<pid\>|jcmd \<pid\> VM.system_properties|
|jinfo -flags \<pid\>|jcmd \<pid\> VM.flags|

***

## **调优**

### **收集器选择**

确定应用程序关注的重点，吞吐量、停顿时间、内存占用、基础设施、JDK版本等，选择最合适的垃圾收集器。

### **核心指标**

- jvm.gc.time: 每次GC的耗时，一般应在500ms以内；
- jvm.gc.meantime: 每次YGC的耗时，一般应在50ms以内；
- jvm.fullgc.count: FGC多久发生一次，一般24小时内只应该发生一次；
- jvm.fullgc.time: 每次FGC的耗时，一般应在1s以内。

### **优化案例**

***

## 故障排查

### CPU飙升

1. 使用【top】命令找出占用CPU高的进程；
2. 再使用【top -Hp \[PID\]】命令找到该进程中CPU占用最高的线程；
3. 使用【printf "%x" \<TID\>】将十进制的TID转换为16进制的；
4. 使用【jstack \<PID\> | grep \<TID\>】查看堆栈快照信息或下载到文件中；

常见原因包括：while死循环、频繁创建对象、超多线程调度。

### 内存问题排查

- 内存溢出的情况可以通过配置【-XX:+HeapDumpOnOutOfMemoryError】，java程序在内存溢出时自动输出dump文件。
- 垃圾回收频繁，使用jstat命令查看垃圾回收情况；
- 内存分析，使用jmap或jcmd获取堆dump文件。

***

## 常规优化思路

### 系统优化

确定系统瓶颈，如果是数据库瓶颈，则判断是否需要优化索引、是否需要引入缓存、最后才考虑分库分表；如果确实是硬件瓶颈了则需要考虑机器扩容了；扩容前还得先尝试优化，首先从应用层面优化，快速失败、填谷削峰、算法优化、使用缓存等，然后对JVM调优以提高吞吐量，最后尝试从网络和操作系统排查。

### JVM优化

1. CPU指标观察：top、jps、jstack等
2. JVM内存指标观察：jmap；
3. JVM　GC指标：查看GC日志。
   1. -XX:+PrintGCDetails // GC详情
   2. -XX:+PrintHeapAtGC // GC前后堆栈
4. 制定优化目标：具体指标；
5. 制定优化方案：
6. 对比前后指标：
7. 持续观察跟踪对比效果。


### 优化案例：元空间导致频繁FullGC

1. 查看GC日志：
   > Full GC (Metadata GC Threshold)
2. 定位到内存碎片化现象
   > Metaspace used 30000k, capacity 50000k, committed 50000k....
3. 分析：元空间以chunk进行预分配，**一个类加载器会独占整个chunk**，used与capacity表示出现了碎片化，要判断是否创建了大量类加载器；
4. jmap下载dump文件分析堆对象，发现大量**DelegatingClassLoader委派类加载器**。
   > 反射膨胀机制：反射首先使用JNI的方式获取类信息，如果调用超过一定频次会创建字节码，使用DelegatingClassLoader。
5. 首先调大metaspace的大小尽快恢复，或者通过设置-Dsun.reflect.inflationThreshold=0即永远不适用字节码存取器获取类信息；
6. 定位代码问题，底层对象转换等，大量使用了BeanUtils.copyProperties，触发反射膨胀机制后，NativeMethodAccessorImpl类它针对每个类的每个属性的Getter及Setter方法都会生成一个DelegatingClassLoader，并将这些DelegatingClassLoader缓存提高，全部改成MapStruct。

### 优化案例：线程大小分配太大

线程栈大小没有改设置，默认1M，后面改成了256k。

### 优化案例：大数组创建太多

导出百万条数据，虽然我们是循环操作，拆分为很多文件，查数据阶段也是循环查询，但是全部查出来然后放到List，这就导致创建了大数组，引发FullGC。

***