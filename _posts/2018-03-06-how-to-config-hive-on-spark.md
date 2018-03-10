---
layout: post
title: "如何配置 Hive On Spark"
date: "2018-03-06"
description: "如何配置 Hive On Spark"
tag: [hive, spark]
---

**环境:** CentOS-6.8-x86_64, hive-2.1.1, spark-1.6.0

### 版本兼容
Hive on Spark 仅在特定版本的 Spark 上测试过, 所以对于给定版本的 Hive 仅保证在指定 版本的 Spark 工作; 其他版本的 Spark 也许可以和给定版本的 Hive 工作, 但是并不保证; 以下是 Hive 版本列表和它们对应兼容 Spark 版本

|Hive Version|Spark Version|
|-|-|
|master|2.2.0|
|2.3.x|2.0.0|
|2.2.x|1.6.0|
|2.1.x|1.6.0|
|2.0.x|1.5.0|
|1.2.x|1.3.1|
|1.1.x|1.2.0|

### Spark 安装
以下是 Spark 安装手册: [YARN Mode](http://spark.apache.org/docs/latest/running-on-yarn.html ), [Standalone Mode](https://spark.apache.org/docs/latest/spark-standalone.html); Hive on Spark 默认支持 Spark on YARN 模式  
对于安装执行需要遵循以下条例:
- 安装 Spark (无论是下载预编译的 Spark 还是从源码编译的)
  - 安装 / 编译兼容版本; Hive 的根 pom.xml 中定义的 <spark.version> 是编译测试过的 Spark 版本号
  - 安装 / 编译兼容分支; 每个版本的 Spark 都有几个版本, 对应不同版本的 Hadoop
  - 一旦 Spark 安装, 找到 <spark-assembly-\*.jar> 的位置
  - 注意, 必须有一个不包含 Hive jars 版本的 Spark, 意味着不编译 Hive profile; 如果你使用 Parquet 表, 推荐开启 "parquet-provided" profile, 否则将会有 Parquet 的依赖冲突; 从安装中移除 Hive jars, 使用以下命令在你的 Spark 代码库执行就可以
    - Spark 2.0.0 以前
    ```
    ./make-distribution.sh --name "hadoop2-without-hive" --tgz "-Pyarn,hadoop-provided,hadoop-2.4,parquet-provided"
    ```
    - Spark 2.0.0 之后
    ```
    ./dev/make-distribution.sh --name "hadoop2-without-hive" --tgz "-Pyarn,hadoop-provided,hadoop-2.7,parquet-provided"
    ```
  -  编译成功后将 spark-1.6.0-bin-hadoop2-without-hive.tgz 安装, 具体步骤可以参考 [如何部署 spark 集群](2017-11-05-how-to-deploy-spark-cluster.md)
- 启动 Spark 集群
找到 <Spark Master URL>, 可以访问 Spark master WebUI  

### 配置 YARN
使用 [fair scheduler](https://hadoop.apache.org/docs/r2.7.1/hadoop-yarn/hadoop-yarn-site/FairScheduler.html) 而不是 [capacity scheduler], 为 YARN 集群上的作业公平的分布相同的资源份额, 在 yarn-site.xml 中配置
```
<property>
    <name>yarn.resourcemanager.scheduler.class</name>
    <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler</value>
</property>
```

### 配置 Hive
- 添加 Spark 依赖到 Hive 中
  - Hive 2.2.0 之前, 将 spark-assembly jar 链接或拷贝到 ${HIVE_HOME}/lib 中
  - Hive 2.2.0 之后, Hive on Spark 运行在 Spark 2.0.0 或更高版本, 则没有 assembly jar
    - 在 YARN 模式 (无论是 yarn-client 还是 yarn-cluster) 运行时, 链接或拷贝以下 jars 到 ${HIVE_HOME}/lib 中
      - scala-library
      - spark-core
      - spark-network-common
    - 在本地模式 (仅用于 debugging) 运行时, 链接或拷贝上述 jars 以下 的 jars 到 ${HIVE_HOME}/lib 中
      - chill-java  chill  jackson-module-paranamer  jackson-module-scala  jersey-container-servlet-core
      - jersey-server  json4s-ast  kryo-shaded  minlog  scala-xml  spark-launcher
      - spark-network-shuffle  spark-unsafe  xbean-asm5-shaded
- 使用 Spark 配置 Hive 执行引擎
  ```
  set hive.execution.engine=spark;
  ```
  Hive 和 远程 Spark Driver 的其他配置属性见 [Spark section of Hive Configuration Properties](https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties#ConfigurationProperties-Spark)
- 为 Hive 设置 Spark 应用的配置, 更多设置见 [Spark configuration](http://spark.apache.org/docs/latest/configuration.html); 这个可以通过添加设置这些属性的 "spark-default.conf" 文件到 Hive 的 classpath 实现, 也可以通过设置这些属性到 Hive 配置中实现; 例如:
  ```
  set spark.master=<Spark Master URL>
  set spark.eventLog.enabled=true;
  set spark.eventLog.dir=<Spark event log folder (must exist)>
  set spark.executor.memory=512m;             
  set spark.serializer=org.apache.spark.serializer.KryoSerializer;
  ```
  **配置属性细节**:
  TODO...
- 允许 Yarn 去缓存必要的 spark 依赖的 jars 到节点上, 这样当应用启动时不需要每次去分发
  - Hive 2.2.0 之前, 上传 spark-assembly jar 到 hdfs, 并将以下配置添加到 hive-site.xml 中
  ```
  <property>
      <name>spark.yarn.jar</name>
      <value>hdfs://xxxx:8020/spark-assembly.jar</value>
  </property>
  ```
  - Hive 2.2.0, 上传在 ${SPARK_HOME}/jars 中的 jars 到 hdfs, 并且将以下配置添加到 hive-site.xml 中
  ```
  <property>
      <name>spark.yarn.jars</name>
      <value>hdfs://xxxx:8020/spark-jars/* </value>
  </property>
  ```
### 配置 Spark
设置执行器内存大小比简单的将其设置为尽可能的大更难; 这有几个点需要被考虑
- 更多的执行器内存意味着它能够为更多的查询开启 mapjoin 优化
- 另一方面, 从 GC 的角度来看, 更多的执行器内存会更难处理
-  一些经验表明, HDFS 客户端不能很好的处理并发写, 如果执行器内核很多将面临着竞争条件

 以下设置需要为集群调优, 这些也可以在 Hive on Spark 之外提交的 Spark 作业应用

 |属性|推荐值|
 |-|-|
 |spark.executor.cores|5-7 之间, 见性能调节章节|
 |spark.executor.memory|yarn.nodemanager.resource.memory-mb * (spark.executor.cores / yarn.nodemanager.resource.cpu-vcores)|
 |spark.yarn.executor.memoryOverhead|spark.executor.memory 的 15-20%|
 |spark.executor.instances|依赖于 spark.executor.memory 和 spark.yarn.executor.memoryOverhead, 见性能调节章节|

#### 性能调节
TODO...

### 安装中遇到的报错
- 在 windows 下使用 idea 复杂且麻烦, 建议使用在 linux 机器上使用命令行编译, 或者在 windows 下的 bash (例如 Git 带的) 环境编译
- spark 启动时可能报错, 在 master 节点上找不到 `org/apache/hadoop/conf/Configuration` 类, 在 worker 节点上找不到
`org/slf4j/Logger` 类, 这是因为编译出来的安装包中没有 hadoop 相关的包, 需要在 spark-env.sh 中配置进去, 根据参考文章解决
- spark-default.conf 和 hive-site.xml 中只要配置一方即可; 在 spark-default.conf 中配置时需要将其加载到 Hive 的 classpath 中, 所以建议在 hive-site.xml 中配置; 其中 spark.master 建议先配置为 spark://${HOSTNAME}:7077, 个人在配置为 yarn-client 时跑 sql 时报错:
`ql.Driver: FAILED: Execution Error, return code 2 from org.apache.hadoop.hive.ql.exec.spark.SparkTask`; 在各种参考文章中配置为 yarn-client 不知为何, 暂未解决...


>**参考:**
[Hive on Spark: Getting Started](https://cwiki.apache.org/confluence/display/Hive/Hive+on+Spark%3A+Getting+Started)  
[Hive on Spark安装配置详解及避坑指南](https://www.jianshu.com/p/a7f75b868568)  
[利用IDEA工具编译Spark源码(1.60~2.20)](http://blog.csdn.net/he11o_liu/article/details/78739699)   
[Spark集群启动报错：NoClassDefFoundError: org/apache/hadoop/conf/Configuration](http://f.dataguru.cn/spark-659076-1-1.html)  
[Using Spark's "Hadoop Free" Build](https://spark.apache.org/docs/latest/hadoop-provided.html)  
[Spark记录-源码编译spark2.2.0](http://www.cnblogs.com/xinfang520/p/7763328.html)  
[Linux搭建Hive On Spark环境](http://blog.csdn.net/pucao_cug/article/details/72773564)
