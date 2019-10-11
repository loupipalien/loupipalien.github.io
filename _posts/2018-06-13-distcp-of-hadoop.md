---
layout: post
title: "Hadoop 的 DistCp"
date: "2018-06-13"
description: "Hadoop 的 DistCp"
tag: [hadoop]
---

### 使用

#### 基本用法

#### 更新与覆盖
-update 用于从源拷贝与目标中不存在或者是与目标版本不同的文件; -overwrite 将会覆盖目标中已存在的目标文件  
更新和覆盖选项保证指定的关注点, 因为它们默认使用一种精确的方式处理源路径变量; 考虑从 /source/first 和 /source/second 到 /target 的一个拷贝, 源路径有以下内容
```
hdfs://nn1:8020/source/first/1
hdfs://nn1:8020/source/first/2
hdfs://nn1:8020/source/second/10
hdfs://nn1:8020/source/second/20
```
当 DistCp 不带 -update 或 -overwrite 调用时, 会在 /target 下创建 first 和 second 目录
```
distcp hdfs://nn1:8020/source/first hdfs://nn1:8020/source/second hdfs://nn2:8020/target
```
在 /target 的目录下会产生以下内容
```
hdfs://nn2:8020/target/first/1
hdfs://nn2:8020/target/first/2
hdfs://nn2:8020/target/second/10
hdfs://nn2:8020/target/second/20
```
当指定了 -update 或 -overwrite, 源目录下的内容就会被拷贝到目标路径下, 而不再包括源目录本身
```
distcp -update hdfs://nn1:8020/source/first hdfs://nn1:8020/source/second hdfs://nn2:8020/target
```
在 /target 的目录下会产生以下内容
```
hdfs://nn2:8020/target/1
hdfs://nn2:8020/target/2
hdfs://nn2:8020/target/10
hdfs://nn2:8020/target/20
```
另外, 如果多个源目录中包含相同名称的文件 (假定文件名为 0), 源目录都将映射条目到目标 /target/0; DistCp 不允许这种冲突, 将会放弃此文件; 现在开了以下拷贝操作
```
distcp hdfs://nn1:8020/source/first hdfs://nn1:8020/source/second hdfs://nn2:8020/target
```
源文件以及大小
```
hdfs://nn1:8020/source/first/1 32
hdfs://nn1:8020/source/first/2 32
hdfs://nn1:8020/source/second/10 64
hdfs://nn1:8020/source/second/20 32
```
目标文件以及大小
```
hdfs://nn2:8020/target/1 32
hdfs://nn2:8020/target/10 32
hdfs://nn2:8020/target/20 64
```
影响
```
hdfs://nn2:8020/target/1 32
hdfs://nn2:8020/target/2 32
hdfs://nn2:8020/target/10 64
hdfs://nn2:8020/target/20 32
```
1 被跳过是因为文件长度和内容都符合; 2 被拷贝是因为它不存在与目标目录中; 10 和 20 都被覆盖是因为内容和源文件不符; 如果使用了 -update, 1 也会被覆盖

#### 原生命名空间扩展属性保留
这一节仅适用于 HDFS; TODO

##### 命令行参数

##### 其他控制参数
使用 -D 设置
|参数名|说明|
|--|--|
|distcp.bytes.per.map|distcp 时一个 map 处理的字节数|
|mapreduce.job.queuename|指定 MR 作业队列|
|mapreduce.job.name|指定 MR 作业名称|
|dfs.client.socket-timeout|DFS 客户端套接字超时时间|
|ipc.client.connect.timeout|IPC 客户端链接超时时间|

#### DistCp 时 Map 的数量如何确定
DistCp 会尝试均分需要拷贝的内容, 这样每个 map 拷贝差不多相等大小的内容; 因为文件时最小的拷贝粒度, 所以配置增加 Map 数目不一定会增加实际并行运行性的 Map 数及总吞吐量; 如果没有使用 -m 选项, distcp 会尝试在在调度工作时指定 map 数目为 min(total_bytes/bytes.per.map, 20 * num_task_trackers), 其中 bytes.per.map 默认是 256 MB

>**参考:**
[DistCp Version2 Guide](http://hadoop.apache.org/docs/r2.7.3/hadoop-distcp/DistCp.html#Command_Line_Options)  
[使用distcp命令跨集群传输数据](https://blog.csdn.net/levy_cui/article/details/53404966)
