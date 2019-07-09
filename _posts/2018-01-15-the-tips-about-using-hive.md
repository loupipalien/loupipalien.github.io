---
layout: post
title: "关于使用 hive 的小技巧"
date: "2018-01-15"
description: "关于使用 hive 的小技巧"
tag: [hive]
---

### 指定 hive 执行引擎 (默认是 mr, 可支持 tez 和 spark 引擎)
set hive.execution.engine = mr

### 指定 mapreduce 作业使用的队列
set mapred.job.queue.name = root.default;
set mapreduce.job.queuename = root.default;

### 指定 tez 作业使用的队列
set tez.queue.name = root.default;

### 指定 mapreduce 作业名
set mapreduce.job.name = job_name_example;

### 设置查询表时表所在目录可递归, tez 引擎会设置为 true, mr 引擎则不会
mapred.input.dir.recursive = true;
mapreduce.input.fileinputformat.input.dir.recursive = true;

### 设置查询结果集文件格式  
set hive.query.result.fileformat = [textfile, sequencefile, rcfile, llap]

### 设置查询转换为 fetch
set hive.fetch.task.conversion = [none|minimal|more]

### 设置查询时输入数据超过多大时不转换
set hive.fetch.task.conversion.threshold = 1073741824

### 设置 FetchTask 的聚合, 可能减少无分组的聚合查询子句的执行时间
set hive.fetch.task.aggr = true

### 数据倾斜优化参数
set hive.map.aggr = true;
set hive.groupby.skewindata = true;

### SQL 运行中可修改的白名单配置
hive.security.authorization.sqlstd.confwhitelist

### 除了 Hive 默认配置项以外, 用户可追加能修改的配置项
hive.security.authorization.sqlstd.confwhitelist.append

### 运行时不可更改的配置项
hive.conf.restricted.list

### 运行时对于用户隐藏的配置项
hive.conf.hidden.list

### Hive 运行时对内使用的配置项
hive.conf.internal.variable.list

###

### 创建临时函数
`CREATE TEMPORARY FUNCTION counter AS 'com.sf.ops.UDFRemainingTimeCounter' using jar 'hdfs:///user/01139983/upload/planing.jar';`

###　hive 控制台日志输出配置项
`${HIVE_HOME}/bin/hive -hiveconf hive.root.logger=DEBUG,console`

### hive 服务开启远程调试
- cli 服务开启远程调试: `${HIVE_HOME}/bin/hive --debug`
- beeline 服务开启远程调试: `${HIVE_HOME}/bin/beeline --debug`
- hiveserver2 服务开启远程调试: `${HIVE_HOME}/bin/hive --service hiveserver2 --debug`

### 设置 mr 任务中何时启动 reduce
set mapred.reduce.slowstart.completed.maps = 0.05

### 设置 mr 的 jvm 参数, 可解决 map 或 reduce 时的 OOM
- 同时设置 map 和 reduce 的 jvm 参数
set mapred.child.java.opts = "-Xmx200m"
- 分开设置map的jvm参数
set mapreduce.map.java.opts = "-Xmx200m -XX:+UseConcMarkSweepGC"
set mapred.map.child.java.opts = "-Xmx200m -XX:+UseConcMarkSweepGC"
- 分开设置reduce的jvm参数
set mapreduce.reduce.java.opts = "-Xmx200m -XX:+UseConcMarkSweepGC"
set mapred.reduce.child.java.opts = "-Xmx200m -XX:+UseConcMarkSweepGC"

>**参考:**

[Configuration Properties](https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties)
