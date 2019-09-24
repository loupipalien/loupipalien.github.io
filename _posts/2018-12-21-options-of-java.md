---
layout: post
title: "Java 的可选项"
date: "2018-12-21"
description: "Java 的可选项"
tag: [java]
---

### 语法
```
java [options] classname [args]
java [options] -jar filename [args]
```
TODO

### 可选项
TODO

#### 高级垃圾收集器可选项
以下可选项用于控制 Java HotSpot VM 如何执行
- -XX:+AggressiveHeap
开启 Java 堆优化; 对于长时间的运行的内存密集型的作业, 基于计算机的配置 (CPU 和 RAM) 可以设置几个参数优化; 默认的, 此选项是禁止的, 堆内存是不优化的

- -XX:+AlwaysPreTouch
TODO

- -XX:+CMSClassUnloadingEnabled
当使用并行标记清除 (CMS) 垃圾收集器时开启 class 卸载; 这个选项默认是开启的; 可指定 `-XX:-CMSClassUnloadingEnabled` 关闭 CMS 垃圾收集器的 class 卸载

- -CMSExpAvgFactor=percent
TODO

- -XX:CMSInitiatingOccupancyFraction=percent
设置老年代的占用比 (0 - 100) 已开始一个 CMS 收集周期; 默认值是 -1; 任何负值 (包括默认值) 意味着使用 `-XX:CMSTriggrRatio` 定义初始占用比的值; 以下示例展示如何设置占用比为 20%, `-XX:CMSInitiatingOccupancyFraction=20`

- -XX:+CMSScavengeBeforeRemark
在 CMS 标记阶段尝试清理, 此选项默认是禁止的

- -XX:CMSTrigerRatio=percent
设置由 `-XX:MinHeapFreeRatio` 指定值的百分比, 表示在一个 CMS 周期开始前已分配的占比; 默认值是 80%; 以下示例展示如何设置占用比为 75%, `-XX:CMSTrigerRatio=75`

- -XX:ConcCMSThreads=threads
设置用于并行 GC 线程的数量; 默认值依赖于 JVM 可使用的 CPU 数; 例如, 将并行 GC 线程数设置为 2, 可按以下指定: `-XX:ConcCMSThreads=2`

- -XX:+DisableExplicitGC
开启禁止对 `System.gc()` 的调用处理; 默认是禁止的, 意味着 `System.gc()` 的调用会被处理; 如果 `System.gc()` 的调用禁止被处理; JVM 在必要时仍然会执行 GC

- -XX:ExplicitGCinvokesConcurent
对使用 `System.gc()` 的请求开启并行 GC 的调用; 此选项默认是禁止的, 仅与 `-XX:+UseConcMarkSweepGC` 一起使用时被开启

- -XX:ExplicitGCinvokesConcurentAndUnloadClasses
对使用 `System.gc()` 的请求开启并行 GC 的调用, 并且在并行 GC 期间卸载 classes; 此选项默认是禁止的, 仅与 `-XX:+UseConcMarkSweepGC` 一起使用时被开启

- -XX:G1HeapRegionSize=size
TODO

- -XX:G1PrintHeapRegions
TODO

- -XX:G1ReservePercent=percent
TODO

- -XX:InitialHeapSize=size
设置内存分配池的初始大小 (字节); 这个值必须是 0 或者 1024 的倍数且大于 1MB; 追加字母 k 或者 K 表示千字节, m 或者 M 表示兆字节, g 或者 G 表示千兆字节; 默认值基于运行时的系统配置, 见 [Java SE HotSpot Virtual Machine Garbage Collection Tuning Guide](http://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/index.html) 的 Ergonomics 章节; 如果此选项设置为 0, 初始值大小就会别设置为老年代和年轻代分配大小的总和; 堆上的年轻代大小可以使用  `-XX:NewSize` 选项设置; 以下示例表示如何用不同的单位设置 6MB 大小的分配池
  ```
  -XX:InitialHeapSize=6291456
  -XX:InitialHeapSize=6144k
  -XX:InitialHeapSize=6m
  ```
- -XX:InitialSurvivorRatio=ratio
用于设置吞吐量垃圾收集器 (使用 `-XX:+UseParallelGC` 或 `-XX:+UseParallelOldGC` 开启) 的初始幸存者空间比例; 自适应大小在使用吞吐量垃圾收集器默认是开启的, 吞吐量垃圾收集器使用
 `-XX:+UseParallelGC` 或 `-XX:+UseParallelOldGC` 开启, 幸存者空间会根据应用行为调整大小, 起始时是初始值; 如果自适应大小是禁止的 (使用 `-XX:-UseAdaptiveSizePolicy` 选项), 则 `-XX:InitialSurvivorRatio` 选项设置的幸存者空间大小会在应用运行的整个期间使用; 以下公式用于计算基于年轻代 (Y) 的大小和初始幸存者空间比例 (R) 的幸存者空间得初始大小, `S=Y/(R+2)`, 2 表示两个幸存者空间, 初始幸存者空间比例指定的越大, 初始幸存者空间越小：默认的, 初始幸存者空间比例是 8; 如果年轻代空间使用了默认值 (2MB), 则幸存者空间初始大小将会是 0.2MB; 以下示例展示如何将幸存者空间比例设置为 4: `-XX:InitialSurvivorRatio=4`

- -XX:InitiatingHeapOccupancyPercent=percent
设置在开始一个并行 GC 周期前堆占用的比例 (0-100); 它用于垃圾收集器基于整个堆的占用触发一个并行 GC 周期, 并不是某一代 (例如 G1 垃圾收集器); 默认的, 初始值设置为 45%, 设置为 0 表示不停的 GC; 以下示例展示如何将初始堆占用设置为 75%: `-XX:InitiatingHeapOccupancyPercent=75`

- -XX:MaxGCPauseMillis=time
为 GC 暂停最大值 (毫秒) 设置一个目标; 这是一个软性的目标, JVM 将会尽可能的达到它; 默认没有设置最大暂停时间; 以下示例展示如何设置最大暂停时间为 500ms: `-XX:MaxGCPauseMillis=500`

- -XX:MaxHeapSize=size
设置内存分配池的最大大小 (字节); 这个值必须是 1024 的倍数, 并且必须大于 2MB; 追加字母 k 或者 K 表示千字节, m 或者 M 表示兆字节, g 或者 G 表示千兆字节; 默认值基于运行时的系统配置; 对于服务器部署, `-XX:InitialHeapSize` 和 `-XX:MaxHeapSize` 常被设置为相同的值; 见 [Java SE HotSpot Virtual Machine Garbage Collection Tuning Guide](http://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/index.html) 的 Ergonomics 章节; 在 Oracle Solaris 7 和 Oracle Solaris 8 SPARC 平台上, 这个值的上限大约是 4000MB 减去已开销; 在 Oracle Solaris 2.6 和 x86 平台上, 这个值的上限大约是 2000MB 减去已开销; 在 Linux 平台上, 这个值的上限大约是 2000MB 减去已开销; `-XX:MaxHeapSize` 等同于 `-Xmx`, 以下示例表示如何用不同的单位设置分配池的最大大小为 80 MB

- -XX:MaxHeapFreeRatio=percent
设置在一次 GC 之后的堆空闲空间的最大允许百分比, 如果空间空间增加了超过了这个值, 那么堆大小将会缩小; 默认值是 75%; 以下示例展示了如何设置最大堆空闲空间的比例为 75%: `-XX:MaxHeapFreeRatio=75`

- -XX:MaxMetaspaceSize=size
设置为类元数据分配的最大内存, 默认的这个大小是不收限制的; 一个应用的元数据依赖于应用本身, 其他运行的应用, 以及系统中的可用内存大小; 以下示例展示如何设置最大的类元数据大小为 256MB: `-XX:MaxMetaspaceSize=256m`

- -XX:MaxNewSize=size
设置堆为年轻代设置的最大内存大小, 默认设置是符合最大效能的

- -XX:MaxTenuringThreshold=threshold
为自适应 GC 大小设置最大的任期阈值; 最大值是 15, 并行吞吐量收集器默认值是 15, CMS 收集器是 6; 以下示例展示如何设置最大任期阈值为 10: `-XX:MaxTenuringThreshold=10`

- -XX:MetaspaceSize=size
设置分配给元数据空间的大小, 在第一次超过这个值时会触发一次垃圾回收; 垃圾回收的阈值会依赖于使用的元数据数据增加或减少; 默认值依赖于平台

- -XX:MinHeapFreeRatio=percent
设置在一次 GC 之后的堆空闲空间得最小允许百分比; 如果空闲空间下降到低于这个值, 堆将会扩容; 默认值是 40%; 以下示例展示如何设置最小空闲堆比例为 25%: `-XX:MinHeapFreeRatio=25`

- -XX:NewRatio=ratio
设置年轻代和老年代的比值大小; 此项默认值是 2; 以下示例展示如何设置年轻代/老年代比值为 1: `-XX:NewRatio=1`

- -XX:NewSize=size
设置堆上年轻代的初始大小 (字节); 加字母 k 或者 K 表示千字节, m 或者 M 表示兆字节, g 或者 G 表示千兆字节; 堆上的年轻代用于新生的对象, 这个区域的 GC 执行次数远多于其他区域; 如果年轻代的值太低, 将会有大量的 minor GCs 产生; 如果这个值设置的太高, 将只有 full GCs 产生, 这将花费很长的时间完成; Oracle 推荐年轻代的大小在堆的四分之一到二分之一之间; `-XX:NewSize` 选项等同于 `-Xmx`; 以下示例展示如何使用不同的单位设置年轻代的初始值为 256MB
  ```
  -XX:NewSize=256m
  -XX:NewSize=262144k
  -XX:NewSize=268435456
  ```

- -XX:ParallelGCThreads=threads
在年轻代和老年代设置并行收集器使用的线程数; 默认值依赖于 JVM 可获得的 CPU 数; 以下示例展示如何设置并行 GC 的线程数为 2: `-XX:ParallelGCThreads=2`

- -XX:+ParallelRefEnabled
开启并行引用处理; 此项默认是禁止的

- -XX:+PrintAdaptiveSizePolicy
开启打印自适应代大小的信息; 此项默认是禁止的

- -XX:+PrintGC
开启打印每一次的 GC 信息; 此项默认是禁止的

- -XX:+PrintGCApplicationConcurrentime
开启打印从最近暂停 (例如 GC 暂停) 开始耗时多久; 此项默认是禁止的

- -XX:+PrintGCApplicationStopTime
开启戴颖最近的暂停耗时多久; 此项默认是禁止的

- -XX:+PrintGCDateStamps
开启打印每一次 GC 的时间戳; 此项默认是禁止的

- -XX:+PrintGCDetails
开启打印每一次 GC 的详细消息; 此项默认是禁止的

- -XX:+PrintGCTaskTimeStamps
开启打印每一个独立 GC 工作线程任务的时间戳; 此项默认是禁止的

- -XX:+PrintStringDeduplicationStatistics
打印详细的去重统计; 此选项默认是禁止的, 见 `-XX:+UseStringDeduplication` 选项

- -XX:+PrintTenuringDistribution
开启打印任期年龄信息; 一岁的对象是最新的幸存者 (它们是在之前的清除后创建, 在最新的清除中幸存, 并且从 eden 移动到 survivor 区), 两岁的对象是在两次清除中幸存 (在第二次清除期间, 它们将从一个 survivor 区拷贝到下一个 survivor 区), 以此类推; 在以下的示例中, 28 992 024 字节在一次清除后幸存, 并且从 eden 区拷贝到 survivor 区, 拷贝的 1 366 864 字节是两岁的对象, 每行中的第三个值是 n 岁对象的累积大小; 此项默认是关闭的; 以下是输出示例
  ```
  Desired survivor size 48286924 bytes, new threshold 10 (max 10)
  - age 1: 28992024 bytes, 28992024 total
  - age 2: 1366864 bytes, 30358888 total
  - age 3: 1425912 bytes, 31784800 total
  ...
  ```
- -XX:+ScavengeBeforeFullGC
在每一次 full GC 前开启一次年轻代的 GC; 此选项默认是开启的; Oracle 推荐你不要禁止它, 在 full GC 前清除年轻代可以减少从老年代到年轻代的可达对象的数量; 在每次 full GC 前禁止年轻代的 GC 这样指定: `-XX:-ScavengeBeforeFullGC`

- -XX:SoftRefLRUPolicyMSPerMB=time
TODO

- -XX:StringDeduplicationAgeThreshold=threshold
`String` 对象到达指定年龄可以被考虑去重; 一个对象的年龄被用来计算在垃圾回收中幸存的次数; 这被认为是调优; 见 `-XX:+PrintTenuringDistribution` 选项; 注意 `String` 对象在到达这年龄晋升到老年代之前, 总是被考虑去重; 此项的默认值是 3, 见 `-XX:+UseStringDeduplication` 项

- -XX:SurvivorRatio=ratio
设置 eden 空间和 survivor 空间的比例; 此项默认值是 8, 以下示例展示如何设置 eden/space 空间的比例为 4: `-XX:SurvivorRatio=4`

- XX:TargetSurvivorRatio=percent
设置在 young GC 后需要的幸存者空间的百分之比; 此项默认值是 50 %; 以下示例展示如何将目标幸存者空间比例设置为 30%: `-XX:TargetSurvivorRatio=30`

- -XX:TLABSize=size
设置本地线程分配的缓存 (TLAB) 初始值大小; 加字母 k 或者 K 表示千字节, m 或者 M 表示兆字节, g 或者 G 表示千兆字节; 如果设置为 0, JVM 会自动初始化值; 以下示例展示如何设置 TLAB 大小到 512 KB: `-XX:TLABSize=512k`

- -XX:+UseAdaptiveSizePolicy
开启使用自适应大小; 此项默认是开启的; 指定 `-XX:-UseAdaptiveSizePolicy` 可以禁止自适应大小, 并且明确的设置内存分配池的大小 (见 ` -XX:SurvivorRatio ` 选项)

- -XX:+UseCMSInitaitingOccupancyOnly
TODO

- -XX:+UseConcMarkSweepGC
为老年代开启 CMS 垃圾收集器的使用; Oracle 推荐你使用 CMS 垃圾收集器, 当应用的延迟要求不能被吞吐量垃圾收集器 (`-XX:+UseParallelGC`) 满足时; G1 垃圾收集器 (-XX:+UseG1GC) 是另一个选择; 此项默认是禁止的, 收集器会根据机器的配置和 JVM 的类型自动选择; 当此项开启时, `-XX:+UseParNewGC` 将自动被设置并且你不应该禁止它, 因为在 JDK 8 中 ` -XX:+UseConcMarkSweepGC -XX:-UseParNewGC` 已过时

- -XX:+UseG1GC
开启使用 G1 (garbage-first) 垃圾收集器; 它是服务器级的垃圾收集器, 针对有大量 RAM 的多核处理器; 它尽可能的满足暂停时间目标, 并且维护好的吞吐量; G1 收集器推荐为要求大量堆内存 (大约 6 GB 或者更大) 且限制 GC 延迟要求 (稳定的可预测的暂停时间小于 0.5 秒) 的应用

- -XX:+UseGCOverheadLimit
开启 JVM 在 GC 上花费时间超过限制比例前抛出一个 `OutOfMemoryError` 异常的使用策略; 这个选项默认是开启的, 如果总体时间的 98% 花费在垃圾收集上, 而少于 2% 的时间堆内存是恢复的, 那么并行 GC 将会抛出一个 `OutOfMemoryError` 异常; 当堆内存较小时, 这项特性可以用于避免应用长时间在 GC 而只有很少的时间在处理; 指定 `-XX:-UseGCOverheadLimit` 可以禁止此选项

- -XX:+UseNUMA
开启通过增加应用的第延迟内存的使用提高应用的在非均匀内存价格 (NUMA) 的机器上优化应用性能; 默认此项是禁止的, 在 NUMA 上不做优化; 此项仅在使用并行垃圾收集器时可用 (`-XX:+UseParallelGC`)

- -XX:UseParallelGC
开启并行清除垃圾收集器的使用 (即吞吐量垃圾收集器), 通过利用多个处理器来提高你应用的性能; 此选项默认是禁止的, 收集器会基于机器配置和 JVM 类型自动选择; 如果开启, `-XX:+UseParallelOldGC` 选项会被自动开启, 除非你明确禁止它

- -XX:+UseParallelOldGC
开启使用并行垃圾收集器进行 full GCs; 默认是禁止的, 在开启 `-XX:+UseParallelGC` 选项时会被自动开启

- -XX:+UseParNewGC
开启使用多个线程对年轻代进行垃圾收集; 默认是禁止的, 如果设置了 `-XX:+UseConcMarkSweepGC ` 选项将会被自动打开; 使用 `-XX:+UseParNewGC` 选项而没有 ` -XX:+UseConcMarkSweepGC` 在 JDK8 中已过时

- -XX:+UseSerialGC
开启使用串行垃圾收集器; 对于对于垃圾回收没有任何特殊要求的, 简单的小应用通常是最好的选择; 默认是禁止的, 收集器会基于机器配置和 JVM 类型自动选择

- -XX:+UseSHM
TODO

- -XX:UseStringDeduplication
开启字符串去重; 此项默认是禁止的, 为了使用这个选项, 你必须开启 G1 垃圾回收器, 见 `-XX:+UseG1GC` 选项; 字符串去重会减少 String 对象的在 Java 堆上的内存占用, 基于许多 String 对象都是相同的事实; 相同的 String 对象可以指向并共享相同的字符数组, 而不是每一个 String 对象都指向自己的字符数组

- -XX:+UseTLAB
开启在年轻代区域使用本地线程分配块 (TLAB), 此项默认是开启的; 可以指定 `-XX:-UseTLAB` 禁止 TLABs 的使用

#### 过期和移除的选项
TODO

>**参考:**
[Launches a Java application](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html)
