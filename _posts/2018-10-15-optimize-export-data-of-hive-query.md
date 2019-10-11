---
layout: post
title: "优化导出 Hive 查询的数据"
date: "2018-10-15"
description: "优化导出 Hive 查询的数据"
tag: [hive]
---

### 问题
Hive-Jdbc 执行完成后调用 ResultSet.next() 方法导出数据, 在数据量较大时这样的单线程导出会很慢, 此外在调用  ResultSet.next() 有时还会触发 NPE 等异常

### 解决方案
如果能够找到执行 SQL 的结果集文件, 那么就可以直接读取结果集文件导出肯定会比调用 ResultSet.next() 方法有效率的多; Hive 在执行语句之前会先生成一个执行计划, 说明了要执行那些任务; 每个执行计划中都有一个 FetchTask, 这个 FetchTask 中包含了了执行结果集存放的 Hdfs 目录; 可以在 hive.exec.pre.hooks 中添加自定义 Hook, 将结果集地址写出到数据库, 导出时再根据 operation_id 在数据库中查到结果集所在位置, 然后将读取文件写出

#### Hook
并不是所有的 SQL 都会有执行结果, 而且没有走 MR 或者 TEZ 引擎执行的 FetchTask 作业也没有结果集路径, 而是从数据源文件中过滤结果, 这样的 SQL 仍然选择通过 ResultSet.next() 获取; 通过继承 Hive 的 `org.apache.hadoop.hive.ql.hooks.ExecuteWithHookContext`类, 在 run() 方法中处理获取地址逻辑
```
...
@Override
public void run(HookContext hookContext) throws Exception {
    if (hookContext.getQueryPlan() == null || hookContext.getQueryPlan().getFetchTask() == null) {
        return;
    }
    FetchTask fetchTask = hookContext.getQueryPlan().getFetchTask();
    // 获取 LogHelper
    Field field = fetchTask.getClass().getSuperclass().getDeclaredField("console");
    field.setAccessible(true);
    console = (SessionState.LogHelper) field.get(fetchTask);
    console.printInfo("Query result hook start ...");
    // 获取根任务
    List<Task<? extends Serializable>> rootTasks = hookContext.getQueryPlan().getRootTasks();
    // 获取 mr 和 tez 作业数 [hadoop-job <-> hive-task]
    int numOfMrJobs = Utilities.getMRTasks(rootTasks).size();
    int numOfTezJobs = Utilities.getTezTasks(rootTasks).size();
    // 有生成作业时, 最终文件路径在对应的 hive.exec.scratchdir 目录下; 不考虑 fetch 任务和 spark 任务
    if (numOfMrJobs + numOfTezJobs > 0) {
        try {
            saveQueryContext(hookContext);
        } catch (Exception e) {
            console.printError(e.getMessage());
        }
    } else {
        console.printInfo("There is only a fetch task, if you want to export data faster, please set hive.fetch.task.conversion = none");
    }
    console.printInfo("Query result hook end ...");
}
...
```

##### 获取 operation_id
-  Hook 中
```
/**
 * 获取解码后的 operationId, 见 SQLOperation.prepare() 方法中 Yarn ATS 说明
 * @param hookContext
 * @return
 */
private static String decodeOperationId(HookContext hookContext) {
    byte[] guid = Base64.decodeBase64(hookContext.getOperationId());
    ByteBuffer byteBuffer = ByteBuffer.wrap(guid);
    return new UUID(byteBuffer.getLong(), byteBuffer.getLong()).toString();
}
```
- 客户端
```
HiveStatement statement = (HiveStatement) resultSet.getStatement();
ByteBuffer buffer = ByteBuffer.wrap(statement.getStmtHandle().getOperationId().getGuid());
String operationId = new UUID(buffer.getLong(), buffer.getLong()).toString();
```
#### 只有 FetchTask 的 SQL
Hive 自身对一些语句做了优化, 例如当 `hive.fetch.task.conversion` 的值为 `more` 时, 对于 `select col_0, from db.tb where partition = ${partition} and col_0 = ${value}` 的语句优化为只有 FetchTask 的作业, 结果集是从数据文件中过滤获得的, 当数据量比较大时,  过滤时长会比较久; 当然这是 `more` 允许 Hive 做比较激进的优化; 那么当 `hive.fetch.task.conversion` 的值为 `none` 时, 对于所有的 SQL 查询都不做优化, 对于`select * from db.tb` 的语句起 MR 作业反而没有 FetchTask 来的快; Hive 中的关于 `hive.fetch.task.conversion` 的优化逻辑
见 [Hive-Fetch-Task](./2018-07-11.hive-fetch-task.md); 另外, 由于没有 MR 或 TEZ 作业, 不会产生结果集文件, 当查询出的数据量较大时导出依然很慢


##### 自定义 Hive-Fetch-Task 的 Limit 优化
经过以上分析, 导出数据的方案也逐渐明确, 大致有几点
- 查询的数据量较小, 可以通过 ResultSet 导出
- 查询的数据量较大, 必须经过读取结果文件集导出, 否则会很慢
- 要避免用户执行的 SQL 经过 Hive-Fetch-Task 优化查询大量的数据, 因为没有结果集文件导出慢


前两点基本可以确定, 当有写出结果集地址时就认为是大数据量, 通过读写文件获取; 当然也有写出结果集地址的 SQL, 但是结果集数量很少, 但是通过操作文件写出并不会比通过 ResultSet 操作慢多少; 关于第三点, 可以通过增加一些自定义的优化解决: 当用户的查询 SQL 带有 Limit 子句时认为查询的数据量较小, 可以考虑走 FetchTask 作业, 没有带 Limit 子句时一律认为 SQL 会产生较大的结果集, 强制走 MR 作业或 TEZ 作业; 这一点通过在 `SimpleFetchOptimizer.checkThreshold()` 方法里增加优化逻辑实现
```
public class SimpleFetchOptimizer extends Transform {
    ...
	private boolean checkThreshold(FetchData data, int limit, ParseContext pctx) throws Exception {
    // we don't want the query that without limit or limit great than a given number convert to fetch task
    boolean isConversion = HiveConf.getBoolVar(pctx.getConf(), HiveConf.ConfVars.HIVEFETCHTASKQUERYLIMITCONVERSION, false);
    if (isConversion) {
        long threshold = HiveConf.getLongVar(pctx.getConf(), HiveConf.ConfVars.HIVEFETCHTASKQUERYLIMITCONVERSIONCEILING, 100000);
        if (limit < 0 || limit > threshold) {
            return false;
        }
    }
	...
	}
...
}
#
public class HiveConf extends Configuration {
...
    HIVEFETCHTASKQUERYLIMITCONVERSION("hive.fetch.task.query.limit.conversion", false, "Determine query limit whether influence fetch task conversion."),

    HIVEFETCHTASKQUERYLIMITCONVERSIONCEILING("hive.fetch.task.query.limit.conversion.ceiling", 10000L, "If hive.fetch.task.query.limit.conversion is set to true,\n" +
        "It will't be converted fetch task when sql query limit number great than this value"),
...
}
```

#### 结果集文件解析
Hive 的查询结果文件格式是 SequenceFile, 也可以通过 `hive.query.result.fileformat` 配置为其他文件格式; 为了是数据可读, 在导出数据时需要解析文件; 由于 SequenceFile, RCfile, Llap 文件格式解析成本较高, 所以将结果集文件设置 TextFile 的格式, 然后解析 TextFile 写出; 解析代码细节如下
```
private static final byte HIVE_DEFAULT_ESCAPE = 92;
private static final byte HIVE_DEFAULT_DELIMITER = 1;

/**
 * 转换行数据 (替换分隔符, 转换数据), line 为结果集文件中的一行数据
 * @param line
 * @param columns
 * @param delimiter
 * @return
 */
private String convert(Text line, List<ExportColumn> columns, String delimiter) {
    StringBuilder sb = new StringBuilder("");

    Text field = new Text();
    int[] positions = getStartPositions(line, HIVE_DEFAULT_DELIMITER);
    for (int i = 0; i < columns.size(); i++) {
        LazyUtils.copyAndEscapeStringDataToText(line.getBytes(), positions[i], positions[i + 1] - positions[i] - 1, HIVE_DEFAULT_ESCAPE, field);
        String value = field.toString();
        String column = columns.get(i).getSensitive() ? transformerService.transformer(value, columns.get(i).getTransformer()) : value;
        if (i != columns.size() - 1) {
            sb.append(column).append(delimiter);
        } else {
            sb.append(column).append(System.lineSeparator());
        }
        // clear for reuse
        field.clear();
    }
    return sb.toString();
}

/**
 * 获取每个字段的起始位置
 * @param text
 * @param delimiter
 * @return
 */
private static int[] getStartPositions(Text text, byte delimiter) {
    List<Integer> delimiterPositions = getDelimiterPositions(text, delimiter);
    int[] startPositions = new int[delimiterPositions.size() + 2];
    // 起始位置默认为 0
    for (int i = 1; i <= delimiterPositions.size(); i++) {
        startPositions[i] = delimiterPositions.get(i - 1) + 1;
    }
    // 末尾位置默认为字节长度 + 一个 delimiter: 即认为 text.getBytes = n * [字节数 + delimiter]; Text.getBytes().length != Text.getLength()
    startPositions[startPositions.length - 1] = text.getLength() + 1;
    return startPositions;
}

/**
 * 获取分隔符位置列表
 * @param text
 * @param delimiter
 * @return
 */
private static List<Integer> getDelimiterPositions(Text text, byte delimiter) {
    List<Integer> positions = new ArrayList<>();
    for (int i = 0; i < text.getLength(); i++) {
        if (text.getBytes()[i] == delimiter) {
            positions.add(i);
        }
    }
    return positions;
}
```

#### 其他
- 结果集文件解析放在应用处理, 当数据量较多并且结果文件数较多时, 多线程的去解析会比较吃 CPU; 将处理过程写成 MR 任务放到 Yarn 上运行可以解决 CPU消耗的问题, 但 MR 有这起作业慢, 不好统计导出总行数等缺点
- 建议将结果集文件拷贝后处理, 否则当执行 SQL 的资源释放, Hive 会自动清理这些临时文件; 另外, 可能有结果集文件为空, 建议拷贝时过滤
