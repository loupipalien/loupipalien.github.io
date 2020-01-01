---
layout: post
title: "MapReduce 中的 Counter"
date: "2019-12-05"
description: "MapReduce 中的 Counter"
tag: [mapreduce]
---

### MapReduce 中的 Counter
Counter 是 MapReduce 中提供的一个计数器, 可以在分布式的 MapReduce 作业中用作统计数据, 在 Yarn 上运行的 Mapreduce 应用, 提供了内置 Counter 的明细页, 其中每个 Counter 的含义可见 [MapReduce计数器](https://www.cnblogs.com/codeOfLife/p/5521356.html) 的解释,

#### 采集 Hive 提交作业的 I/O 信息
这里采集每条 SQL 语句产生作业读取的字节数和记录数, 写出的字节数和记录数, 对应的 Counter 如下
- FileSystemCounters#HDFS_BYTES_READ
- HIVE#`RECORDS_IN(_)*(\\S+)*`
- FileSystemCounters#HDFS_BYTES_WRITTEN
- HIVE#`RECORDS_OUT_(\\d+)(_)*(\\S+)*`
```Java
public class HiveHook implements ExecuteWithHookContext {

    private SessionState.LogHelper console = SessionState.getConsole();

    @Override
    public void run(HookContext context) throws Exception {
        Map<String, MapRedStats> stats = SessionState.get().getMapRedStats();
        // counters
        String hiveCounters = conf.get(HiveConf.ConfVars.HIVECOUNTERGROUP.varname);
        long inBytes = getCounterValue(stats, "FileSystemCounters", "HDFS_BYTES_READ");
        long inRecords = getCounterValue(stats, hiveCounters, getCounterName(stats, hiveCounters, RECORDS_IN_PATTERN));
        long outBytes = getCounterValue(stats, "FileSystemCounters", "HDFS_BYTES_WRITTEN");
        long outRecords = getCounterValue(stats, hiveCounters, getCounterName(stats, hiveCounters, RECORDS_OUT_PATTERN));
    }
}

/**
 * @see org.apache.hadoop.hive.ql.history.HiveHistoryImpl#getRowCountTableName
 */
private static final Pattern RECORDS_IN_PATTERN = Pattern.compile("RECORDS_IN(_)*(\\S+)*");
private static final Pattern RECORDS_OUT_PATTERN = Pattern.compile("RECORDS_OUT_(\\d+)(_)*(\\S+)*");

/**
 * HIVE 的 Counter 组中 RECORDS_IN 和 RECORDS_OUT 的 name 可能被追加后缀
 * @see org.apache.hadoop.hive.ql.exec.AbstractMapOperator#initializeMapOperator
 * @see org.apache.hadoop.hive.ql.exec.tez.TezJobMonitor#printDagSummary
 * @see org.apache.hadoop.hive.ql.exec.FileSinkOperator#initializeOp
 * @param stats
 * @param group
 * @param pattern
 * @return
 */
private static String getCounterName(Map<String, MapRedStats> stats, String group, Pattern pattern) {
    for (MapRedStats mapRedStats : stats.values()) {
        Counters.Group counters = mapRedStats.getCounters().getGroup(group);
        if (counters != null) {
            Iterator<Counters.Counter> iterator = counters.iterator();
            while (iterator.hasNext()) {
                Counters.Counter counter = iterator.next();
                if (pattern.matcher(counter.getName()).find()) {
                    return counter.getName();
                }
            }
        }
    }
    return pattern.toString();
}

private static long getCounterValue(Map<String, MapRedStats> stats, String group, String name) {
    long value = 0;
    for (MapRedStats mapRedStats : stats.values()) {
        Counter counter = mapRedStats.getCounters().findCounter(group, name);
        if (counter != null && counter.getValue() >= 0) {
            value += counter.getValue();
        }
    }
    return value;
}
```

>**参考:**
- [MapReduce Application Master REST API’s](https://hadoop.apache.org/docs/r2.7.3/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapredAppMasterRest.html)
- [MapReduce计数器](https://www.cnblogs.com/codeOfLife/p/5521356.html)
- [MapReduce 计数器简介](https://cloud.tencent.com/developer/article/1043827)
