---
layout: post
title: "记录 SparkSQL 的使用明细"
date: "2019-12-05"
description: "记录 SparkSQL 的使用明细"
tag: [spark]
---
### 记录 SparkSQL 的使用明细
从 SparkShell 中运行 SQL 与从 SparkSQL 中运行 SQL 的执行方式不完全一致, SparkShell 中运行 SQL 当调用 ·`Dataset` 的 action 类方法时会触发 `SQLEexecution#withNewExecutionId` 方法, SparkSQL 中运行 SQL 则不会触发此方法; action 方法会额外触发 `QueryExecutionListener.onSuccess` 或 `QueryExecutionListener.onFailed` 的事件, 以及 `SparkListenerSQLExecutionStart` 和 `SparkListenerSQLExecutionEnd` 事件, 可以分别在 `QueryExecutionListener` 和 `DagUsageListener` 中得到回调, 在回调中可以解析 Spark 生成的执行计划, 在执行计划的 DAG 中找到 `HiveTableScanExec` 和 `FileSourceScanExec` 节点, 从而得到 SQL 中使用的 Hive 库表字段  
为了使 SparkSQL 中执行 SQL 也产生上述事件, 可以修改 `SparkSQLDriver#run` 方法如下
```Java
override def run(command: String): CommandProcessorResponse = {
    // TODO unify the error code
    try {
      context.sparkContext.setJobDescription(command)
      // Produce some events (reference Dataset#withAction and SQLExecution#withNewExecutionId)
      val start = System.nanoTime()
      // Add custom key
      val executionId = context.conf.getConfString("spark.custom.sql.execution.id", "-1").toLong + 1
      context.conf.setConfString("spark.custom.sql.execution.id", s"$executionId")
      val execution = context.sessionState.executePlan(context.sql(command).logicalPlan)
      try {
        val callSite = context.sparkContext.getCallSite()
        context.sparkContext.listenerBus.post(SparkListenerSQLExecutionStart(
          executionId, callSite.shortForm, callSite.longForm, execution.toString,
          SparkPlanInfo.fromSparkPlan(execution.executedPlan), System.currentTimeMillis()))
        try {
          hiveResponse = execution.hiveResultString()
          tableSchema = getResultSetSchema(execution)
        } finally {
          val now = System.currentTimeMillis()
          context.sparkContext.listenerBus.post(SparkListenerSQLExecutionEnd(executionId, now))
        }
        val end = System.nanoTime()
        context.sparkSession.listenerManager.onSuccess("run", execution, end - start)
        new CommandProcessorResponse(0)
      } catch {
        case e: Exception =>
          context.sparkSession.listenerManager.onFailure("run", execution, e)
          throw e
        case e: Throwable => throw e
      }
    } catch {
        case ae: AnalysisException =>
          logDebug(s"Failed in [$command]", ae)
          new CommandProcessorResponse(1, ExceptionUtils.getStackTrace(ae), null, ae)
        case cause: Throwable =>
          logError(s"Failed in [$command]", cause)
          new CommandProcessorResponse(1, ExceptionUtils.getStackTrace(cause), null, cause)
    }
  }
```
`SQLEexecution#withNewExecutionId` 方法中, Spark 为 `QueryExecution` 设置了一个 `spark.sql.execution.id`, 用于将一个查询执行产生的作业联系起来, 例如在计算 Metrics 时方便将所有作业 Metrics 累加 (`SQLListner#getExecutionMetrics`); SparkSQL 中的查询执行则无法通过此方法计算 Metrics, 可以将 `SparkListenerSQLExecutionStart` 和 `SparkListenerSQLExecutionEnd` 事件中间产生的 `SparkListenerTaskEnd` 事件视为一个查询执行产生的, 累加 Metrics 从而得到总的 Metrics; 这里也有例外情况, 如 IITS (insert into table select) 和 CTAS (crate table as select) 会产生多个  `SparkListenerSQLExecutionStart` 和 `SparkListenerSQLExecutionEnd`, 这时可以使用 `QueryExecutionListener.onSuccess` 或 `QueryExecutionListener.onFailed` 的事件来辅助确定哪些事件是同一个 SQL 产生的

>**参考:**
- [Spark Lisener 概要](https://www.jianshu.com/p/0ce5a0b6ca9d)
- [SQLListener Spark Listener](https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-SQLListener.html)
- [Spark SQL 处理流程分析](https://blog.csdn.net/lovehuangjiaju/article/details/50375431)
