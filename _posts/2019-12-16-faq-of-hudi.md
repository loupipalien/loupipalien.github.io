---
layout: post
title: "Hudi 的 FAQ"
date: "2019-12-16"
description: "Hudi 的 FAQ"
tag: [hudi]
---
### Hudi 的 FAQ

#### 常规
##### Hudi 对我或者我的组织有什么帮助
如果你正在寻求在 HDFS 或云存储上快速摄取数据, Hudi 可以为你提供帮助的工具; 还有, 如果你有消耗大量资源的 ETL/hive/spark 作业, Hudi 通过提供一种增量读写数据的方法潜在的提供帮助  
作为组织, Hudi 可以帮助你构建一个[有效的数据湖](https://docs.google.com/presentation/d/1FHhsvh70ZP6xXlHdVsAI0g__B_6Mpto5KQFlZ0b8-mM), 可以解决一些非常复杂, 低端存储的管理问题, 更快的将数据递到数据分析师, 工程师, 科学家的手上

##### Hudi 不能够做什么
Hudi 并非为任何 OLTP 场景设计的, 这种情况下通常你使用的是已有的 NoSQL/RDBMS 数据存储; Huid 不能替代你的内存分析数据库 (至少 现在不能); Hudi 支持大约几分钟级别的近实时摄取, 并以延迟为代价来高效的批量摄取; 如果你确实希望小于分钟级别的处理延迟, 那么请坚持使用你最喜欢的流处理解决方案

##### 什么是增量处理? 为什么 Hudi 的 docs/talks 总是在提及它
增量处理最先是 `@Vinoth Chandar` 在 O'reilly 的 [博客](https://www.oreilly.com/ideas/ubers-case-for-incremental-processing-on-hadoop) 中提出的, 这也引出了大量的工作; 用纯技术的术语说, 增量处理仅是以流处理风格编写小批量程序; 典型的批量作业每隔几个小时就消费所有输入以及重计算所有输出一次; 典型的流处理作业持续的或每隔几秒钟消费新输入以及重计算新的或变化的去输出; 虽然以批量的方式冲计算所有输出更为简单, 但它这很浪费昂贵的资源; Hudi 有能力用流风格编写同一批量管道, 并每隔几分钟运行一次  
虽然我们仅将它称为流处理, 但也称其为增量处理, 以区别于使用 Apache Flink, Apache Apex 或者 Apache Kafka Stream 构建的纯流处理管道

##### 写时复制 (COW) 和 读时合并 (MOR) 存储类型有什么区别
**写时复制:** 这个存储类型可以使客户端摄取数据到列文件格式, 如 parquet; 任何新的数据都会使用 COW 存储类型写入 Hudi 数据集, 并且是写成新的 parquet 文件; 更新一个已存在的行集合将会导致重写整个 parquet 文件, 包含被影响行的文件都会更新; 因此, 所有此类数据集的写入都会被 parquet 的写性能限制, 越大的 parquet 文件会花费越多的时间拿到摄取的数据  
**读时合并:** 这个存储类型可以客户端快速摄取数据到行文件格式, 例如 avro; 任何新数据都会使用 MOR 表类型写入 Hudi 数据集, 并且是写成存储 avro 编码字节数据的新 log/delta 文件; 一个压缩过程 (可配置为内联或者异步的) 将会转换日志文件格式为列文件格式 (parquet); 两种不同的 `InputFormat` 暴露了数据的两种不同视图, 读优化视图展示列文件的读性能, 而实时视图展示列文件和日志文件的读性能; 更新一个已存在的行数据将导致其中一件事发生
- 基于之前压缩产生的 parquet 文件, 进一步压缩 log/delta 文件
- 当之前没有任何压缩过程发生时则写更新到一个 log/delta 文件

所以, 所有此类数据集的写入被 avro/log 文件的写入性能限制, 这比 parquet 快很多; 尽管, 读取 log/delta 文件对比列文件 (parquet) 付出的代价更高  
更多细节可在[这里](https://hudi.apache.org/concepts.html#storage-types--views)找到

##### 对于我的工作负载如何选择存储类型
Hudi 的一个关键目标是提供 `upsert` 功能, 这比重写整个表或者分区快几个数量级  
如果是以下情况时选择写时复制
- 你寻求一个简单的选项, 可以替代你已存在的 parquet 表而对实时数据没有需求
- 你当前的工作是为处理更新为重写整个表或分区, 然而在每个分区实际上只有少部分文件更改了
- 你乐于保持事务操作更简单 (无压缩等), 可以接受摄取/写入性能被 [parquet 文件大小](https://hudi.apache.org/configurations.html#limitFileSize) 限制, 以及更新被影响或污染的此类文件数量限制
- 你的工作负载是相对平滑的并且没有突然的爆发大量的更新或插入到旧分区中; COW 将所有的合并成本汇聚在写入端, 因此这样突然的更改会堵塞你的摄取以及影响到满足正常模型摄取的延迟目标

如果是以下情况时选择读时合并
- 你想数据尽可能快的被摄取以及尽可能多的被查询
- 你的工作负载可能会突然变化 (例如: 在上流数据库中大量旧分区的更新导致了 DFS 上旧分区的大量更新); 异步压缩可以帮助分摊引起这种场景的写放大, 并且正常的摄取也可以更得上上游的变化

与你的选择无关的, Hudi 提供的
- 快照隔离以及批量记录的原子性写入
- 增量拉取
- 数据去重的能力

更多见[这里](https://hudi.apache.org/concepts.html#storage-types)

##### Hudi 是否是一个分析型数据库
一个典型的数据库会有一堆长时间运行的负责写和读的存储服务器; Hudi 的架构是非常不同的并且是有充分理由的; 它是高度的写和读/查分离的, 可以被独立的扩展以处理大规模的挑战; 所以它也许看起来并不总像一个数据库  
尽管, Hudi 被设计的非常像一个数据库, 并且提供了类似的功能 (upserts, change capture) 以及语义 (事务写入, 快照隔离读取)

##### 我如何对存储在 Hudi 中的数据建模
当写入数据到 Hudi 时, 你可以像你在一个键值从年初上指定一个键字段 (对单个分区/数据集是唯一的), 一个分区字段 (表示键将要放入的分区) 以及指定如何对批量写入数据记录的去重处理的 preCombine/combine 逻辑来为记录建模; 这个模型可以使 Hudi 执行主键约束就像你在数据库上获得的那样; 示例见[这里](https://hudi.apache.org/writing_data.html#datasource-writer)

##### Hudi 是否支持云存储/对象存储
是; 一般来说, Hudi 能够在任何 Hadoop 文件系统实现上提供它的功能, 因此可以在[云存储](https://hudi.apache.org/configurations.html#talking-to-cloud-storage) (Amazon S3 或 Microsoft Azure 或 Google Cloud Storage) 上读写数据集; 随着时间, Hudi 会结合特定的设计概念使得在云上构建 Hudi 数据集更加容易, 例如 [s3 的文件一致性检查](https://hudi.apache.org/configurations.html#withConsistencyCheckEnabled), 零移动, 重命名等

##### Hudi 支持什么版本的 Hive/Spark/Hadoop
到 2019 年九月, Hudi 可以支持 Spark 2.1+, Hive 2.x, Hadoop 2.7+ (不支持 Hadoop 3)

##### Hudi 是如何在一个数据集中存储数据的
在较高的层级上说, Hudi 基于 MVCC 设计将数据写为版本化的 parquet/base 文件以及包含了基础文件变化的日志文件; 所有文件都以数据集的分区模式存储, 这与 Apache Hive 表在 DFS 上的布局很相似; 更多细节见[这里](https://hudi.apache.org/concepts.html#file-management)

#### 使用 Hudi
##### 有什么方法写入一个 Hudi 数据集
典型的, 你从源获取了部分更新/插入一个集合, 并且触发了对一个数据集的[写操作](https://hudi.apache.org/writing_data.html#write-operations); 如果你从任何标准的源摄取数据, 例如 Kafka 或拖尾 DFS, [delta streamer](https://hudi.apache.org/writing_data.html#deltastreamer) 工具是非常有用的并且提供了一种简单, 自管理的方法来获取数据写入 Hudi; 你也可以使用 Spark datasource API 编写你自己的代码从自定义源抓取数据或者使用一个 [Hudi datasource](https://hudi.apache.org/writing_data.html#datasource-writer) 来写入 Hudi

##### 如何部署一个 Hudi 作业
关于 Hudi 编写的好处是它像其他 spark 作业一样运行在 YARN/Mesos 或者 K8S 集群上; 所以你可以使用 Spark UI 查看写入操作

##### 我如何查询我刚写入的 Hudi 数据集
除非 Hive Sync 是开启的, 否则和其他数据源一样, 使用以上其中一个方法写入的 Hudi 数据集可以简单的通过 Spark 数据源查询
```Java
val hoodieROView = spark.read.format("org.apache.hudi").load(basePath + "/path/to/partitions/*")
val hoodieIncViewDF = spark.read().format("org.apache.hudi")
     .option(DataSourceReadOptions.VIEW_TYPE_OPT_KEY(), DataSourceReadOptions.VIEW_TYPE_INCREMENTAL_OPT_VAL())
     .option(DataSourceReadOptions.BEGIN_INSTANTTIME_OPT_KEY(), <beginInstantTime>)
     .load(basePath);
```
>限制: 注意, 当前从 Spark 数据源中读取实时视图是不支持的; 以下请使用 Hive 路径

如果在 [deltastreamer](https://github.com/apache/incubator-hudi/blob/master/docker/demo/sparksql-incremental.commands#L45) 工具或 [datasource](https://hudi.apache.org/configurations.html#HIVE_SYNC_ENABLED_OPT_KEY) Hive Sync 是开启的, 该数据集在 Hive 可以作为几个表使用, 也能够使用 HiveQL, Presto 或者 SparkSQL 读取; 详情见 [这里](https://hudi.apache.org/querying_data.html)

##### Hudi 在一个输入中如何处理重复记录的键
当在数据集触发一个 `upsert` 操作, 提供的批量记录包含了给定键的多个条目时, 这些条目会通过重复调用 [payload](https://hudi.apache.org/configurations.html#PAYLOAD_CLASS_OPT_KEY) 类的 [preCombine()](https://github.com/apache/incubator-hudi/blob/master/hudi-common/src/main/java/org/apache/hudi/common/model/HoodieRecordPayload.java#L38) 方法最终简化为一条; 默认的, 我们挑选最大值的记录, 最大值 (通过调用 `.compareTo()` 决定) 使用最后写入获胜的风格语义  
对于一个 `insert` 或者 `bulk_insert` 操作, 则没有 这样的 pre-combining 执行; 因此, 如果你的输入包含重复记录, 数据集也就包含重复记录; 如果你不想去重记录则可以触发 `upsert` 或者考虑在 [datasource](https://hudi.apache.org/configurations.html#INSERT_DROP_DUPS_OPT_KEY) 或 [deltastreamer](https://cwiki.apache.org/confluence/github.com/apache/incubator-hudi/tree/master/hudi-utilities/src/main/java/org/apache/hudi/utilities/deltastreamer/HoodieDeltaStreamer.java#L210) 指定去重输入的选项

##### 对如何将输入记录合并到存储上的记录我如何实现自己的逻辑
同上, `payload` 类定义了方法 ([combineAndGetUpdateValue()](https://github.com/apache/incubator-hudi/blob/master/hudi-common/src/main/java/org/apache/hudi/common/model/HoodieRecordPayload.java), [getInsertValue()](https://github.com/apache/incubator-hudi/blob/master/hudi-common/src/main/java/org/apache/hudi/common/model/HoodieRecordPayload.java)) 控制存储上的记录如何与到来的更新/插入生成写回存储的最终值

##### 使用 Hudi 我如何删除数据集上的记录
GDPR 已使删除成为每个人数据管理工具箱中的必备工具, Hudi 支持软删除和硬删除; 有关如何实际执行的细节见[这里](https://hudi.apache.org/writing_data.html#deletes)

##### 我如何迁移我的数据到 Hudi
Hudi 对于你的整个数据集一次性重写到 Hudi 中提供了内置支持, 可使用 hudi-cli 中的 `HDFSParquetImporter` 工具; 你也可以通过使用 Spark datasource APIs 进行简单的读写来实现; 一旦迁移, 写入可以使用[这里](https://cwiki.apache.org/confluence/display/HUDI/FAQ?focusedCommentId=138021762#FAQ-WhataresomewaystowriteaHudidataset)讨论的一般方法执行; 这个主题的细节讨论见[这里](https://hudi.apache.org/migration_guide.html), 包括部分迁移的方法

##### 我如何将 hudi 的配置传递给我的 spark 作业
覆盖 datasource 和低级 Hudi 写入客户端(内部调用 deltastreamer 和 datasource) 的 Hudi 配置选项在[这里](https://hudi.apache.org/configurations.html); 在任何工具上例如 DeltaStreamer 上调用 `--help` 会打印所有使用选项; 在写入客户端层定义了大量控制更新插入, 文件大小的行为的选项, 以下是我们如何传递它们到可用于写入数据的不同选项
- 对于 Spark 数据源, 你可以使用 `DeltaStreamer` API 的 `option` 来传递这些配置
```Java
inputDF.write().format("org.apache.hudi")
  .options(clientOpts) // any of the Hudi client opts can be passed in as well
  .option(DataSourceWriteOptions.RECORDKEY_FIELD_OPT_KEY(), "_row_key")
  ...
```
- 当直接使用 `HoodieWriteClient` 时, 你可以使用连接中提及的配置来构建 `HoodieWriteConfig` 对象
- 当使用 `HoodieDeltaStreamer` 工具进行摄取时, 你可以在 properties 文件中设置这些属性, 并将此文件作为命令行 `--props` 的参数传递

##### 我是否可以注册我的 Hudi 数据集到 Apache Hive metastore
是; 可以通过单独执行执行 [Hive Sync 工具](https://hudi.apache.org/writing_data.html#syncing-to-hive)或者使用在 [deltastreamer](https://github.com/apache/incubator-hudi/blob/master/docker/demo/sparksql-incremental.commands#L45) 工具或 [datasource](https://hudi.apache.org/configurations.html#HIVE_SYNC_ENABLED_OPT_KEY) 中的选项

##### Hudi 的索引工作是如何的 & 它们有什么益处 ?
索引组件是 Hudi 写入的关键部分, 它始终将给定的 recordKey 映射到 Hudi 内部的 fileGroup; 这能快速定位一个给定写入操作影响的或污染的文件组; Hudi 对于索引支持以下选项
- HoodieBloomIndex (default): 使用布隆过滤器将放置在 parquet/base 文件 (以及不久的日志文件) 页脚的信息分类
- HoodieGlobalBloomIndex: 默认索引仅在单个分区内强制键的唯一性： 例如用户期盼知道给定记录被存储的分区; 这甚至可以帮助[非常大的数据集](https://eng.uber.com/uber-big-data-platform/)很好的建立索引
- HBaseIndex: Apache HBase 是一个键值存储, 通常建立于 HDFS 之上; 你可以在 HBase 中存储索引, 如果你早已经熟悉 HBase 这将会很方便

如果你喜欢你可以实现自己的索引, 通过继承 HoodieIndex 并配置索引类名到配置中

##### Hudi cleaner 是做什么
Hudi cleaner 进程通常在一个 commit 或 deltacommit 之后运行, 用于删除不在需要的旧文件; 如果你使用增量拉取特性, 确保你配置 cleaner [保留足够多的 commits](https://hudi.apache.org/configurations.html#retainCommits) 用于回放; 另一个考虑是为你的长时间运行才能完成的作业提供足够的时间; 否则, cleaner 可能删除一个正在执行或者准备执行的作业需要读取的文件, 这将导致作业
失败; 典型的, 对于每隔 30 分钟运行一次的摄取默认配置 10, 以保留 5 个小时的数据; 如果你的摄取运行非常频繁或者你想给查询更多的运行时间, 考虑增加这个配置值: `hoodie.cleaner.commits.retained`

##### Hudi 的 schema 演变过程是什么
Hudi 对记录使用 Avro 作为内部规范表示, 主要是由于它良好的 [schema 的兼容性和演变](https://docs.confluent.io/current/schema-registry/avro.html)属性; 这是你的摄取或 ETL 管道可靠性的关键因素; 只要 s传递到 Hudi 的 schema (无论是在 DeltaStreamer 的 schema provider 配置中显式指定, 还是通过 Spark 数据源的数据集  schemas 隐式指定) 是向后兼容的 (例如: 无字段删除, 只有新字段增加到 schema), Hudi 可以无缝的处理读写新老数据并保持 Hive schema 的更新

##### 我如何为一个 MOR 数据集运行压缩
在 MOR 数据集上运行压缩最简单的方式是运行 [压缩内联](https://hudi.apache.org/configurations.html#withInlineCompaction), 这会使摄取花费更多的时间; 在通常情况下, 当你有少量的迟到数据流入旧分区时, 这可能特别有用; 在这种情况下, 你可能只想积极的压缩最后 N 个分区, 同时等待足够的日志来积累较旧的分区; 最终结果是你已转换了大多数最近的数据, 这更有可能被查询优化的列格式  
这就是说, 出于明显的原型即压缩不阻塞摄取, 你可能想异步的运行它; 这个可通过在你的工作流调度或者 notebook 中单独调度一个分离的[压缩作业](https://github.com/apache/incubator-hudi/blob/master/hudi-utilities/src/main/java/org/apache/hudi/utilities/HoodieCompactor.java)来实现; 如果你使用 delta streamer, 你可以在 [持续模式](https://github.com/apache/incubator-hudi/blob/master/hudi-utilities/src/main/java/org/apache/hudi/utilities/deltastreamer/HoodieDeltaStreamer.java#L222) 下运行, 这样摄取和压缩都会被一个 spark 运行时间内同时进行

##### 对于 Hudi 写入我可以期盼怎样的执行/摄取延迟
写入 Hudi 的速度取决于写入操作以及你在进行文件大小调整时所要进行的一些权衡; 就像数据库在磁盘上的直接/原始文件 I/O 上产生开销一样, Hudi 操作也会产生开销在支持数据的特性上, 如读取/写入原始 DFS 文件; 就是说, Hudi 实现了数据库文献中的先进技术, 以使这些内容降至最低; 鼓励用户在尝试评估 Hudi 的性能时具有这种观点; 俗话说: 没有免费的午餐 (至少现在没有)

| 存储类型 | 负载类型 | 性能 | 技巧 |
| :--- | :--- | :--- | :--- |
| 写时复制 | bulk insert | - | - |
| 写时复制 | insert| - | - |
| 写时复制 | upsert/de-duplicate & insert | - | - |
| 读时合并 | bulk insert | - | - |
| 读时合并 | insert | - | - |
| 读时合并 | upsert/de-duplicate & insert | - | - |

与许多管理时间序列数据的典型系统一样, 如果你的键具有时间戳前缀或单调增加/减少, 则 Hudi 的性能会更好; 你几乎总是可以实现这一目标; 即使你具有 UUID 密钥, 也可以按照以下[技巧](https://www.percona.com/blog/2014/12/19/store-uuid-optimized-way/)来获得有序的密钥; 另请参阅 [调优指南](https://cwiki.apache.org/confluence/display/HUDI/Tuning+Guide) 获取有关JVM和其他配置的更多提示

##### 对于 Hudi 读取/查询我可以期盼怎样的性能
- 对于 ReadOprimized 视图: 你可以认为如同在 Hive/Spark/Presto 的 标准 parquet 表一样有相同的最好的列查询性能
- 对于 Incremental 视图: 你可以认为速度与给定窗口时间内有多少数据改变以及扫描条目使用的时间相关; 例如: 如果再上一个小时中一个有 1000 个文件的分区只有 100 个文件变更, 那么你可以认为使用 Hudi 的增量拉取相比于全量扫描分区来发现新数据快 10 倍
- 对于 RealTime 视图: 你可以认为与在 Hive/Spark/Presto 的基于 avro 的表有相同的性能

##### 我如何避免创建大量的小文件
Hudi 中的一项关键设计决策是避免创建小文件, 并且始终写入适当大小的文件, 在获取/写入上花费更多时间以保持查询始终高效; 写入非常小的文件然后将它们缝合在一起的常用方法只能解决由小文件引起的系统可伸缩性问题, 并且无论如何都会通过暴露小文件而使查询变慢  
执行上插/插入操作时, Hudi可以保持配置的目标文件大小; (注意: bulk_insert 操作不提供此功能, 而旨在代替普通的`spark.write.parquet`)  
对于写时复制, 这就像配置 [base/parquet 文件的最大大小](https://hudi.apache.org/configurations.html#limitFileSize)和[软限制](https://hudi.apache.org/configurations.html#compactionSmallFileSize)一样简单, 在此限制以下的文件将被视为小文件; Hudi 将在写入时尝试将足够的记录添加到一个小文件中, 以使其达到配置的最大限制; 例如, 对于 `compactionSmallFileSize = 100MB` 和 `limitFileSize = 120MB`, Hudi 会选择所有 < 100MB 的文件, 并尝试将其增加到120MB  
对于读时合并, 几乎没有其他设置可以设置; 具体来说, 你可以配置最大日志大小和一个因子, 该因子表示当数据从 avro 移至 parquet 文件时大小减小  
`[HUDI-26](https://issues.apache.org/jira/browse/HUDI-26) - Introduce a way to collapse filegroups into one and reindex #491 OPEN` 将使用此项进一步提升, 通过将较小的文件组合并成较大的文件

##### 我如何使用 DeltaStreamer 或者 Spark DataSource API 写入一个非分区 Hudi 数据集
Hudi 支持写入非分区数据集; 为了写入非分区的 Hudi 数据集并执行 hive 表同步, 你需要在传递的属性中设置以下配置
```
hoodie.datasource.write.keygenerator.class=org.apache.hudi.NonpartitionedKeyGenerator
hoodie.datasource.hive_sync.partition_extractor_class=org.apache.hudi.hive.NonPartitionedExtractor
```

##### 为什么我们必须设置两种不同的方式来配置 Spark 以与 Hudi 一起使用 ?
非 Hive 引擎倾向于自己制作 DFS 清单以查询数据集; 例如, Spark 开始直接从文件系统 (HDFS 或 S3) 读取路径; 从 Spark 的调用可能是以下其中一种
- org.apache.spark.rdd.NewHadoopRDD.getPartitions
- org.apache.parquet.hadoop.ParquetInputFormat.getSplits
- org.apache.hadoop.mapreduce.lib.input.FileInputFormat.getSplits

不了解 Hudi 文件的布局, 引擎只能直接读取所有的 parquet 文件然后展示这些文件的数据, 会导致大量的重复记录  
从高层次上讲, 有两种方法可以配置查询引擎以正确读取 Hudi 数据集
- 使它们调用在 `HoodieParquetInputFormat#getSplits` 和 `HoodieParquetInputFormat#getRecordReader` 中的方法
  - Hive 本地执行此操作, 因为 InputFormat 是 Hive 中用于插入新表格式的抽象; HoodieParquetInputFormat 扩展了 MapredParquetInputFormat, 它不过是 hive 的一种输入格式, 我们将 Hudi 表注册到这些输入格式支持的 Hive 元数据中
  - 当 Presto 看到一个U seFileSplitsFromInputFormat 注解时, 它也会回退到调用输入格式以获取拆分, 然后继续使用其自己的优化/矢量化 parquet 读取器来查询写时复制表
  - 可以使用 `--conf spark.sql.hive.convertMetastoreParquet = false` 将 Spark 强制退回到 HoodieParquetInputFormat 类
- 使引擎调用路径过滤器或其他方式来直接调用 Hudi 类以过滤 DFS 上的文件并挑选最新的文件切片
  - 即使我们可以强制 Spark 回退到使用 InputFormat 类, 但这样做可能会失去使用 Spark 的优化 parquet 读取路径的能力
  - 为了保持原生 parquet 读取性能的优势, 我们将 `HoodieROTablePathFilter` 设置为路径过滤器, 并在 Spark Hadoop 配置中明显示设置; 该文件中的逻辑: 确保 Hoodie 相关文件的文件夹 (路径) 或文件总是选择了最新的文件分片; 这将过滤出重复的条目并显示每个记录的最新条目

##### 有一个现有的数据集, 并希望使用该数据的一部分计算 Hudi 数据集
你可以批量导入部分数据到一个新的 hudi 表; 例如， 你想尝试一个月的数据
```Java
spark.read.parquet("your_data_set/path/to/month")
     .write.format("org.apache.hudi")
     .option("hoodie.datasource.write.operation", "bulk_insert")
     .option("hoodie.datasource.write.storage.type", "storage_type") // COPY_ON_WRITE or MERGE_ON_READ
     .option(RECORDKEY_FIELD_OPT_KEY, "<your key>").
     .option(PARTITIONPATH_FIELD_OPT_KEY, "<your_partition>")
     ...
     .mode(SaveMode.Append)
     .save(basePath);
```
拥有初始副本后, 你只需在每个回合中选择一些数据样本就可以对此进行 upsert 操作
```Java
spark.read.parquet("your_data_set/path/to/month").limit(n) // Limit n records
     .write.format("org.apache.hudi")
     .option("hoodie.datasource.write.operation", "upsert")
     .option(RECORDKEY_FIELD_OPT_KEY, "<your key>").
     .option(PARTITIONPATH_FIELD_OPT_KEY, "<your_partition>")
     ...
     .mode(SaveMode.Append)
     .save(basePath);
```
对于读时合并, 你可能还需要尝试安排和运行压缩作业; 你可以使用 `org.apache.hudi.utilities.HoodieCompactor` 提交 spark 作业直接运行压缩, 也可以使用 [HUDI CLI](https://hudi.incubator.apache.org/admin_guide.html#compactions) 运行压缩

>**参考:**
- [FAQ](https://cwiki.apache.org/confluence/display/HUDI/FAQ)
