# [首页](/blog/)

> 监控&调优

***

## **JVM通用参数**

- -Xms：初始堆大小，会在启动时申请该大小的内存并保留，并不是实际会占用；
- -Xmx：最大堆大小，一般设置为等于Xms，即不支持堆内存动态调整；
- -Xmn：新生代的内存大小；
- -Xss：每个线程的堆栈大小，默认值为0，表示使用系统默认值，64位系统中默认值为1M；
- -XX:MetaspaceSize：元空间的初始空间大小，达到该值就会触发Full GC进行类型卸载，同时收集器会对该值进行调整；
- -XX:MaxMetaspaceSize：设置元空间最大值，默认值为-1，表示不限制；
- -XX:SoftRefLRUPolicyMSPerMB：表示虚拟机可以忍受多久软引用不被回收，**这个参数不代表软引用能存在的最大时间**。建议设置为2000 - 5000ms，如果是0则每次都会把软引用回收掉，使用公式（clock - timestamp <= freespace * SoftRefLRUPolicyMSPerMB）判断每个软引用对象是否可以不回收，clock为上次垃圾回收的时间，timestamp为软引用对象上次被调用的时间，**这个公式表示空闲空间越小能够忍耐的软引用对象空闲的时间会越短**。

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
- -flag \[+\|-\]\<name\>：打开或关闭指定参数；
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
-dump:[live,]format=b,file=<fileName>,[parallel=<number>]
#dump命令格式，live参数表示只dump存活对象，parallel参数表示并行遍历堆的线程数。
```

### **jhat**

jdk内置工具（JDK9后移除），用于分析堆转储文件，执行jhat命令会启动一个web服务器，默认端口为7000。

```
jhat [option] <file>
```
- -J\<flag\>：设置server的启动参数，如-J-Xmx512m，设置堆大小用于分析较大的dump文件。

### **jvisualvm**

Java自带的JVM监控工具（JDK14后移除），也可用于分析堆转储文件。

### **jcmd**

JDK7开始提供，是集成式的多功能工具箱。

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

#### 大内存硬件上的文档服务

- 原因：因为文件序列化产生的大对象都是直接进入老年代，只能通过Full GC进行清理，当内存增大后，反而造成停顿的时间更长。
- 方案一：不分代收集，使用Shenandoah或ZGC；
- 方案二：建立逻辑集群来利用硬件资源，减少每个Java虚拟机使用的内存，使用注重延迟时间的CMS收集器。

#### 定时大文件数据分析导致停顿时间变长

- 原因：定时加载超大的数据文件到内存中，导致内存占用过大频繁触发Minor GC，由于新生代的大对象仍然存活，复制算法导致垃圾收集暂停时间明显变长。
- 方案：直接去掉Survivor空间（副作用也会很大），这样大对象经过一次Minor GC后会直接进入老年代，不会导致后续的Minor GC停顿时间变长。

#### safepoint导致长时间停顿

- 分析过程：
  1. 先查看GC日志，real值符合预期，而user、sys值较大，表示是因为用户线程停顿花费了太多时间；
  2. 使用-XX:+PrintSafepointStatistics和-XX:PrintSafepointStatisticsCount=1查看安全点日志，查看垃圾收集线程自旋时间（spin值），显示花费大量时间自旋等待全部用户线程进入安全点；
  3. 添加-XX:+SafepointTimeout和-XX:SafepointTimeoutDelay=2000参数，找出进入安全点超时的线程，并进行优化。
- 方案：**可数循环（循环索引使用int或更小的类型）默认不会设置安全点**，虽然循环次数较少但是单次循环执行时间较长，导致可数循环也可能耗费很长时间，**将循环索引的数据类型改为long即可**。

### 反射膨胀机制导致频繁Full GC

- 原因：由于使用大量使用了BeanUtils，当触发反射膨胀机制后，每个类的每个属性的Getter及Setter方法都会生成对应的GeneratedMethodAccessorXXX类，导致元空间内存不足，并频繁触发Full GC。
- 方案：牺牲一定的性能，关闭反射膨胀机制，或者改成非反射的对象处理工具类。

- **反射膨胀机制**：Java虚拟机有两种方法获取被反射的类的信息，默认会使用JNI存取器，当使用JNI存取器访问同一个类超过一定次数（通过参数-Dsun.reflect.inflationThreshold设置，默认15），会改为使用字节码存取器，会生成代理类GeneratedMethodAccessorXXX，这些类是通过DelegatingClassLoader加载。

***

## **CPU飙升排查**

常见原因包括：while死循环、频繁创建对象、超多线程调度、外部系统命令调用。

1. 使用【top】命令找出占用CPU高的进程；
2. 使用【top -Hp \[PID\]】命令找到该进程中CPU占用最高的线程；
3. 使用【printf "%x" \<TID\>】将十进制的TID转换为16进制；
4. 使用【jstack \<PID\> \| grep \<TID\>】查看堆栈快照信息。

***

## **系统优化**

通过链路追踪工具确定系统的瓶颈所在，如果是数据库瓶颈，则判断是否需要优化索引、是否需要引入缓存、是否需要分库分表；如果确实是硬件瓶颈了则需要考虑机器扩容；扩容前还得先尝试优化，首先从应用层面优化，快速失败、填谷削峰、算法优化、使用缓存等，然后对JVM调优以提高吞吐量，最后尝试从网络和操作系统排查。

***