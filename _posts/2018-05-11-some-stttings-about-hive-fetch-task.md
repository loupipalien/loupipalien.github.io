---
layout: post
title: "关于 Hive Fetch Task 的一些设置"
date: "2018-05-1"
description: "关于 Hive Fetch Task 的一些设置"
tag: [hive]
---

**环境:** CentOS-6.8-x86_64, Hadoop-2.7.3, Hive-2.1.1

### Hive Fetch Task 的设置项
- hive.fetch.task.conversion = more
一些查询可以被转换为单个 FETCH 任务以减少延迟; 当前查询必须是单个源的, 没有任何子查询, 任何聚合, 或 distincts (会触发 RS - ReduceSinkOperator, 此操作要求一个 MapReduce 任务), 横向视图和连接
- hive.fetch.task.aggr = false
没有 group-by 子句的聚合查询 (例如, select count(\*) from src) 的最终聚合在单个 reudce 任务中查询; 如果参数设置为 true, Hive将选举最终的聚合到一个 fetch task, 可能会减少查询时间
- hive.fetch.task.conversion.threshold = 1073741824
应用于 hive.fetch.task.conversion 的输入阈值 (字节), 如果目标表是原生的, 输入长度可以通过文件长度的合计算得来; 如果表不是原生的, 表的存储处理器可以选择性的实现 org.apache.hadoop.hive.ql.metadata.InputEstimator 接口; 一个负的阈值意味着 hive.fetch.task.conversion 应用时不依赖输入长度阈值

#### 关于 hive.fetch.task.conversion.threshold 源码
```
// SimpleFetchOptimizer 类
private boolean checkThreshold(FetchData data, int limit, ParseContext pctx) throws Exception {
  if (limit > 0) {
    if (data.hasOnlyPruningFilter()) {
      /* partitioned table + query has only pruning filters */
      return true;
    } else if (data.isPartitioned() == false && data.isFiltered() == false) {
      /* unpartitioned table + no filters */
      return true;
    }
    /* fall through */
  }
  long threshold = HiveConf.getLongVar(pctx.getConf(),
      HiveConf.ConfVars.HIVEFETCHTASKCONVERSIONTHRESHOLD);
  if (threshold < 0) {
    return true;
  }
  Operator child = data.scanOp.getChildOperators().get(0);
  if(child instanceof SelectOperator) {
    // select *, constant and casts can be allowed without a threshold check
    if (checkExpressions((SelectOperator)child)) {
      return true;
    }
  }
  long remaining = threshold;
  remaining -= data.getInputLength(pctx, remaining);
  if (remaining < 0) {
    LOG.info("Threshold " + remaining + " exceeded for pseudoMR mode");
    return false;
  }
  return true;
}
```
- 当 limit 大于 0, 且所查询表时分区表, 过滤字段只有分区字段时返回 true
- 当 limit 大于 0, 当所查询表不为分区表且没有过滤字段时返回 true
- 当 hive.fetch.task.conversion.threshold < 0 时返回 true
- 当 select star, constant, casts 时返回 true
- 当查询输入的数据小于 hive.fetch.task.conversion.threshold 时返回true, 否则为 false

>**参考:**
[Configuration Properties](https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties)  
[How to enable Fetch Task instead of MapReduce Job for simple query in Hive](http://www.openkb.info/2015/01/how-to-enable-fetch-task-instead-of.html)
[make hive query faster with fetch task](https://vcfvct.wordpress.com/2016/02/18/make-hive-query-faster-with-fetch-task/)  
