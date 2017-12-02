---
layout: post
title: "Spark Streaming 编程指南 (2.2.0)"
date: "2017-12-02"
description: "Spark Streaming 编程指南"
tag: [spark]
---

### 概述
Spark Streaming 是核心 Spark API 的拓展, 支持可伸缩, 高吞吐, 可容错的实时数据流处理; 可以处理来自如 Kafka, Flume, Kinesis， 或者 TCP 套接字的多种源数据, 可以被使用例如 map, reduce, join 和 window 等高阶函数表达的复杂算法处理; 最后, 处理的数据可以被推送到文件系统, 数据库, 实时仪表盘等; 事实上, 也可以在数据流上应用机器学习和图形处理算法  
[!image](#)  

在内部其工作原理如下, Spark Streaming 接受实时输入数据流并切分成多批, 然后被 Spark 引擎处理后批量生成最终的结果流  
[!image](#)  

Spark Streaming 提供一个高阶的抽象被称为离散流或 DStreams, 它代表一个持续的数据流; DStreams 可以从例如 Kafka, Flume, Kinesis等源数据数据流创建, 或者通过应用高阶的操作其他 DStreams; 在内部, 一个 DStreams 被表示为一个序列的 RDDs  
本指南将展示如何使用 DStreams 开始写 Spark Streaming 程序; 可以使用 Sacla, Java, 或 Python (在 Spark 1.2 引入) 写 Spark Streaming 程序, 都会被展示在本指南中; 可以在本指南中找到选项卡, 可以在不同的语言代码段之间选择  
**注意:** 有少量 APIs 在 Python 中是不同的或不可用, 在本文中, 你将会发现用 `Python API` 高亮标注了这些不同  

### 一个简单的例子
在讨论如何写你自己的 Spark Streaming 程序的细节之前, 一起来瞥一眼一个简单的 Spark Streaming 程序是什么样子的; 假设我们要计算从监听 TCP 套接字的数据服务器接受的文本数据中的单词数量, 所需要做的如下  
- Pyhton
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

如果你已经下载并构建了 Spark, 可以按照如下方式运行例子; 你首先需要运行 NetCat (一个可以在大多数类 Unix 系统中找到的小工具) 作为被使用的数据服务器
```
$ nc -lk 9999
```
然后, 在另一个终端, 你可以启动这个例子
- Pyhton
- Scala
- Java

```
$ ./bin/run-example streaming.NetworkWordCount localhost 9999
```
然后, 任何被输入在运行 netcat 服务器终端中行将会被每秒统计和输出在屏幕上一次, 看起来就像以下这样
- Python
- Scala
- Java

|terminal one (Netcat)|terminal two (JavaNetworkWordCount)|
|:--|:--|
|$ nc -lk 9999 <br> hello world  <br> ...|$ ./bin/run-example streaming.NetworkWordCount localhost 9999 <br> ... <br>  ------------------------------------------- <br> Time: 1357008430000 ms <br> ------------------------------------------- <br> (hello,1) <br> (world,1) <br> ...|

### 基础概念
接下来, 我们越过这个简单的例子, 详尽的介绍 Spark Streaming 的基础知识
- **链接**  
类似于 Spark, Spark Streaming 也可以从 Maven 中央仓库中获得; 为了写出你自己的 Spark Streaming 程序, 你需要将以下的依赖添加到你的 SBT 或 Maven 项目中
  - SBT

  ```
  libraryDependencies += "org.apache.spark" % "spark-streaming_2.11" % "2.2.0"
  ```
  - Maven

  ```
  <dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming_2.11</artifactId>
    <version>2.2.0</version>
  </dependency>
  ```

对于处理来自于例如 Kafka, Flume, Kinesis 等数据源的数据并没有在核心 Spark Streaming API 中, 你需要添加对应的组件包 `spark-streaming-xyz-2.11` 到依赖中, 如下是一些常见的示例

|Source|Artifact|
|-|-|
|Kafka|spark-streaming-kafka-0-8_2.11|
|Flume|spark-streaming-flume_2.11|
|Kinesis|spark-streaming-kinesis-asl_2.11 [Amazon Software License]|

最新列表, 请参考 [Maven repository](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22org.apache.spark%22%20AND%20v%3A%222.2.0%22) 以获得支持的所有源和组件列表
- **初始化 SparkContext**    
为了初始化一个 Spark Streaming 程序, 需要创建一个 StreamingContext 的实例, 它是所有 Spark Streaming 功能的主入点
  - Python
  - Scala
  - Java

  可以从 [SparkConf](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.SparkConf) 实例中创建一个 [JavaStreamingContext](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.StreamingContext)

  ```
  import org.apache.spark.*;
  import org.apache.spark.streaming.api.java.*;
  
  SparkConf conf = new SparkConf().setAppName(appName).setMaster(master);
  JavaStreamingContext ssc = new JavaStreamingContext(conf, new Duration(1000));
  ```
  `appName` 参数是你的应用展示在集群 UI 上的名字, `master` 是一个 [Spark, Mesos, YARN](https://spark.apache.org/docs/latest/submitting-applications.html#master-urls) 集群的 URL, 或者是运行在本地模式的特定的 "local[\*]" 字符串; 特别的, 当运行在一个集群上时, 你不会想硬编码 `master` 在程序中, 而是在使用 `spark-submit` 发布应用时在那里接受它; 然而, 对于本地测试或单元测试, 你可以传递 "local[\*]" 去运行 Spark Stremng 程序; 注意这样内部创建的一个 [JavaSparkContext](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/api/java/JavaSparkContext.html) (Spark 功能的起始点), 它可以作为 `ssc.SparkContext` 被访问
  批量间隔必须基于你应用的延迟要求和可用的集群资源设置, 更多细节见 [Performance Tuning](https://spark.apache.org/docs/latest/streaming-programming-guide.html#setting-the-right-batch-interval) 章节
  `JavaStreamingContext` 实例也可以从一个已存在的 `JavaSparkContext` 实例创建
  ```
  import org.apache.spark.streaming.api.java.*;

  JavaSparkContext sc = ...   //existing JavaSparkContext
  JavaStreamingContext ssc = new JavaStreamingContext(sc, Durations.seconds(1));
  ```
  在一个 `context` 被创建后, 你需要做如下的事
    - 通过创建输入 DStreams 定义输入源
    - 通过应用转换和在 DStreams 上的输出操作定义流计算
    - 使用 `streamingContext.start()` 开始接受数据并处理
    - 使用 `streamingContext.awaitTermination` 等待处理停止 (手动或由于任何错误)
    - 处理过程可以通过使用 `streamingContext.stop()` 手动的停止

  **记忆要点:**
    - 一旦一个 `context` 被启动, 不能有新的流计算被设置或加入它
    - 一旦一个 `context` 上下文被停止, 它不能被重新启动
    - 在同一时间只能有一个活跃的 `StreamingContext` 在一个 JVM 上
    - 在 `StreamingContext` 调用 `stop()` 同时也停止了 `SparkContext`; 为了仅停止 `StreamingContext`, 需设置 `stop()` 的可选参数 `stopSparkContext` 为 `false`
    - 一个 `SparkContext` 可以被复用去创建多个 `StreamingContext`, 只要在创建下一个 `StreamingContext` 前上一个 `StreamingContext` 被停止了 (没有停止 `SparkContext`)
- **离散流 (DStreams)**
- **输入 DStreams 和 接收器**
- **DStreams 的转换**
- **DStreams 的输出操作**
- **数据帧和 SQL 操作**
- **MLlib 操作**
- **缓存 / 持久化**
- **检查点**
- **累加器, 广播变量, 检查点**
- **部署应用**
- **监控应用**

### 性能调节
- 减少批处理时间
- 设置合适的批量间隔
- 内存调节

### 容错语义

### 从这里去哪儿

>**参考:**  
[Spark Streaming Programming Guide](https://spark.apache.org/docs/latest/streaming-programming-guide.html)
