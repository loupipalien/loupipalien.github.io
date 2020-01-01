---
layout: post
title: "数据仓库的演进"
date: "2019-12-16"
description: "数据仓库的演进"
tag: [BigData]
---

### 数据仓库
比尔恩门((Bill Inmon) 在 1991 年出版的 <<Building the Data Warehouse>> 中提出 --- 数据仓库是一个面向主题的(Subject Oriented), 集成的(Integrate), 相对稳定的(Non-Volatile), 反应历史变化(Time Varint) 的数据集合, 用于至此管理决策;

#### 数据仓库的趋势
数据仓库是伴随着企业信息化发展起来的, 在企业信息化的过程中随着信息化的升级, 数据量变的越来越大, 决策要求越来越苛刻, 数据仓库技术也在不停的发展来解决这些需求, 总的来说趋势如下
- 数据越来越实时化
- 决策越来越智能化
- 数据类型越来越多样化

#### 数据流管道架构
Inmon 在 1990 年代提出数仓仓库的概念并给出了完整的建设方法, 随着与日俱增的数据量的需求, 在此架构中使用大数据工具来替代经典数仓中的传统工具; 可以把此架构叫做**离线大数据架构**, 时至今日此架构仍然适用

##### Lambda 架构
后续随着业务实时性要求的提高, 开始在原有离线大数据架构基础上增加了一个加速层, 使用流处理来直接完成一些实时性较高的指标计算, 这就是 **Lambda 架构**  
![Lambda 架构](http://1fykyq3mdn5r21tpna3wkdyi-wpengine.netdna-ssl.com/wp-content/uploads/2017/03/Figure-1-Hoodie.png)  
在 Lambda 架构中有流处理层和批处理层的双重计算; 每隔几个小时, 就会启动一个批处理过程来计算准确的业务状态, 并将批量更新加载到服务层; 同时, 流处理层计算并提供相同的状态, 以避免上述的多小时延迟; 但是在被更精确的批处理计算状态覆盖之前, 此状态只是一个近似状态; 因为流处理和批处理的状态略有不同, 所以需要批处理和流处理提供不同的服务层, 并在这个上面再做合并抽象

##### Kappa 架构
在 Lambda 架构中对同一份数据进行了流处理和批处理, 虽然提供了数据的实时性但也消耗了更多的资源; 另外, 从广义上将所有的数据计算都可以描述为生产者生产数据流, 消费者不对逐条迭代消费这个流中的数据, 基于这个特定是的可以在流处理层通过增加并行性和资源来对有状态的业务数据重新处理, 依靠有效非检查点 (checkpoint) 的大量的状态管理来让流处理的结果不再是一个近似值, 于是 Kappa 架构应运而出  
![Kappa 架构](https://s.iteblog.com/pic/hudi/kappa_architecture_hudi-iteblog.png)
但流处理系统往往需要在内存中存储大量的数据和状态, 由于内存往往是有限的, 所以对于数据驻留的能力也是有效的, 对于历史数据的分析又会被重新定向到对时延要求不高的 HDFS 上

##### Uber 架构 (杜撰的)
为了解决时延和资源之间的冲突, Uber 开发了 Hudi 来解决这一点; Hudi 是一个增量数据处理框架, 它通过实现增量更新 HDFS 上的数据集, 可以用较少的资源在短时间内将数据变更应用于 HDFS 上的数据集； Hudi 数据集通过自定义 InputFormat 与当前 Hadoop 生态系统 (包括 Hive, Presto, Spark 以及 Parquet) 集成, 使得该框架对最终用户来说是无缝的  
![Uber 架构](https://s.iteblog.com/pic/hudi/hudi_simplified_architecture-iteblog.png)  

>**参考:**
- [如果你也想做实时数仓](https://zhuanlan.zhihu.com/p/82864697)
- [OPPO 实数据仓库](https://www.infoq.cn/article/FJxTIbYCUPlB*Gcokey5)
- [Hudi: Uber Engineering’s Incremental Processing Framework on Apache Hadoop](https://eng.uber.com/hoodie/)
- [大数据架构如何做到流批一体](https://www.infoq.cn/article/Uo4pFswlMzBVhq*Y2tB9)
