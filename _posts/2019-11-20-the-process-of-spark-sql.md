---
layout: post
title: "Spark SQL 的处理过程"
date: "2019-11-20"
description: "Spark SQL 的处理过程"
tag: [spark]
---

### Spark SQL 的处理过程
- sql 经过 parser 解析成一个 UnresolvedLogicPlan (`ParSerInteface.parsePlan`)
- UnresolvedLogicPlan 经过 analyzer 的 rules 结合 catalog 实现生成 ResolvedLogicPlan (`QueryExecution.analyzed`)
  - UnresolvedLogicPlan 经过 cacheManager 使用已有缓存替换可能的分段 (`QueryExecution.withCacheData`)
- ResolvedLogicPlan 经过 optimizer 的 rules 对其进行优化生成 OptimizedLogicalPlan (`QueryExecution.opimiziedPlan`)
- OptimizedLogicalPlan 经过 planner 在应用一系列的 GenericStrategy 的可能的 PhysicalPlan 选出最优的 PhysicalSparkPlan (`QueryExecution.sparkPlan`)
- PhysicalSparkPlan 通过插入 shuffle 操作和所需的内部行格式转换准备一个 PlannedSparkPlan () (`QueryExecution.executedPlan`)
- 执行可执行的计划 (executedPlan) 返回 RDD

![spark-sql](https://ask.qcloudimg.com/http-save/yehe-2934635/970khf01lg.jpeg?imageView2/2/w/1620)

#### 解析器
TODO
#### 分析器
TODO
#### 优化器
TODO
#### 计划器
TODO

>**参考:**
- [Spark SQL与DataFrame](https://blog.csdn.net/lovehuangjiaju/article/details/48661847)
- [sparkSQL运行架构](https://blog.csdn.net/book_mmicky/article/details/39956809)
