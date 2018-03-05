---
layout: post
title: "Hive 中控制 mapreduce 的作业数"
date: "2018-03-05"
description: "Hive 中控制 mapreduce 的作业数"
tag: [hive]
---

**环境:** CentOS-6.8-x86_64, hadoop-2.7.3, hive-2.1.1

### 控制 Hive 作业中的 map 数
主要决定因素有输入的文件个数, 输入的文件大小, Hadoop 集群设置的文件块大小 (通常为 128M 或 256M, 可使用 set dfs.block.size 查询, 该参数是 Hadoop 集群设置, Hive 中不可修改)
- 小文件问题  
每个输入文件如果大于块大小则会按块大小切分起对应个数的 map, 小于块大小的文件也会起一个 map; 文件切分为多个 map 并行计算会加快执行速度, 但也不是越多的 map 数越好, 当 map 任务启动和初始化的时间远大于逻辑计算的时间反而会执行变慢
- 多记录问题
当一个文件不大于块大小, 但只有一个或几个字段, 这个文件就会有比较多的记录, 这时只起一个 map 处理也比较耗时

针对小文件问题可以采用合并小文件减少 map 数处理, 而多记录问题则可以切分文件增加 map 数处理

#### 如何合并输入小文件以减少 map 数
```
// 每个 map 最大输入大小, 决定合并后的文件数; 默认值: 256000000, 老版本参数名: mapred.max.split.size
set mapreduce.input.fileinputformat.split.maxsize = 100000000;
// 每个 map 最小输入大小, 决定合并后的文件数; 默认值: 1, 老版本参数名: mapred.min.split.size
set mapreduce.input.fileinputformat.split.minsize = 1;
// 每个节点上切割的最小的大小, 决定了多个 data node 上的文件是否需要合并; 默认值: 1, 老版本参数名: mapred.min.split.size.per.rack
set mapreduce.input.fileinputformat.split.minsize.per.node = 100000000;
// 每个机架上切割的最小的大小，决定了多个交换机上的文件是否需要合并; 默认值: 1, 老版本参数名: mapred.min.split.size.per.node
set mapreduce.input.fileinputformat.split.minsize.per.rack = 100000000;
// 合并 Hive 输入小文件类名
set hive.input.format = org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
```
TODO 解释如何切分

#### 如何切分输入文件以增加 map 数
例如执行一些语句
```
select data_desc, count(1), count(distinct id), sum(case when …), sum(case when ...), sum(…) from a group by data_desc;
```
如果 a 表只有一个文件但包含很多记录数, 使用一个 map 处理是非常耗时的; 处理思路是基于 a 表创建多个输入文件的临时表后, 再基于临时表计算
```
create table a_tmp as select * from a distribute by rand(123);
select data_desc, count(1), count(distinct id), sum(case when …), sum(case when ...), sum(…) from a_tmp group by data_desc;
```

### 控制 Hive 作业中的 reduce 数
主要决定的配置参数为 hive.exec.reducers.bytes.per.reducer (默认值: 256000000), hive.exec.reducers.max (默认值: 1009) 以及 mapreduce.job.reduces (默认值: -1)  
- 使用 mapreduce.job.reduces 设置 (老版本参数名: mapred.reduce.tasks)
当有 reduce 作业时, 设置 mapreduce.job.reduces 参数, 则会启动 mapreduce.job.reduces 值个数的 reduce 作业;
- 使用 hive.exec.reducers.bytes.per.reducer 设置
会将输入按照设置的大小切割为多个 reduce 任务, 当 mapreduce.job.reduces 设置为 -1 时, 最大的 reduce 个数为 hive.exec.reducers.max
与 map 设置同理, 并不是 reduce 个数越多越好; 大数据量配置合理的 reduce 数, 使用单个 reduce 处理合适的数据量

#### 什么情况下只有一个 reduce

>**参考:**
[Hive任务优化--控制hive任务中的map数和reduce数](http://blog.csdn.net/michael_zhu_2004/article/details/8284089)  
[Hive中如何确定map数](http://blog.javachen.com/2013/09/04/how-to-decide-map-number.html)
