---
layout: post
title: "Spark 编程指南 (2.2.0)"
date: "2017-11-30"
description: "Spark 编程指南"
tag: [spark]
---

### 概述

### 连接 Spark

### 初始化 Spark
- 使用 Shell

### 弹性分布式数据集 (RDDs)
- 并行化集合
- 外部数据集
- RDD 操作
  - 基础
  - 传递函数到 Spark
  - 理解必包
    - 例子
    - 本地模式 vs 集群模式
    - 打印一个 RDD 的元素
  - 使用键值对
  - 转化  
  以下表格中列举了一些 Spark 支持的常见转化, 详情参考 RDD API 文档 ([Scala](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.RDD), [Java](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/api/java/JavaRDD.html), [Pyhton](https://spark.apache.org/docs/latest/api/python/pyspark.html#pyspark.RDD), [R](https://spark.apache.org/docs/latest/api/R/index.html)) 和键值对 RDD 函数文档 ([Scala](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions), [Java](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/api/java/JavaPairRDD.html))

  |转换|说明|
  |-|-|
  |map(func)|传递源中的每个元素通过函数 func, 返回由此形成的一个新的分布式数据集|
  |filter(func)|选择源中通过 func 函数返回为真的元素, 返回由此形成的一个新的分布式数据集|
  |flatMap(func)|类似于 map, 但每个输入项可以被映射到 0 或 多个输出项 (所以 func 应该返回一个序列而不是单个项)|
  |mapPartitions(func)|-|
  |mapPartitionsWithIndex(func)|-|
  |sample(withReplacement, fraction, seed)|-|
  |union(otherDataset)|返回一个源数据集和参数数据集元素并集的新数据集|
  |intersection(otherDataset)|返回一个源数据集和参数数据集元素交集的新数据集|
  |distinct([numTasks])|返回一个包含源数据集不同元素的新 RDD|
  |groupByKey([numTasks])|-|
  |reduceByKey(func, [numTasks])|-|
  |aggregateByKey(zeroValue)(seqOp, combOp, [numTasks])|-|
  |sortByKey([ascending], [numTasks])|-|
  |join(otherDataset, [numTasks])|-|
  |cogroup(otherDataset, [numTasks])|-|
  |cartesian(otherDataset)|-|
  |pipe(command, [envVars])|-|
  |caalesce(numPartitions)|-|
  |repartition(numPartitions)|-|
  |repartitionAndSortWithinPartitons(partitioner)|-|

  - 处理  
  以下表格中列举了一些 Spark 支持的常见处理, 详情参考 RDD API 文档 ([Scala](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.RDD), [Java](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/api/java/JavaRDD.html), [Pyhton](https://spark.apache.org/docs/latest/api/python/pyspark.html#pyspark.RDD), [R](https://spark.apache.org/docs/latest/api/R/index.html)) 和键值对 RDD 函数文档 ([Scala](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions), [Java](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/api/java/JavaPairRDD.html))

  |处理|说明|
  |reduce(func)|使用一个函数 func 聚合数据集中的元素 (函数使用两个参数返回一个值), 这个函数应该具有交换性和结合性, 这样才能在并行中计算正确|
  |collect()|将数据集中的所有元素作为一个数组返回给驱动程序, 在过滤或其他返回重要的规模小的数据子集的操作后使用是非常有用的|
  |count()|返回数据集中元素的个数|
  |first()|返回数据集中第一个元素 (类似于 take(1))|
  |take(n)|将数据集中的前 n 个元素作为一个数组返回|
  |takeSample(withReplacement, num, [seed])|-|
  |takeOrdered(n, [ordering])|-|
  |saveAsTextFile(path)|-|
  |saveAsSequenceFile(path)(Java or Scala)|-|
  |countByKey()|-|
  |foreach(func)|-|

  - 混洗操作
    - 背景
    - 性能影响
- RDD 持久化
  - 选择哪一个存储级别
  - 移除数据

### 共享变量
- 广播变量
- 累加器

### 部署集群

### 用 Java/Scala 发布 Spark 作业

### 单元测试

### 从这儿到哪里去
