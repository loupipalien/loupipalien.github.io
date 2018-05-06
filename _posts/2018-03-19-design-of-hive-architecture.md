---
layout: post
title: "Hive 的架构设计"
date: "2018-03-19"
description: "Hive 的架构设计"
tag: [hive]
---

### 设计
本文包含了 Hive 设计和架构的细节, 可在  [hive.pdf](https://cwiki.apache.org/confluence/display/Hive/Design#) 获取关于 Hive 简要的技术报告

![Image](/images/posts/2018-03-19-design-of-hive-architecture/1.png)

#### Hive 架构
上图展示了 Hive 的主要组件以及和 Hadoop 的交互; 如图所示, Hive 的主要组件如下:
- UI: 用户提交查询或其他操作到系统的用户接口; 到 2011 年, 系统有一个命令行接口和一个基于 GUI 的 web 正在开发
-  Driver: 接受查询的组件; 组件实现了会话处理器的概念, 并且模仿 JDBC / ODBC 接口提供了执行和获取的 APIs
- Compiler: 解析查询的组件, 处理不同查询语句块和查询表达式的语义分析, 并且在从 metastore 中查询来的表和分区元数据的帮助下最终生成一个执行计划
-  Metastore: 此组件存储在数仓中各种表和分区的所有结构化信息, 包括字段和字段类型信息, 必要的读写数据的序列化器和反序列化器, 以及数据存储的对应的 HDFS 地址;
- Execution Engine: 此组件执行由编译器创建的执行计划; 计划是一个关于阶段的有向无环图; 执行引擎管理着这些计划中不同阶段的依赖, 并且在正确的系统组件上执行这些阶段

上图也显示一个典型的提交到系统的查询流程:
- UI 调用执行接口到 Driver (第 1 步)
- Driver 为查询创建一个会话句柄并发送查询到 Compiler 生成一个执行计划 (第 2 步)
- Compiler 从 Metastore 获取必要的元数据 (第 3, 4 步), 元数据用于对查询语法树中的表达式做类型检查, 同时也基于查询谓词删减分区
- Compiler 生成一个关于阶段的有向无环图, 每一个阶段都对应着一个 map/reduce 作业, 一个元数据操作或者 HDFS 上的操作 (第 5 步); 对于 map/reduce 阶段, 执行计划包含 map 操作符树 (操作符树在 mappers 上执行) 和 reduce 操作符树 (用于需要 reducers 的操作)
- 执行引擎提交这些阶段到合适的组件中 (第 6, 6.1, 6.2, 6.3 步); 在每个任务中 (mapper/reducer), 将反序列化器关联到表或者从 HDFS 文件中按行将中间结果读出, 并且这些将传递到关联的操作符树; 一旦输出生成, 它会通过序列化器写成一个临时的 HDFS 文件 (在操作不需要 reduce 的情况下这个将在 mapper 时发生); 临时文件用于为计划后续阶段的 map/reduce 提供数据; 对于 DML 的操作最终的临时文件会被移动到表地址下; 这个空间用于确保不会读到脏数据 (文件重命名会被 HDFS 自动操作)
- 对于查询, 临时文件的内容会直接被执行引擎从 HDFS 中读取, 作为从 Driver 的获取调用的一部分 (第 7, 8, 9 步)

#### Hive 数据模型
Hive 中的数据被组织成以下形式:
- Tables: 类似于关系型数据库中的表; 表可以被过滤, 投影, 联合以及合并; 另外, 表的所有数据都被存储在 HDFS 的目录中; Hive也支持外部表的概念, 即通过提供正确的地址到表创建的 DDL 语句中, 一个表可以创建在 HDFS 上预先存在文件或目录上; 类似关系型数据库, 在表中的行会被组织进类型字段
- Partiitons: 每个表可以有一个或多个分区键决定数据怎么存储, 例如有一个日期分区字段 ds 的样例表 T, 每个分区的文件数据存储在 HDFS 的 <table location>/ds=<date> 目录中; 分区允许系统基于查询谓词检查去删减数据, 例如对于对 T 表中符合谓词 T.ds = '2008-09-01' 行有兴趣的样例查询, 将只会查看 HDFS 的 <table location>/ds=<date> 目录中的文件
-  Buckets: 每个分区中的数据可会依次基于表中字段的哈希值划分到桶中; 分区目录中每个桶会被存储为一个文件; 分桶允许系统以来样例数据有效的预估查询 (这些查询使用表中的样例子句)


除了原始字段类型 (整型, 浮点型, 字符串, 日期以及布尔型), Hive 也支持数组和映射类型; 用户可以用原始类型, 集合类型或者其他用户自定义类型组合他们自己的编程类型; 类型系统和 SerDe (序列化/反序列化) 以及对象检查器接口紧密相关; 用户可以通过实现他们自己的对象检查器来创建他们的数据类型, 使用这些对象检查器他们可以创建他们自己的 SerDes 去序列化和反序列化数据到 HDFS 文件中; 这里提供了两个接口提供了必要的钩子去拓展 Hive 的能力, 当 Hive 需要理解其他数据格式和更丰富的类型; 內建对象检查器, 例如 ListObjectInspector, StructObjectInspector 以及 MapObjectInspector 提供必要的原始类型使用拓展的方式去组合更丰富的类型; 对于映射 (关联数组) 和数组, 一些有用 的內建函数例如 size 和 index 等操作符都是有提供的; 点符号用于指向內建类型, 例如 a.b.c = 1 是查看 a 的 b 字段的 c 字段和 1 比较

#### Metastore

##### 动机
Metastore 提供了两个重要的但是经常被忽略的数据仓库的特性: 数据抽象和数据发现; 没有 Hive 提供的数据抽象, 用户在查询时不得不提供关于数据格式, 抽取器以及加载器的信息; 在 Hive 中, 在表创建期间和每次表被引用的重用是都提供了响应信息; 这和传统数仓系统是十分接近的; 第二个功能, 数据发现, 能够使用户发现和探索数仓中指定数据和相关数据; 其他工具可以使用元数据构建去揭示或可能提高数据的信息和可用性; Hive 通过元数据仓库实现了这些特性, 元数据仓库和 Hive 查询处理系统整合在一起的, 所以数据和元数据是同步的

##### Metastore 对象
- Database: 一个表空间, 在未来将会被视为管理的单元; default 数据库用于那些用户没有指定数据库名的表
- Table: 数据表的元数据包括字段, 属主, 存储和 SerDe 信息; 它也可以存储用户提供的任何键值对信息; 存储信息包括底层数据的位置信息, 文件的输入输出格式和桶信息; SerDe 元数据包括序列化和反序列化的实现类以及任支持的实现所需信息; 以上所有的信息可以在建表期间提供
- Partition: 每个分区可以有它自己的子弹和 SerDe 以及存储信息; 这有利于空间修改而不影响到老的分区

##### Metastore 架构
Metastore 是数据库或文件支持存储的对象存储; 数据库支持存储实现使用一个名为 [DataNucleus](http://www.datanucleus.org/) object-relational-mapping (ORM) 解决方案; 存储于关系型数据库的主要动机是元数据的可查询能力; 将数据和元数据使用分离储存而不是使用 HDFS 的弊处在于同步性和可伸缩性问题; 另外, 因为要随机更新文件, 没有明确的方式可以将对象存储实现与 HDFS 之上; 再加上可查询和关系型存储的优点, 使我们的方法更合理些  
metastore 可以使用以下几种方式配置: 远程和内嵌; 在远程模式中, metastore 是一个 [Thrift](https://thrift.apache.org/) 服务; 对于非 Java 客户端这种模式是有用的; 在内嵌模式中, Hive 客户端使用 JDBC 直连底层 metastore; 这种模式是有用的, 因为它避免了需要去维护和监控其他系统; 两种模式可以共存 (更新: 本地 metastore 是第三种可能, 详细见 [Hive Metastore Administration](https://cwiki.apache.org/confluence/display/Hive/AdminManual+MetastoreAdmin))

##### Metastore Interface
Metastore 提供了一个 [Thrift 接口](https://thrift.apache.org/docs/idl) 去操纵和查询 Hive 元数据, Thrift 提供了与许多主流语言的绑定; 第三方工具可以使用这些接口去整合 Hive 元数据到其他业务元数据仓库中

#### Hive 查询语言
TODO...

#### 编译器
TODO...
- 解析器: 将查询语句转换为解析语法树表示
- 语法分析器: 转换解析语法树为一个内部的查询表示, 它仍是基于块的而不是操作树;
- 逻辑计划生成器
- 查询计划生成器

#### 优化器
TODO...

#### Hive APIs
Hive [APIs](https://cwiki.apache.org/confluence/display/Hive/Hive+APIs+Overview) 概览描述了 Hive 提供的大量公共 APIs

>**参考:**
[Hive Design](https://cwiki.apache.org/confluence/display/Hive/Design)  
