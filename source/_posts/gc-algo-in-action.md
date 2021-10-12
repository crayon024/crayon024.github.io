---
title: GC 策略结合日志分析
date: 2021-07-09 18:54:14
updated: 2021-07-09 18:54:14
categories: 
  - JVM
tags: 
  - jvm
  - gc
  - java
---

## Java 8 默认 GC 策略，并行 GC

java -Xloggc:gc.java_8_default.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xms512m -Xmx512m week02.work01.GCLogAnalysis  <!--more-->

共生成对象次数：10277 次

因为Java 8 默认采用 ParallelGC 策略，而策略默认会打开 AdaptiveSizePolicy 自适应大小策略。

添加参数关闭 -XX:-UseAdaptiveSizePolicy 后，可以看到 Y 和 O 按照默认的 1：2 分配。

```plain
CommandLine flags: -XX:InitialHeapSize=536870912 -XX:MaxHeapSize=536870912 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:-UseAdaptiveSizePolicy -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC 
```

首先日志打印了部分相关的启动参数配置，可以看到默认使用了 ParallelGC，开启了压缩指针。

```plain
0.231: [GC (Allocation Failure) [PSYoungGen: 131501K->21491K(153088K)] 131501K->47291K(502784K), 0.0243533 secs] [Times: user=0.02 sys=0.08, real=0.02 secs] 
```

第一次 GC 发生在 Y 区，原因是 Allocation Failure，Eden 区没有足够的空间分配对象触发的。 此次 GC 后 Y 区占用空间少了了大概 110 M，总空间占用大概少了 80 M。大概晋升了 30M 到老年代中。用时 24 ms。

```plain
0.497: [Full GC (Ergonomics) [PSYoungGen: 21495K->0K(153088K)] [ParOldGen: 315331K->232451K(349696K)] 336826K->232451K(502784K), [Metaspace: 2575K->2575K(1056768K)], 0.1176023 secs] [Times: user=0.07 sys=0.35, real=0.11 secs] 
```

大概 8 次 YGC 后，发生第一次 FGC。这次的 FGC ，Y 区被清空，释放大概 20 M空间，O 区释放了大概 80 M 。总空间大概释放了 100 M，暂停时间 117 ms。
发生 FGC 的原因： [Major GC和Full GC的区别是什么？触发条件呢？ - 知乎](https://www.zhihu.com/question/41922036/answer/93079526)。

不同的 GC 策略有着不同的触发条件，Full GC 的执行方式也可能不一样。

并行 GC 默认启用 CPU 核心数的 GC 线程，**-XX:ParallelGCThreads=N** 控制。也是就说并行 GC 时 CPU 资源倾力去 GC。**使用并行 GC 策略目标更倾向于提高系统整体的吞吐量，单次的 GC 暂停时间有时可能会比较长**，业务线程的响应可能比较慢。

## 串行 GC

java -Xloggc:gc.serialGC.log -XX:+UseSerialGC -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xms512m -Xmx512m week02.work01.GCLogAnalysis 

共生成对象次数:11672

```plain
0.245: [GC (Allocation Failure) 2021-08-10T22:00:12.680-0800: 0.245: [DefNew: 139776K->17472K(157248K), 0.0273220 secs] 139776K->46776K(506816K), 0.0275854 secs] [Times: user=0.00 sys=0.02, real=0.02 secs] 
```

YGC 和 并行 GC 的 log 日志大概是一样，DefNew -年轻代释放了大概 120 M 的空间，总空间减少了大概 90 M。可以知道此次 YGC 大概有 30M 跑到了老年代。用时 27 ms。

```plain
1.054: [Full GC (Allocation Failure) 2021-08-10T22:00:13.490-0800: 1.054: [Tenured: 349521K->341013K(349568K), 0.0228613 secs] 506738K->341013K(506816K), [Metaspace: 2575K->2575K(1056768K)], 0.0229624 secs] [Times: user=0.02 sys=0.00, real=0.03 secs] 
```

大概发生 10 次 YGC 后，触发了一次 FGC。Tenured - 老年代释放了 10 M 的对象，总空间少了大概 160。也可以看出 DefNew 被清空了。用时 230 ms。**相比并行 GC 大概慢了一倍。**

## CMS GC

java -Xloggc:gc.CMSGC.log -XX:+UseConcMarkSweepGC -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xms512m -Xmx512m week02.work01.GCLogAnalysis 

共生成对象次数:12707

```plain
CommandLine flags: -XX:InitialHeapSize=536870912 -XX:MaxHeapSize=536870912 -XX:MaxNewSize=178958336 -XX:MaxTenuringThreshold=6 -XX:NewSize=178958336 -XX:OldPLABSize=16 -XX:OldSize=357912576 
```

* **-XX:MaxTenuringThreshold=6，对象年龄，控制晋升速率。本身由 JVM 自动调整，6 设置一个最大的限制值。**
* free-list，老年代采用**标记-清除，不整理**，通过维护 free-list 分配空间。

```plain
2021-08-10T22:26:09.781-0800: 0.273: [GC (Allocation Failure) 2021-08-10T22:26:09.781-0800: 0.273: [ParNew: 139776K->17471K(157248K), 0.0223003 secs] 139776K->46040K(506816K), 0.0225398 secs] [Times: user=0.02 sys=0.07, real=0.02 secs] 
```

**ParNew****~~新生代~~****（区分新生代（Eden 区）和年轻代）并行回收**，释放了大概 120 M 空间，GC 暂停 22ms。总空间释放了大概 90 M，大概也是晋升了 30 M 的内存。

```plain
0.472: [GC (CMS Initial Mark) [1 CMS-initial-mark: 212269K(349568K)] 229884K(506816K), 0.0009085 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
0.474: [CMS-concurrent-mark-start]
0.477: [CMS-concurrent-mark: 0.003/0.003 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
0.477: [CMS-concurrent-preclean-start]
0.478: [CMS-concurrent-preclean: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
0.478: [CMS-concurrent-abortable-preclean-start]
```

针对老年代的回收，触发了 CMS 的 6 个处理阶段。可以看到

1. **初始标记**：**STW**  0.9 ms。标记所有的 Root 对象，Root 引用的对象，年轻代存活对象引用的对象。（跨代引用）
2. 并发标记：遍历老年代，标记存活对象。
3. 并发预清理：因为并发执行，业务线程还在执行，引用关系一直在发生改变，这一步通过卡表的概念，标记脏表。

```plain
0.581: [GC (CMS Final Remark) [YG occupancy: 18010 K (157248 K)]
0.581: [Rescan (parallel) , 0.0002754 secs]
0.581: [weak refs processing, 0.0000085 secs]
0.581: [class unloading, 0.0003406 secs]
0.582: [scrub symbol table, 0.0001719 secs]
0.582: [scrub string table, 0.0000610 secs][1 CMS-remark: 343397K(349568K)] 361407K(506816K), 0.0011452 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
0.582: [CMS-concurrent-sweep-start]
0.583: [CMS-concurrent-sweep: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
0.583: [CMS-concurrent-reset-start]
0.584: [CMS-concurrent-reset: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

4. **最终标记：STW**来准确的标记对象，为接下来的删除对象做准备。
5. 并发清除：回收不使用的对象占用的空间
6. 并发重置：重置 CMS 算法为了回收内部维护的相关数据，为下一次做准备。

CMS 默认使用 CPU 核心数 / 4 的 GC 线程进行回收。**CMS 的目的在于降低系统的延迟，保证业务响应，用户体验更好些**。

在吞吐量方面，相比并行 GC 投入所有的核来GC，会差一些。

**（假设并行GC自定义线程数和 CMS 一样，并行 GC 的吞吐量的优势还在吗？**

个人理解：应该还是有的。CMS 的并发清除步骤多，分配到的 GC 线程次数多的话，业务线程分配不到资源，并发执行的优势没有了；采用的算法（没有整理，用 free-list）内存空间的利用率也是个问题。相当于分配了那么多资源，但是没发挥出来。


CMS 的存在的问题：

* 浮动垃圾：最终标记后，并发清除的操作不需要 STW，在清除的过程中可能要分配新的对象内存等等，所以需要预留一定的内存空间给业务线程使用。如果没有足够的空间就会导致并发失败。

```plain
0.647: [CMS-concurrent-abortable-preclean-start]
0.660: [GC (Allocation Failure) 2021-08-10T22:26:10.168-0800: 0.660: [ParNew: 157246K->17471K(157248K), 0.0154900 secs] 437850K->342263K(506816K), 0.0155564 secs] [Times: user=0.09 sys=0.01, real=0.02 secs] 
0.690: [GC (Allocation Failure) 2021-08-10T22:26:10.198-0800: 0.690: [ParNew: 157247K->157247K(157248K), 0.0000176 secs]2021-08-10T22:26:10.198-0800: 0.690: [0.690: [CMS-concurrent-abortable-preclean: 0.001/0.043 secs] [Times: user=0.11 sys=0.01, real=0.04 secs] 
 (concurrent mode failure): 324792K->289452K(349568K), 0.0785084 secs] 482039K->289452K(506816K), [Metaspace: 2575K->2575K(1056768K)], 0.0785991 secs] [Times: user=0.03 sys=0.04, real=0.08 secs] 
0.782: [GC (Allocation Failure) 2021-08-10T22:26:10.290-0800: 0.782: [ParNew: 139542K->17470K(157248K), 0.0045904 secs] 428995K->335339K(506816K), 0.0046549 secs] [Times: user=0.03 sys=0.00, real=0.00 secs] 
0.787: [GC (CMS Initial Mark) [1 CMS-initial-mark: 317869K(349568K)] 335860K(506816K), 0.0001277 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
0.787: [CMS-concurrent-mark-start]
```

可以看到在 pre-clean-start 后，日志出现了 concurrent mode failure 字样后，经历几次 Minor GC 后，重新从 Inital Mark 开始进行 CMS 操作。

**这里有个疑问，从日志看没有退化 GC 的情况，而是重新进行了 CMS。从《深入理解 Java 虚拟机 第 3 版》说会直接退化成 Serial Old GC，不知该如何验证？**

## G1 GC

java -Xloggc:gc.G1GC-30ms-clean.log -XX:+UseG1GC -XX:MaxGCPauseMillis=30  -XX:+PrintGCDateStamps -Xms512m -Xmx512m week02.work01.GCLogAnalysis

共生成对象次数：5910

```plain
CommandLine flags: -XX:InitialHeapSize=536870912 -XX:MaxGCPauseMillis=30 -XX:MaxHeapSize=536870912 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseG1GC -XX:-UseLargePagesIndividualAllocation 

6.629: [GC pause (G1 Evacuation Pause) (young) 264M->210M(512M), 0.0096778 secs]
[GC pause (G1 Humongous Allocation) (young) (initial-mark) 234M->222M(512M), 0.0039857 secs]
```

看到 G1 Humongous Allocation 因为大对象分配失败，触发了 initial-mark。

```plain
6.650: [GC concurrent-root-region-scan-start]
6.650: [GC concurrent-root-region-scan-end, 0.0003724 secs]
6.650: [GC concurrent-mark-start]
6.685: [GC concurrent-mark-end, 0.0347063 secs]
6.698: [GC remark, 0.0030734 secs]
6.709: [GC cleanup 319M->307M(512M), 0.0005021 secs]
6.709: [GC concurrent-cleanup-start]
6.709: [GC concurrent-cleanup-end, 0.0000177 secs]
6.959: [GC pause (G1 Evacuation Pause) (young)-- 431M->394M(512M), 0.0097375 secs]
6.978: [GC pause (G1 Evacuation Pause) (mixed) 398M->383M(512M), 0.0051953 secs]
```

后开始 region 的扫描，标记，重标记，cleanup 日志都有体现。
mix 模式类似 FGC。