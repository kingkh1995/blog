## [首页](/blog/)

> 调优及监控

***

## 常用JVM参数

- -Xms：初始堆大小，一般设置为等于Xmx，即不支持堆内存动态调整；
- -Xmx：最大堆大小；
- -Xss：每个线程的堆栈大小，默认值为0，表示使用系统默认值，64位系统中为1M；
- -Xmn：新生代大小，官方推荐配置为整个堆大小的3/8；
- -XX:SurvivorRatio=：新生代中Eden区域与Survivor区域的容量比值，默认值为8，即8:1:1；
- -XX:PermSize：永久代初始大小，已废弃；
- -XX:MaxPermSize：永久代最大值，已废弃；
- -XX:MetaspaceSize：元空间的初始空间大小，达到该值就会触发Full GC进行类型卸载，同时收集器会对该值进行调整；
- -XX:MaxMetaspaceSize：设置元空间最大值，默认是 -1，表示不限制；
- -XX:MaxTenuringThreshold：生存区对象的最大年龄；
- -XX:PretenureSizeThreshold：任何超过该阈值的对象都不会尝试在新生代进行分配而是直接进入老年代；
- -XX:SoftRefLRUPolicyMSPerMB=：能够忍受的软引用存在的最大时间。

***

## 性能调优（G1）

优化的目标就是避免触发Full GC，尽量不要让对象进入老年代，减少Survivor区触发动态年龄判断。

- -XX:MaxGCPauseMillis：修改预期GC暂停事件，太小会跟不上分配内存的速度，太大会使得暂停时间过长。**要结合系统可以接收的请求响应时间和GC的收集时间修改，默认200ms一般不建议修改**。
- -XX:ParallelGCThreads: stw 阶段工作的 GC 线程数，一般设置为 CPU 核心数 -1。
- -XX:ConcGCThreads：非 stw 阶段工作的 GC 线程数，会影响系统的吞吐量，**系统如果是计算密集型建议是 CPU 核数的 1/4 ~ 1/3，iO 密集型建议是 1/2**。
- -XX:G1ReservePercent：G1为老年代预留的空间比例，默认是10%，如果新生代晋升失败会触发 Old GC，**如果经常出现晋升失败场景则应该提高该比例**。
- -XX:MaxMetaspaceSize：元空间最大大小，如果有大量生成动态类的需求则应该提高该值。
- -XX:TraceClassLoading / -XX:TraceClassUnloading：
开启追踪类加载和类卸载，可以在Tomcat的catalina.out 日志文件中查看，用于排查问题。
- -XX:SoftRefLRUPolicyMSPerMB：JVM可以忍受多久软引用不被回收，如果是0则每次都会把软引用回收掉释放内存，**这个参数不代表软引用能存在的最大时间**。需要根据自身需求调整，建议这个参数设置2000 - 5000ms。

***

##  JVM监控命令

### jps

查看java进程及其相关的信息

- -l：输出主类全名或jar路径
- -q：只输出PID
- -m：输出JVM启动时传递给main方法的参数
- -v：输出JVM启动时指定的JVM参数

### jinfo

```
jinfo [option] <pid>
```
用来查看JVM参数和动态修改部分JVM参数。

- -flag \<name\>：打印指定名称的参数
- -flag [+|-]\<name\>：打开或关闭参数
- -flag \<name\>=\<value\>：设置参数
- flags：打印所有JVM参数
- -sysprops：打印所有系统配置

### jstat

```
jstat [option] <pid> [interval] [count]
```
查看JVM运行时的状态信息，包括内存状态、垃圾回收等，interval是打印间隔时间（毫秒），count是打印次数（默认一直打印）。

- -class：类加载行为统计
- -compiler：HotSpt的JIT编译器行为统计
- -printcompilation：HotSpot编译方法统计

#### **GC相关**
- -gc：垃圾回收行为统计，**输出实际的值**；
- -gcnew：新生代行为统计；
- -gcold：年老代行为统计；
- -gcutil：垃圾回收行为统计，**输出使用百分比**；
- -gccause：同-gcutil，并附加最近两次垃圾回收事件的原因；
- -gccapacity：各个垃圾回收代容量和空间统计，包含new、old、meta；
- -gcnewcapacity：新生代与其相应的内存空间的统计；
- -gcoldcapacity：年老代与其相应的内存空间的统计；
- -gcmetacapacity：元空间统计。

### jstack

```
jstack [option] <pid>
```
用来查看JVM线程快照（包含java线程和本地线程）

- -l：额外显示锁信息。

### jmap

```
jmap [option] <pid>
```
生成堆dump文件和查看堆相关的各类信息

- -clstats：打印类加载器统计信息；
- -heap：打印堆的概要信息；
- -histo\[:live\]：打印堆中对象的统计信息，指定live表示只统计存活对象；
- **-dump**：
    ```
    -dump:[live]format=b,file=<fileName>,parallel=<number>
    ```
    - live：如果指定则只包括存活的对象；
    - format：b表示打印为二进制文件；
    - file：文件名
    - parallel：并行遍历堆的线程数，0表示使用默认的线程数。

### jhat

```
jhat [option] [dumpfile]
```
用来分析dump文件，更推荐使用图形化分析工具（自带的jvisualvm、mat、visualVM等）。

- -port \<port\>: HTTP服务器端口，默认是7000，启动一个web服务器用于访问。

### jcmd

```
jcmd <pid> [option]
```
包含了jps、jmap、jstack等的功能

- VM.version：虚拟机版本信息；
- VM.flags：虚拟机参数；
- Thread.print：打印线程堆栈信息；
- GC.class_histogram：类加载行为统计；
- GC.heap_dump \[file\]：堆信息dump；
- VM.native_memory：本地内存追踪，可用于分析内存泄漏。

***

## 故障排查

### CPU飙升

1. 使用【top】命令找出占用CPU高的进程；
2. 再使用【top -Hp \[PID\]】命令找到该进程中CPU占用最高的线程；
3. 使用jstack命令查看堆栈信息或下载到文件中；
4. 因为top命令查看的线程ID是十进制的，而堆栈信息中的线程ID是16进制的，需要转换后在堆栈信息中找到占用最高的线程信息。

### 内存问题排查

- 内存溢出的情况可以通过配置【-XX:+HeapDumpOnOutOfMemoryError】，java程序在内存溢出时自动输出dump文件。
- 垃圾回收频繁，使用jstat命令查看垃圾回收情况；
- 内存分析，使用jmap或jcmd获取堆dump文件。

***

## 常规优化思路

确定系统瓶颈，如果是数据库瓶颈，则判断是否需要优化索引、是否需要引入缓存、最后才考虑分库分表；如果确实是硬件瓶颈了则需要考虑机器扩容了；扩容前还得先尝试优化问，首先从应用层面优化，快速失败、平滑流量、异步处理等，然后从JVM方法层面优化，最后尝试从网络和操作系统排查。

***