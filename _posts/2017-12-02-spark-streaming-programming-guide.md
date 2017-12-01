---
layout: post
title: "Spark Streaming 编程指南 (2.2.0)"
date: "2017-12-02"
description: "Spark Streaming 编程指南"
tag: [spark]
---

### 概述
Spark Streaming 是核心 Spark API 的拓展, 支持可伸缩, 高吞吐, 可容错的实时数据流处理; 可以处理来自如 Kafka, Flume< Kinesis， 或者 TCP 套接字的多种源数据, 可以被使用例如 map, reduce, join 和 window 等高阶函数表达的复杂算法处理; 最后, 处理的数据可以被推送到文件系统, 数据库, 实时仪表盘等; 事实上, 也可以在数据流上应用机器学习和图形处理算法
[!image](#)
在内部其工作原理如下, Spark Streaming 接受实时输入数据流并切分成多批, 然后被 Spark 引擎处理后批量生成最终的结果流
[!image](#)
Spark Streaming 提供一个高阶的抽象被称为离散流或 DStreams, 它代表一个持续的数据流; DStreams 可以从例如 Kafka, Flume, Kinesis等源数据数据流创建, 或者通过应用高阶的操作其他 DStreams; 在内部, 一个 DStreams 被表示为一个序列的 RDDs  
本指南将展示如何使用 DStreams 开始写 Spark Streaming 程序; 可以使用 Sacla, Java, 或 Python (在 Spark 1.2 引入) 写 Spark Streaming 程序, 都会被展示在本指南中; 可以在本指南中找到选项卡, 可以在不同的语言代码段之间选择  
**注意:** 有少量 APIs 在 Python 中是不同的或不可用, 在本文中, 将会发现用 `Python API` 高亮标注了这些不同
### 一个简单的例子
在讨论如何写你自己的 Spark Streaming 程序的细节之前, 一起来瞥一眼一个简单的 Spark Streaming 程序是什么样子的; 假设我们要计算从监听 TCP 套接字的数据服务器接受的文本数据中的单词数量
, 所需要做的如下
- Scala
- Java
首先, 我们创建一个 [JavaStreamingContext](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html) 对象, 它是整个所有流功能的主入口; 我们可以创建一个有两个线程的本地 StreamingContext, 并且批量间隔为 1 秒
```
import org.apache.spark.*;
import org.apache.spark.api.java.function.*;
import org.apache.spark.streaming.*;
import org.apache.spark.streaming.api.java.*;
import scala.Tuple2;

// Create a local StreamingContext with two working thread and batch interval of 1 second
SparkConf conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount");
JavaStreamingContext jssc = new JavaStreamingContext(conf, Durations.seconds(1));
```
使用这个 context, 我们可以创建一个表示来自一个 TCP 源的流数据的 DStreams, 指定一个主机名 (例如: localhost) 和端口 (例如: 9999)
```
// Create a DStream that will connect to hostname:port, like localhost:9999
JavaReceiverInputDStream<String> lines = jssc.socketTextStream("localhost", 9999);
```
这个 `lines` DStream 代表将从数据服务器接受的流数据, 此流中的每一条记录是文本中的一行, 然后我们使用空格将文本行切割成词
```
// Split each line into words
JavaDStream<String> words = lines.flatMap(x -> Arrays.asList(x.split(" ")).iterator());
```
`flatMap` 是一个 DStream 操作, 可以通过从源 DStream 中的每条记录生成多个新记录创建一个新的 DStream; 在这个例子中每一行就会被切分成多个词, `words` DStream 表示词的流; 注意, 我们使用一个 [FlatMapFunction](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.api.java.function.FlatMapFunction) 对象定义这个转换, 就像我们在使用中发现的那样, 在 Java API 中有很多这样方便的类帮助定义 DStream 转换  
接下来, 我们来统计这些词
```
// Count each word in each batch
JavaPairDStream<String, Integer> pairs = words.mapToPair(s -> new Tuple2<>(s, 1));
JavaPairDStream<String, Integer> wordCounts = pairs.reduceByKey((i1, i2) -> i1 + i2);

// Print the first ten elements of each RDD generated in this DStream to the console
wordCounts.print();
```
使用一个 [PairFuntion](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.api.java.function.PairFunction) 将这个 `words` DStream 进一步映射 (一对一的转换) 为一个键值对 (word, 1) 的 DStream; 然后, 它将使用一个 [Function2](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.api.java.function.Function2) 对象去归纳获取每个批次数据中词的频次; 最终, `wordCounts.print()` 将会打印出每秒生成的一些统计  
注意, 当这些代码行被执行事, Spark Streaming 仅设置了在它启动后将执行的计算, 还没有开始真正的处理; 在所有转换都被设置后开始处理, 我们最后调用 `call` 方法
```
// Start the computation
jssc.start();
// Wait for the computation to terminate       
jssc.awaitTermination();
```
全部的代码可以在 Spark Streaming 的例子 [JavaNetWorkCount](https://github.com/apache/spark/blob/v2.2.0/examples/src/main/java/org/apache/spark/examples/streaming/JavaNetworkWordCount.java) 中找到  
- Pyhton

如果你已经下载并构建了 Spark, 可以按照如下方式运行例子; 你首先需要运行 NetCat (一个可以在大多数类 Unix 系统中找到的小工具) 作为被使用的数据服务器
```
$ nc -lk 9999
```

### 基础概念
- 链接
 初始化 SparkContext,
- 离散流 (DStreams)
- 输入 DStreams 和 接收器
- DStreams 的转换
- DStreams 的输出操作
- 数据帧和 SQL 操作
- MLlib 操作
- 缓存 / 持久化
- 检查点
- 累加器, 广播变量, 检查点
- 部署应用
- 监控应用
### 性能调节
- 减少批处理时间
- 设置合适的批量间隔
- 内存调节
### 容错语义
### 从这里去哪儿
