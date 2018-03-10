---
layout: post
title: "如何正确的为 MapReduce 配置内存分配"
date: "2018-03-09"
description: "如何正确的为 MapReduce 配置内存分配"
tag: [hadoop, yarn]
---

**环境:** CentOS-6.8-x86_64, hadoop-2.7.3

### 概述
YARN 会将集群上每个机器的可用计算资源统计; 基于可用资源, YARN 将会与运行在集群上的应用 (例如 MapReduce) 的资源请求协商; YARN 通过分配 Container 为每个应用提供处理容量; Container 是 YARN 中处理容量的基本单元, 是资源元素的封装 (内存, CPU 等); 在本文中假定使用的物理集群 slave 节点每个都有 48 GB 的内存, 12 个磁盘和 2 个 hex core 的 CPU (共 12 核)

###  配置 YARN
在 Hadoop 集群中, 对于内存, CPU, 磁盘的使用均衡是非常重要的, 这样在处理时才不会被集群资源中某一种而限制; 作为通用推荐, 我们发现对集群使用的最佳平衡是为每一个磁盘和内核分配 1 - 2 个 Container; 例如我们的集群节点有 12 个磁盘和 12 个内核, 所以我们将允许为每个节点分配最多 20 个 Container  
集群的每个机器有 48 GB 的内存, 应该为操作系统使用保留一些内存; 在每个节点上, 我们将分配 40 GB 的内存让 YARN 使用, 并未操作系统保留 8 GB 的内存; 以下属性会设置 YARN 在节点上使用的最大内存; 在 yarn-site.xml 中配置
```
...
<property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>40960</value>
</property>
...
```
接下来是为 YARN 提供指导如何将所有的可用资源拆分到 Containers 中; 需要指定为每个 Container 分配的最小内存单位, 我们想每个节点最大允许 20 的 Container, 因此每个 Container 最少需要  (40 GB 总内存) / (20 # Containers) = 2 GB
```
...
<property>
    <name>yarn.scheduler.minimum-allocation-mb</name>
    <value>2048</value>
</property>
...
```
YARN 将会为每个 Container 分配超过 yarn.scheduler.minimum-allocation-mb 的内存

### 配置 MapReduce2
MapReduce2 运行 YARN 之上, 并且使用 YARN Container 去调度和执行它的 Map 和 Reduce 任务; 当配置 MapReduce2 在 YARN 上的资源使用时, 需要考虑以下三个方面
- 为每个 Map 和 Reduce 任务限制物理内存
- 为每个任务限制 JVM heap 大小
- 每个任务获得的总虚拟内存

你可以定义每个 Map 和 Reduce 任务将会获得多大的内存, 因为每个 Map 和 Reduce 将会运行在单独的 Container 中, 这些最大内存设置应该至少大于等于 YARN 中 Container 的最小分配内存  
在我们的集群中, 为一个 Container 设置的最小内存时 2 GB, 因此我们为 Map 任务的 Container 分配 4 GB, 为 Reduce 任务的 Container 分配 8 GB; 在 mapred-site.xml 中配置
```
...
<property>
    <name>mapreduce.map.memory.mb</name>
    <value>4096</value>
</property>
<property>
    <name>mapreduce.reduce.memory.mb</name>
    <value>8192</value>
</property>
...
```
每个 Container 将会为 Map 和 Reduce 任务运行 JVM, JVM heap 大小应该设置低于以上 Map 和 Reduce 定义的大小, 这样它们就会在 YARN 分配的的 Container 内存限制内; 在 mapred-site.xml 中配置
```
...
<property>
    <name>mapreduce.map.java.opts</name>
    <value>-Xmx3072m</value>
</property>
<property>
    <name>mapreduce.reduce.java.opts</name>
    <value>-Xmx6144m</value>
</property>
...
```
以上配置了 Map 和 Reduce 任务使用的物理内存上限; 每个 Map 和 Reduce 任务使用的虚拟内存 (物理内存 + 页内存) 由每个 YARN Container 的虚拟内存比率所决定; 可以铜鼓以下配置设置, 默认值是 2.1; 在 yarn-site.xml 中配置
```
...
<property>
    <name>yarn.nodemanager.vmem-pmem-ratio</name>
    <value>2.1</value>
</property>
...
```
这样, 我们集群配置以上的设置, 每个 Map 任务将会获得以下的分配内存
- 总物理内存分配 = 4 GB
- 每个 Map 任务 Container 的 JVM heap 空间上限 = 3 GB
- 虚拟内存上限 = 4 * 2.1 = 8.4 GB

在 YARN 和 MapReduce2 中, 将不再为 Map 和 Reduce 任务预先配置静态槽, 因为作业的需要, 对于Maps 和 Reduces 的动态资源分配在集群中是可获得的; 在我们的集群中设置以上配置, YARN 可以为一个节点分配 10 个 mapper 或 5 个 reducers 或者在限制内置换

**参考:**
[HOW TO PLAN AND CONFIGURE YARN AND MAPREDUCE 2 IN HDP 2.0](https://hortonworks.com/blog/how-to-plan-and-configure-yarn-in-hdp-2-0)
