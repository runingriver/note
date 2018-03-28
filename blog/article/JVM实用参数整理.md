---
title: JVM实用参数整理
date: 2017/03/08 11:12:22
toc: false
list_number: false
categories:
- Java
tags:
- JVM
---

**前言：**在配置和优化Java Web应用中经常要去搜索各个启动配置参数，比较麻烦，且少有看见总结很全的，而最近在看《深入理解Java虚拟机》这本书，正好配合日常所需好好地整理下。


## 一，说明
JVM参数分为标准参数（以`-XX:`开头）和非标准参数（`java -X`命令查看），他们之间格式不一样。
标准参数格式如下：
`-XX:+<option>` 开启option参数
`-XX:-<option>` 关闭option参数
`-XX:<option>=<value>` 设置option参数值
其他的为非标准参数,默认参数可以通过：java -XX:+PrintFlagsInitial命令查看。
分代叫法对比：
年轻代：`Young Generation`，也称新生代`New Generation` 
年老代：`Old Generation`，也称终身代`Tenured Generation`
永久代：`Perm Generation`

Tip：在参考过程中发现参数小错误，因为下面均纯手打，如存在错误，麻烦指出，谢谢！

## 二，内存分配相关参数

| 参数名 | 简述 | 详解 |
| :--------- | :------ | :------ |
| `-Xms` | JVM启动初始堆大小，如：`-Xms4096m`，`-Xms4g` | JVM启动，应用初始堆内存大小。启动后堆的大小调整受`MinHeapFreeRatio`和`MaxHeapFreeRatio`控制，最大不超过`-Xmx`，最小不小于`-Xms`。 |
| `-Xmx` | JVM最大堆大小，如：`-Xmx4096m`，`-Xmx4g`  | Java程序最大堆内存，默认物理内存的1/4，不设置时默认等于`-Xms` |
| `-Xmn` | 年轻代大小，当需将对象留在Eden区设置  | 年轻代 = `eden+ 2 survivor space`，整个堆大小=`年轻代大小(Young Generation)` + `年老代大小(Old Generation)` + `持久代大小(permanent Generation)`。增大年轻代后,将会减小年老代大小.此值对系统性能影响较大,Sun官方推荐配置为整个堆的3/8  |
| `-Xss` | 每个线程栈大小 | JDK5.0后默认1M，以前是256K |
| `MinHeapFreeRatio` | 当堆可用内存小于此值时扩展堆大小 | 当堆空余内存小于整个堆大小的MinHeapFreeRatio（默认40%）时，JVM会增加堆大小，默认值40，当-Xmx 和-Xms 相等时参数无效。 |
| `MaxHeapFreeRatio` | 当堆空闲内存大于此值是缩小堆大小 | 当堆空余内存大于整个堆大小的MaxHeapFreeRatio（默认70%）时，JVM会缩小堆大小，默认值70，当-Xmx 和-Xms 相等时参数无效。 |
| `-XX:NewSize` | 年轻代初始大小 | 通常我们会设置一个初始大小，如`-XX:NewSize=256m` |
| `-XX:MaxNewSize` | 年轻代最大值 |  |
| `-XX:PermSize` | 永久代初始大小 | 通常根据程序会设置一个初始大小，如：`-XX:PermSize=256m` |
| `-XX:MaxPermSize` | 永久代最大值 | 不设置时默认等于`-XX:PermSize`，大部分情况默认64M，经测试内存8G，perm是82M |
| `-XX:NewRatio` | Young与Old的大小比例 | 默认值为2，即：`Young:Old=1:2`，Young占整个堆的1/3大小，Old占2/3，这只是一个理论值表示他们能达到的最大大小，计算中这里省略了Perm代，因为二者实际占用物理内存大小是随对象创建，销毁，晋升而改变，当可用内存为0时二者内存也不一定能达到这个阈值。 |
| `-XX:SurvivorRatio` | Eden与Survivor的比例 | 默认8，即：`2 Survivor：Eden=2:8`，From或To Survivor占整个Young的1/10，这是理论最大值，并非实际大小 |
| `-XX:TargetSurvivorRatio` | 设置Survivor空间使用率 | 默认50%，`-XX:TargetSurvivorRatio=90`设定当Survivor空间使用到达90%时，将Survivor中部分对象送人Old，是否还是在当相同年龄对象(A)总大小大于等于Survivor空间一半时，将年龄大于等于A的所有对象送入Old，有待考证！ |
| `-XX:MaxTenuringThreshold` | 经过多少次Young GC后晋升到年老代 | 默认值15，所有新建对象默认在Eden区分配（容纳不下的对象先Minor GC然后再分配担保机制在Old分配，大对象也有可能直接进入Old，如果启动了本地线程私有分配缓冲区（TLAB Eden区中）可能会在TLAB上分配），经过第一次Minor GC，存活的对象会移到From Survivor，年龄+1，多次GC后存活对象会移到Old区。15是一个最大值，当Survivor区中相同年龄的所有对象大小的总和大于Survivor空间的一半时，年龄大于或等于该年龄的对象可以直接进入老年代；Minor GC后Survivor无法容纳Eden的所有存活对象时，这些无法放入Survivor的对象会直接进入Old。 |
| `-XX:PretenureSizeThreshold` | 对象大小超过该值直接在Old分配 | Parallel Scavenge收集器无效，通常那种很长的字符串数组大小超过该值会直接在Old分配，设置可以避免大量的Eden Survivor内存复制。默认0，不开启。 |
| `-XX:+UseAdaptiveSizePolicy` | 动态调整各区大小 | 默认开启，动态调整Java堆中各个区域的大小及进入老年代的年龄。 |

Tip：空间分配担保
当新创建一个对象时Eden空间不够，触发一次Minor GC，如果Eden中仍有大量对象存活，以致Survivor无法容纳这些对象，此时会开启空间分配担保，将无法容纳的对象移到Old区，如果Old区的空间无法容纳这些对象，会进行一次Full GC，若仍无法容纳，会触发`HandlePromotionFailure`失败，失败后会再发起一次Full GC。Jdk6 update24后`HandlePromotionFailure`失败不再生效，如果空间不足，直接Full GC！

Tip：配置`-XX:NewSize=32m -XX:MaxNewSize=512m -XX:NewRatio=3`Young空间怎么分配？
Young中多个区域都存在绝对值和相对值的参数，当二者同时存在时绝对值起作用，如上，JVM会尝试为Young分配1/4的堆大小，但不会小于32MB和大于521MB。

- - -

## 三，GC收集器参数
### 垃圾收集器介绍
| 参数名| 详解 |
| :--------- | :------ |
| `Serial收集器` | 单CPU环境简单高效，适用桌面程序。 |
| `ParNew收集器` | 是Serial的多线程版，多核下比Serial更能有效利用cpu资源，只能和CMS配合使用！ |
| `Parallel Scavenge` 收集器 | 可控制的吞吐量（吞吐量=运行用户代码时间/(运行用户代码时间+垃圾回收时间)），更高效地利用CPU，适合运算多交互少场景。 |
| `Serial Old收集器` | 单线程的老年代的标记-整理算法的实现。 |
| `Parallel Old 收集器` | 是Parallel Scavenge 的老年代实现，多线程！ |
| `CMS收集器` | 最短回收停留，更快的服务响应速度，多用于B/S模式！缺点：由于采用与用户线程并发收集策略，可能出现卡顿，可能产生浮动垃圾，基于标记-清除的算法实现，导致Old区碎片，大对象分配找不到空间发生Full GC。 注：默认使用Old空间占68%就开始回收，可以设置内存整理策略`UseCMSCompactAtFullCollection` FullGC时进行内存整理和`CMSFullGCsBeforeCompaction`多少次FullGC后进行一次内存整理。默认情况，每次FullGC都进行一次内存整理。 |
| `G1收集器` | JDK7开始支持，不再分代，而是分Region的思想，现商用不多。 |
### 垃圾收集器组合
![jvm组合](/images/garbage.jpg)

注：连线的才能一起使用！**年轻代都是用`复制算法`，老年代都使用`标记-整理算法`**
### 垃圾收集器选择
| 参数名 | 收集器 | 解释 |
| ------- | --------- | ------ |
| `-XX:+UseSerialGC` | `Serial +Serial Old`串行收集器 | client模式的默认收集器，打开后会默认使用`Serial +Serial Old`组合回收  |
| `-XX:+UseParallelGC` | `Parallel Scavenge+Serial Old` | server模式的默认收集器，打开后会默认使用`Parallel Scavenge+Serial Old`组合回收 |
| `-XX:+UseParallelOldGC` | `Parallel Scavenge+Parallel Old` | 默认关闭，打开后默认使用`Parallel Scavenge+Parallel Old`组合回收 |
| `-XX:+UseConcMarkSweepGC` | `ParNew+CMS+Serial Old` | 默认关闭，打开后默认使用`ParNew+CMS+Serial Old`组合回收，Serial Old将作为CMS收集器出现Concurrent Mode Failure失败后的备用收集器 |
| `-XX:+ParNewGC` | `ParNew+Serial Old` | 默认关闭，打开后默认使用`ParNew+Serial Old`组合回收 |
Tip：client模式表示启动时加`-client`参数，对应server模式加`-server`参数！

### 收集器相关参数
| 参数名 | 简述 | 解释 |
| :------- | :------ | :------ |
| `-XX:ParallelGCThreads` | 并行GC时内存回收的线程数 | 当`cpu数量<=8`时，默认8个，`cpu数量>8`时，设置小于cpu数量 |
| `-XX:+ScavengeBeforeFullGC` | Full GC前是否先Minor GC | 默认true，Full GC前是否先Minor GC |
| `-XX:+UseGCOverheadLimit` | 默认开启 | 禁止GC无限制的执行，如果过于频繁，就直接抛OOM异常 |
| `-XX:+UseTLAB` | server模式默认开启 | 优先在本地线程缓冲区中分配对象，避免分配内存时的锁定过程，TLAB分配在eden区 |
| `-XX:+HeapDumpOnOutOfMemoryError` |  | 参数表示当JVM发生OOM时，自动生成DUMP文件。 |
| `-XX:HeapDumpPath` | `-XX:HeapDumpPath=${目录}/java_heapdump.hprof` | 表示生成DUMP文件的路径，也可以指定文件名称,默认文件名为：`java__heapDump.hprof`  |

`Parallel Scavenge`收集器参数：

| 参数名 | 简述 | 解释 |
| :--- | :--- | :--- |
| `-XX:GCTimeRatio` | GC时间占总时间比值 | 默认99，即允许1%的GC时间，仅在使用`Parallel Scavenge`收集器时生效。 |
| `-XX:MaxGCPauseMillis` | GC最大停顿时间 | 仅在使用`Parallel Scavenge`收集器时生效 |

`CMS`收集器参数：

| 参数名 | 简述 | 解释 |
| :--- | :--- | :--- |
| `-XX:CMSInitiatingOccupancyFraction` | 默认值68 | 设置CMS收集器在Old区使用多少后触发GC，默认Old区使用占总Old的68%时触发。 |
| `-XX:+UseCMSCompactAtFullCollection` | 默认开启 | 设置当CMS GC完后进行内存碎片整理 |
| `-XX:CMSFullGCsBeforeCompaction` | 无默认值  | 设置CMS进行若干次GC后再启动一次内存碎片整理 |

 Full GC触发原因：
1. Minor GC导致Survivor不能容纳下所有存活对象触发分配担保
2.  `System.gc()` 被显式调用
3. Perm Gen被写满
4. Old 被写满
5.  `Perm Gen`永久代出现GC，jdk1.7一般指存放class对象空间不足，通常会出现以下几种异常：
 1. `java.lang.OutOfMemoryError: PermGen space`；
 2. `java.lang.OutOfMemoryError:GC overhead limit exceeded`
 3. `ERROR 500.jsp._jspService:117]-Could not get a resource from the pool`  类加载异常，这里异常时加载解析的jsp类时异常！
 4. `Timeout`，一般Full GC都会伴随这系统对外的连接Timeout异常！
6. CMS GC时出现`promotion failed`和`concurrent mode failure`
`promotion failed`:在进行Minor GC时，survivor space放不下、对象只能放入旧生代，而此时旧生代也放不下；
`concurrent mode failure`：在执行CMS GC的过程中同时有对象要放入旧生代，而此时旧生代空间不足。
7. 统计得到的Minor GC晋升到旧生代的平均大小大于旧生代的剩余空间
这是一个较为复杂的触发情况，Hotspot为了避免由于新生代对象晋升到旧生代导致旧生代空间不足的现象。
在进行Minor GC时，做了一个判断，如果之前统计所得到的Minor GC晋升到旧生代的平均大小大于旧生代的剩余空间，那么就直接触发Full GC。
例如：程序第一次触发Minor GC后，有6MB的对象晋升到旧生代，那么当下一次Minor GC发生时，首先检查旧生代的剩余空间是否大于6MB，如果小于6MB，则执行Full GC。
当新生代采用PS GC时，方式稍有不同，PS GC是在Minor GC后也会检查，例如上面的例子中第一次Minor GC后，PS GC会检查此时旧生代的剩余空间是否大于6MB，如小于，则触发对旧生代的回收。


Tip：Full GC通常耗时较长，可能会导致停止应用响应STW(Stop-The-World Even )。
Tip：`Minor GC`指对survivor和eden区的gc；`Major GC`是指老年代和永久代的GC；`Full GC`通常就是`Major GC`，特指`stop the world`时的GC，且官方没有给出明确的定义！


## 四，GC日志参数
常用设置GC打印格式参数:
- `-XX:+PrintGCApplicationStoppedTime` 输出GC造成应用暂停的时间
- `-XX:+PrintGCTimeStamps` GC发生相对JVM启动的时间，样式：
   `188.220: [GC [PSYoungGen: 1279472K->80396K(1245696K)]`
- `-XX:+PrintGCDateStamps` GC具体发送时间，也包括相对JVM启动时间，样式：
   `2017-07-22T18:33:16.432+0800: 122.713: [GC [PSYoungGen: 1256318K->100567K(1262080K)]`
- `-XX:+DisableExplicitGC`： 关闭`System.gc()`，调用`System.gc()`会变成一个空操作
- `-Xloggc:$CATALINA_BASE/logs/gc.log` gc日志产生的路径
- `-xx:+printGCdetails`：打印GC详情到输出设备
即类似`[GC [PSYoungGen: 230112K->960K(219648K)] 265174K->36086K(303616K), 0.0045880 secs]`的输出
- `-verbose:gc`： verbose详细的意思,显示详细的GC信息，被`-xx:+printGCdetails`参数包括，可不写！
- `-XX:+PrintTenuringDistribution`：指定JVM 在每次新生代GC时，输出幸存区中对象的年龄分布。
- `-XX:+HeapDumpOnOutOfMemoryError` `-XX:HeapDumpPath=${PATH}/`: 当发生OOM时,dump下堆内存,HeapDumpPath指定目录时会自动生成`java_pid[xxx].hprof`的文件,这里也可以指定生成的文件名!
- `-XX:ErrorFile`: 当jvm出现致命错误时，会生成一个错误文件 `hs_err_pid<pid>.log`，其中包括了导致jvm crash的重要信息，当出现crash时，该文件默认会生成到工作目录下，也可通过ErrorFile参数指定:`-XX:ErrorFile=${PATH}/hs_err_pid_%p.log`

Tip：可参考：《深入理解Java虚拟机》第三章3.5.8节 理解GC日志

注意： 
一般系统都加上了这两个参数：`-verbose:gc`和`-xx:+printGCdetails` ，但是测试删掉`-verbose:gc`参数并没有什么改变！ 因为它已被printGCdetails参数包括了。

只有`-verbose:gc`参数，则GC样式为：`[GC 223188K->40186K(2260992K), 0.0315090 secs] `

只有 `-xx:+printGCdetails` 样式：
```
[GC [PSYoungGen: 1221316K->88958K(1279488K)] 1382475K->259135K(3114496K), 0.0625040 secs] [Times: user=0.18 sys=0.03, real=0.06 secs]
```
其中，`1221316K->88958K(1279488K)`表示：GC前该区域已使用容量->GC后该内存区域已使用量（该区域的占内存总大小）；
`1382475K->259135K(3114496K)`表示：GC前Java堆已使用大小->GC后Java堆已使用大小（Java堆总大小）

Tip：通常配置GC日志的方法：
```
-XX:+DisableExplicitGC -verbose:gc -XX:+PrintGCDateStamps -XX:+PrintGCDetails -Xloggc:$CATALINA_BASE/logs/gc.log
```

Tip：如果只有`-verbose:gc`参数，gc日志会输出到控制台上，如果`-verbose:gc`和`-Xloggc:filename`参数共存，以`-Xloggc`为准。

参考：
1. 《深入理解Java虚拟机》第三章3.5.8节 理解GC日志















