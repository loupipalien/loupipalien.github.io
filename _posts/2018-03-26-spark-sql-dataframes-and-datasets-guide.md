---
layout: post
title: "Spark SQL, DataFrames and Datasets 指南"
date: "2018-03-26"
description: "Spark SQL, DataFrames and Datasets 指南"
tag: [spark]
---

### 概述
Spark SQL 是 Spark 处理结构化数据的模块; 与基础的 Spark RDD API 不同, Spark SQL 提供的接口提供给 Spark 更多的关于数据和执行计算的结; 内在的, Spark SQL 使用这些额外的信息去执行额外的优化; 这里有几种包括 SQL 和 Datasets API 在内的与 Spark SQL 交互的方法; 当计算结果使用相同的执行引擎, 独立于你使用的表达计算的 API/语言; 这种统一意味着开发者可以依据哪种 APIs 对于给定的表达式提供了最自然的转换, 轻松地在切换不同的 APIs  
本文所有的示例使用的包含在 Sperk 的样例数据可以在 `spark-shell`, `pyspark` 或 `sparkR` 中运行

#### SQL
使用 Spark SQL 是为了执行 SQL 查询; Spark SQL 可以可以从已存在的 Hive 安装中读取数据; 如何配置这一特性详细请参见 [Hive Tables](https://spark.apache.org/docs/latest/sql-programming-guide.html#hive-tables) 章节; 当从另一种编程语言中运行 SQL 时结果将会以 [Dataset/DataFrame](https://spark.apache.org/docs/latest/sql-programming-guide.html#datasets-and-dataframes) 的形式返回; 你也可以使用 [command-line](https://spark.apache.org/docs/latest/sql-programming-guide.html#running-the-spark-sql-cli) 或者 [JDBC/ODBC](https://spark.apache.org/docs/latest/sql-programming-guide.html#running-the-thrift-jdbcodbc-server) 同 SQL 接口交互

#### Datasets 和 DataFrames
Dataset 是一个分布式数据集合; Datasets 是 Spark 1.6 新增加的接口, 在 Spark SQL 的优化执行引擎的收益下提供了 RDDs (强类型, 可以使用强大的匿名函数) 操作的收益; Datasets 可以由 JVM 的对象 [构建](https://spark.apache.org/docs/latest/sql-programming-guide.html#creating-datasets) 然后使用功能转换 (map, flatMap, filter, 等等); Dataset 的 API 在 [Scala](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset) 和 [Java](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/sql/Dataset.html) 中是可用的; Python 并不支持 Dataset API; 但是由于 Python 的动态特性, Dataset API 的许多收益是可用的 (例如: 你可以通过名称 row.columnName 自然的访问行的字段); 这一点和 R 是类似的  
DataFrame 是组织成命名的列的 Dataset; 在概念上等于关系型数据库中的一个表, 或者在 Pyhton/R 中的一个数据框架, 但是在 hood 下有着更丰富的优化; DataFrame 可以从很多 [源](https://spark.apache.org/docs/latest/sql-programming-guide.html#data-sources) 中构建, 例如: 结构化数据文件, Hive 表, 外部数据库, 已存在的 RDDs; DataFrame API 在 Scala, Java, [Python](https://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.DataFrame) 和 [R](https://spark.apache.org/docs/latest/api/R/index.html) 中都可用; 在 Scala 和 Java 中, DataFrame 被表示为一个行 Dataset; 在 [Scala API](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset) 中, DataFrame 是 Dataset[Row] 的一个简单类型别名; 但在 [Java API](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/sql/Dataset.html) 中, 用户需要使用 Dataset<Row> 去表示 DataFrame  
在本文中, 我们将会经常使用 Scala/Java 的行 Dataset 作为 DataFrame

### 快速开始

#### 起始点: SparkSession
在 Spark 中所有功能的主入点是 [SparkSession](https://spark.apache.org/docs/latest/api/java/index.html#org.apache.spark.sql.SparkSession) 类; 使用 `SparkSession.builder()` 创建基础的 SparkSession
- Scala
- Python
- R
- Java
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java 可以找到
import org.apache.spark.sql.SparkSession;

SparkSession spark = SparkSession
  .builder()
  .appName("Java Spark SQL basic example")
  .config("spark.some.config.option", "some-value")
  .getOrCreate();
```

Spark 2.0 版本的 `SparkSession` 提供了对 Hive 特性的內建支持, 包括使用 HiveQL 的查询, 访问 Hive 的 UDFs, 以及从 Hive 表中读取数据的能力; 为了使用这些特性, 你不需要有一个已经安装好的 Hive

#### 创建 DataFrames
使用 SparkSession, 应用可以从 [已存在的 RDD](https://spark.apache.org/docs/latest/sql-programming-guide.html#interoperating-with-rdds), Hive 表, 从 [Spark 数据源](https://spark.apache.org/docs/latest/sql-programming-guide.html#data-sources) 中创建 DataFrames; 以下示例基于 JSON 文件内容创建一个 DataFrame
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java 可以找到
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;

Dataset<Row> df = spark.read().json("examples/src/main/resources/people.json");

// Displays the content of the DataFrame to stdout
df.show();
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+
```

#### 泛型 Dataset 操作 (也叫做 DataFrame 操作)
在 Scala, Java, Python 和 R 中对于结构化数据操作, DataFrame 提供了一个领域指定语言  
正如以上所述, 在 Spark 2.0 中, 在 Scala 和 Java API 中 DataFrames 仅是行 Dataset; 这些操作也被称为 "泛型转换", 与 "类型转换" 相比, 具有强类型的Scala/Java数据集; 这里有一些使用 Datasets 处理结构化数据的一些基本示例
- Scala
- Python
- R
- Java
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java 可以找到
// col("...") is preferable to df.col("...")
import static org.apache.spark.sql.functions.col;

// Print the schema in a tree format
df.printSchema();
// root
// |-- age: long (nullable = true)
// |-- name: string (nullable = true)

// Select only the "name" column
df.select("name").show();
// +-------+
// |   name|
// +-------+
// |Michael|
// |   Andy|
// | Justin|
// +-------+

// Select everybody, but increment the age by 1
df.select(col("name"), col("age").plus(1)).show();
// +-------+---------+
// |   name|(age + 1)|
// +-------+---------+
// |Michael|     null|
// |   Andy|       31|
// | Justin|       20|
// +-------+---------+

// Select people older than 21
df.filter(col("age").gt(21)).show();
// +---+----+
// |age|name|
// +---+----+
// | 30|Andy|
// +---+----+

// Count people by age
df.groupBy("age").count().show();
// +----+-----+
// | age|count|
// +----+-----+
// |  19|    1|
// |null|    1|
// |  30|    1|
// +----+-----+
```

对于在 Dataset 上可执行的完整的类型操作列表参见 [API 文档](https://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/Dataset.html)  
另外, 对于简单的列引用和表达式, Datasets 也一个有丰富的函数库, 包括字符串处理, 日期算法, 通用的数学操作等; 可用函数的完整列表见 [DataFrame Function Reference](https://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/functions.html)

#### 运行 SQL 查询程序
`SparkSession` 的 `sql` 函数可以开启应用去运行 SQL 查询程序并返回一个 Dataset<Row> 形式的结果
- Scala
- Python
- R
- Java
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java 可以找到
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;

// Register the DataFrame as a SQL temporary view
df.createOrReplaceTempView("people");

Dataset<Row> sqlDF = spark.sql("SELECT * FROM people");
sqlDF.show();
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+
```

#### 全局临时视图
在 Spark SQL中的临时视图的会话范围的, 如果创建它的会话中断了就会消失; 如果你想有一个共享在所有会话中的临时视图, 且存活到 Saprk 应用中断, 你可以创建一个全局的临时视图; 全局临时视图捆绑与一个系统保留的数据库 `global_temp`, 并且我们必须使用限定名指向它, 例如 `SELECT * FROM global_temp.view1`
- Scala
- Python
- R
- Java
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java 可以找到
// Register the DataFrame as a global temporary view
df.createGlobalTempView("people");

// Global temporary view is tied to a system preserved database `global_temp`
spark.sql("SELECT * FROM global_temp.people").show();
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+

// Global temporary view is cross-session
spark.newSession().sql("SELECT * FROM global_temp.people").show();
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+
```

#### 创建 Datasets
Datasets 类似于 RDDs, 然而不再使用 Java 序列化器或 Kryo, 为了处理和在网络上传输使用一个指定的 [Encoder](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Encoder) 去序列化对象; 所有的编码器和标准序列化负责将对象转化为字节, 编码器是代码动态生成的, 并且使用一种允许 Spark 执行许多操作 (例如过滤, 排序, 哈希等) 不用将字节反序列化为对象的格式
- Scala
- Java
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java 可以找到
import java.util.Arrays;
import java.util.Collections;
import java.io.Serializable;

import org.apache.spark.api.java.function.MapFunction;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.Encoder;
import org.apache.spark.sql.Encoders;

public static class Person implements Serializable {
  private String name;
  private int age;

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }

  public int getAge() {
    return age;
  }

  public void setAge(int age) {
    this.age = age;
  }
}

// Create an instance of a Bean class
Person person = new Person();
person.setName("Andy");
person.setAge(32);

// Encoders are created for Java beans
Encoder<Person> personEncoder = Encoders.bean(Person.class);
Dataset<Person> javaBeanDS = spark.createDataset(
  Collections.singletonList(person),
  personEncoder
);
javaBeanDS.show();
// +---+----+
// |age|name|
// +---+----+
// | 32|Andy|
// +---+----+

// Encoders for most common types are provided in class Encoders
Encoder<Integer> integerEncoder = Encoders.INT();
Dataset<Integer> primitiveDS = spark.createDataset(Arrays.asList(1, 2, 3), integerEncoder);
Dataset<Integer> transformedDS = primitiveDS.map(
    (MapFunction<Integer, Integer>) value -> value + 1,
    integerEncoder);
transformedDS.collect(); // Returns [2, 3, 4]

// DataFrames can be converted to a Dataset by providing a class. Mapping based on name
String path = "examples/src/main/resources/people.json";
Dataset<Person> peopleDS = spark.read().json(path).as(personEncoder);
peopleDS.show();
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+
```

#### 与 RDDs 的交互操作
对于将已有 RDDs 转化到 Datasets 中 Spark SQL 支持两种不同的方式; 第一种方法使用反射推断包含指定类型对象的 RDDs 的模式; 当你已经知道要写入 Spark 应用的模式, 反射将使代码更简洁并且也能工作的很好; 第二种创建 Datasets 的方法时通过一个编程接口, 这个接口允许你构造一个模式然后将其应用到一个已存在的 RDD; 这种方法可以更细致, 它允许你在列和列的类型在运行时才知晓的情况下构建 Datasets

##### 使用反射推断模式
- Scala
- Pyhton
- Java
Spark SQL 支持自动的将 [JavaBeans](http://stackoverflow.com/questions/3295496/what-is-a-javabean-exactly) 的 RDD 转换为 DataFrame; 定义表模式的 BeanInfo 可使用反射获得; 当前, Spark SQL 不支持包含 Map 字段的 JavaBeans, 尽管已经支持内嵌的 JavaBeans 和 List 以及 Aarry; 你可以通过创建一个实现 Serializable 接口和有所有字段的 getter 和 setter 方法的类来创建 JavaBean
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java 可以找到
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.function.Function;
import org.apache.spark.api.java.function.MapFunction;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.Encoder;
import org.apache.spark.sql.Encoders;

// Create an RDD of Person objects from a text file
JavaRDD<Person> peopleRDD = spark.read()
  .textFile("examples/src/main/resources/people.txt")
  .javaRDD()
  .map(line -> {
    String[] parts = line.split(",");
    Person person = new Person();
    person.setName(parts[0]);
    person.setAge(Integer.parseInt(parts[1].trim()));
    return person;
  });

// Apply a schema to an RDD of JavaBeans to get a DataFrame
Dataset<Row> peopleDF = spark.createDataFrame(peopleRDD, Person.class);
// Register the DataFrame as a temporary view
peopleDF.createOrReplaceTempView("people");

// SQL statements can be run by using the sql methods provided by spark
Dataset<Row> teenagersDF = spark.sql("SELECT name FROM people WHERE age BETWEEN 13 AND 19");

// The columns of a row in the result can be accessed by field index
Encoder<String> stringEncoder = Encoders.STRING();
Dataset<String> teenagerNamesByIndexDF = teenagersDF.map(
    (MapFunction<Row, String>) row -> "Name: " + row.getString(0),
    stringEncoder);
teenagerNamesByIndexDF.show();
// +------------+
// |       value|
// +------------+
// |Name: Justin|
// +------------+

// or by field name
Dataset<String> teenagerNamesByFieldDF = teenagersDF.map(
    (MapFunction<Row, String>) row -> "Name: " + row.<String>getAs("name"),
    stringEncoder);
teenagerNamesByFieldDF.show();
// +------------+
// |       value|
// +------------+
// |Name: Justin|
// +------------+
```

##### 编程指定模式
- Scala
- Python
- Java
当 JavaBean 的类不能提前被定义 (例如: 记录的结构被编码成了字符串, 或者文本 dataset 将要被解析, 字段需要为不同的用户有不同的映射), 一个 Dataset<Row> 可以通过以下三步编程创建
  - 从原始的 RDD 创建一个行 RDD
  - 通过 StructType 创建模式视图匹配在第一步中创建的行 RDD 的结构
  - 通过 SparkSeesion 提供的 createDataFreme 方法将模式应用到行 RDD 上
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java 可以找到
import java.util.ArrayList;
import java.util.List;

import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.function.Function;

import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;

import org.apache.spark.sql.types.DataTypes;
import org.apache.spark.sql.types.StructField;
import org.apache.spark.sql.types.StructType;

// Create an RDD
JavaRDD<String> peopleRDD = spark.sparkContext()
  .textFile("examples/src/main/resources/people.txt", 1)
  .toJavaRDD();

// The schema is encoded in a string
String schemaString = "name age";

// Generate the schema based on the string of schema
List<StructField> fields = new ArrayList<>();
for (String fieldName : schemaString.split(" ")) {
  StructField field = DataTypes.createStructField(fieldName, DataTypes.StringType, true);
  fields.add(field);
}
StructType schema = DataTypes.createStructType(fields);

// Convert records of the RDD (people) to Rows
JavaRDD<Row> rowRDD = peopleRDD.map((Function<String, Row>) record -> {
  String[] attributes = record.split(",");
  return RowFactory.create(attributes[0], attributes[1].trim());
});

// Apply the schema to the RDD
Dataset<Row> peopleDataFrame = spark.createDataFrame(rowRDD, schema);

// Creates a temporary view using the DataFrame
peopleDataFrame.createOrReplaceTempView("people");

// SQL can be run over a temporary view created using DataFrames
Dataset<Row> results = spark.sql("SELECT name FROM people");

// The results of SQL queries are DataFrames and support all the normal RDD operations
// The columns of a row in the result can be accessed by field index or by field name
Dataset<String> namesDS = results.map(
    (MapFunction<Row, String>) row -> "Name: " + row.getString(0),
    Encoders.STRING());
namesDS.show();
// +-------------+
// |        value|
// +-------------+
// |Name: Michael|
// |   Name: Andy|
// | Name: Justin|
// +-------------+
```

#### 聚合
[內建的 DataFrames 函数](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.functions$) 提供通用的聚合函数, 例如 count(), countDistinct(), avg(), max(), min() 等等; 虽然这些函数是为 DataFrames 设计的, Spark SQL 在 [Scala](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.expressions.scalalang.typed$) 和 [Java](https://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/expressions/javalang/typed.html) 也有其中部分的类型安全版本可以在强类型的 Datasets 上使用; 此外, 用户并没有被限制预定义一些聚合函数, 完全可以创建自己的聚合函数

##### 泛型的用户自定义聚合函数
用户可以扩展 [UserDefinedAggregateFunction](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.expressions.UserDefinedAggregateFunction) 抽象类去实现一个自定义泛型聚合函数; 例如, 自定义的平均函数如下
- Scala
- Java
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaUserDefinedUntypedAggregation.java 可以找到
import java.util.ArrayList;
import java.util.List;

import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SparkSession;
import org.apache.spark.sql.expressions.MutableAggregationBuffer;
import org.apache.spark.sql.expressions.UserDefinedAggregateFunction;
import org.apache.spark.sql.types.DataType;
import org.apache.spark.sql.types.DataTypes;
import org.apache.spark.sql.types.StructField;
import org.apache.spark.sql.types.StructType;

public static class MyAverage extends UserDefinedAggregateFunction {

  private StructType inputSchema;
  private StructType bufferSchema;

  public MyAverage() {
    List<StructField> inputFields = new ArrayList<>();
    inputFields.add(DataTypes.createStructField("inputColumn", DataTypes.LongType, true));
    inputSchema = DataTypes.createStructType(inputFields);

    List<StructField> bufferFields = new ArrayList<>();
    bufferFields.add(DataTypes.createStructField("sum", DataTypes.LongType, true));
    bufferFields.add(DataTypes.createStructField("count", DataTypes.LongType, true));
    bufferSchema = DataTypes.createStructType(bufferFields);
  }
  // Data types of input arguments of this aggregate function
  public StructType inputSchema() {
    return inputSchema;
  }
  // Data types of values in the aggregation buffer
  public StructType bufferSchema() {
    return bufferSchema;
  }
  // The data type of the returned value
  public DataType dataType() {
    return DataTypes.DoubleType;
  }
  // Whether this function always returns the same output on the identical input
  public boolean deterministic() {
    return true;
  }
  // Initializes the given aggregation buffer. The buffer itself is a `Row` that in addition to
  // standard methods like retrieving a value at an index (e.g., get(), getBoolean()), provides
  // the opportunity to update its values. Note that arrays and maps inside the buffer are still
  // immutable.
  public void initialize(MutableAggregationBuffer buffer) {
    buffer.update(0, 0L);
    buffer.update(1, 0L);
  }
  // Updates the given aggregation buffer `buffer` with new input data from `input`
  public void update(MutableAggregationBuffer buffer, Row input) {
    if (!input.isNullAt(0)) {
      long updatedSum = buffer.getLong(0) + input.getLong(0);
      long updatedCount = buffer.getLong(1) + 1;
      buffer.update(0, updatedSum);
      buffer.update(1, updatedCount);
    }
  }
  // Merges two aggregation buffers and stores the updated buffer values back to `buffer1`
  public void merge(MutableAggregationBuffer buffer1, Row buffer2) {
    long mergedSum = buffer1.getLong(0) + buffer2.getLong(0);
    long mergedCount = buffer1.getLong(1) + buffer2.getLong(1);
    buffer1.update(0, mergedSum);
    buffer1.update(1, mergedCount);
  }
  // Calculates the final result
  public Double evaluate(Row buffer) {
    return ((double) buffer.getLong(0)) / buffer.getLong(1);
  }
}

// Register the function to access it
spark.udf().register("myAverage", new MyAverage());

Dataset<Row> df = spark.read().json("examples/src/main/resources/employees.json");
df.createOrReplaceTempView("employees");
df.show();
// +-------+------+
// |   name|salary|
// +-------+------+
// |Michael|  3000|
// |   Andy|  4500|
// | Justin|  3500|
// |  Berta|  4000|
// +-------+------+

Dataset<Row> result = spark.sql("SELECT myAverage(salary) as average_salary FROM employees");
result.show();
// +--------------+
// |average_salary|
// +--------------+
// |        3750.0|
// +--------------+
```

##### 类型安全的用户自定义聚合函数
对于强类型的 Datasets 用户自定义聚合围绕着 [Aggregator](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.expressions.Aggregator) 抽象类解决; 例如, 类型安全的自定义的平均函数如下
- Scala
- Java
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaUserDefinedTypedAggregation.java 可以找到
import java.io.Serializable;

import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Encoder;
import org.apache.spark.sql.Encoders;
import org.apache.spark.sql.SparkSession;
import org.apache.spark.sql.TypedColumn;
import org.apache.spark.sql.expressions.Aggregator;

public static class Employee implements Serializable {
  private String name;
  private long salary;

  // Constructors, getters, setters...

}

public static class Average implements Serializable  {
  private long sum;
  private long count;

  // Constructors, getters, setters...

}

public static class MyAverage extends Aggregator<Employee, Average, Double> {
  // A zero value for this aggregation. Should satisfy the property that any b + zero = b
  public Average zero() {
    return new Average(0L, 0L);
  }
  // Combine two values to produce a new value. For performance, the function may modify `buffer`
  // and return it instead of constructing a new object
  public Average reduce(Average buffer, Employee employee) {
    long newSum = buffer.getSum() + employee.getSalary();
    long newCount = buffer.getCount() + 1;
    buffer.setSum(newSum);
    buffer.setCount(newCount);
    return buffer;
  }
  // Merge two intermediate values
  public Average merge(Average b1, Average b2) {
    long mergedSum = b1.getSum() + b2.getSum();
    long mergedCount = b1.getCount() + b2.getCount();
    b1.setSum(mergedSum);
    b1.setCount(mergedCount);
    return b1;
  }
  // Transform the output of the reduction
  public Double finish(Average reduction) {
    return ((double) reduction.getSum()) / reduction.getCount();
  }
  // Specifies the Encoder for the intermediate value type
  public Encoder<Average> bufferEncoder() {
    return Encoders.bean(Average.class);
  }
  // Specifies the Encoder for the final output value type
  public Encoder<Double> outputEncoder() {
    return Encoders.DOUBLE();
  }
}

Encoder<Employee> employeeEncoder = Encoders.bean(Employee.class);
String path = "examples/src/main/resources/employees.json";
Dataset<Employee> ds = spark.read().json(path).as(employeeEncoder);
ds.show();
// +-------+------+
// |   name|salary|
// +-------+------+
// |Michael|  3000|
// |   Andy|  4500|
// | Justin|  3500|
// |  Berta|  4000|
// +-------+------+

MyAverage myAverage = new MyAverage();
// Convert the function to a `TypedColumn` and give it a name
TypedColumn<Employee, Double> averageSalary = myAverage.toColumn().name("average_salary");
Dataset<Double> result = ds.select(averageSalary);
result.show();
// +--------------+
// |average_salary|
// +--------------+
// |        3750.0|
// +--------------+
```

### 数据源
Spark SQL 支持通过 DataFrame 接口操作多种数据源; DataFrame 可以使用关系型转换操作并且也可以用于创建临时视图; 注册为一个临时视图的 DataFrame 允许你在其数据上运行 SQL 查询; 本章主要描述用于 Spark 数据源加载和保存数据的通用方法, 以及对于內建数据源可用的执行选项

#### 通用的加载/保存函数
最简单的形式, 默认的数据源 (默认是 parquet, 除非使用 spark.sql.sources.default 配置了其他值) 可以使用所有操作
- Scala
- Python
- R
- Java
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java 可以找到
Dataset<Row> usersDF = spark.read().load("examples/src/main/resources/users.parquet");
usersDF.select("name", "favorite_color").write().save("namesAndFavColors.parquet");
```

##### 手动指定选项
你可以手动指定数据源, 传递你想用于数据源的任何额外参数配合使用; 数据源可以使用它们的全限定名指定 (例如: org.apache.spark.sql.parquet), 对于內建数据源你可以使用它们的简单名称 (json, parquet, jdbc, orc, libsvm, csv, text); DataFrame 加载的任何数据源都可以使用以下语法转换为其他类型  
加载 JSON 文件
- Scala
- Python
- R
- Java
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java 可以找到
Dataset<Row> peopleDF =
  spark.read().format("json").load("examples/src/main/resources/people.json");
peopleDF.select("name", "age").write().format("parquet").save("namesAndAges.parquet");
```
加载 CSV 文件
- Scala
- Python
- R
- Java
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java 可以找到
Dataset<Row> peopleDFCsv = spark.read().format("csv")
  .option("sep", ";")
  .option("inferSchema", "true")
  .option("header", "true")
  .load("examples/src/main/resources/people.csv");
```

##### 在文件上直接运行 SQL
相比使用读取 API 将文件加载到 DataFrame 再查询它, 你可以直接在文件上使用 SQL 查询
- Scala
- Python
- R
- Java
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java 可以找到
Dataset<Row> sqlDF =
  spark.sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`");
```

##### 保存模式
保存操作能可选的使用一种保存模式, 当数据已存在是指明如何处理; 认识到保存模式没有使用任何锁和原子操作是重要的; 另外, 当执行重写时, 在写出新数据之前会删除老数据
|Scala/Java|任何语言|意义|
|-|-|-|
|SavaMode.ErrorIfExists (默认)|"error" 或 "errorifexists" (默认)|当保存 DataFrame 到数据源时, 如果数据已存在, 则期待着抛出一个异常|
|SaveMode.Append|"append"|当保存 DataFrame 到数据源时, 如果数据/表已存在, DataFrame 的内容将会追加到已存在的数据|
|SaveMode.Overwrite|"overwrite"|当保存 DataFrame 到数据源时, 如果数据/表已存在, DataFrame 的内容将会替代在已存在的数据|
|SaveMode.Ignore|"ignore"|当保存 DataFrame 到数据源时, 如果数据/表已存在, 保存操作将不会保存 DataFrame 的内容, 并不会改变已存在的数据; 类似于 SQL 中的 CREATE TABLE IF NOT EXISTS|

##### 保存到持久表
使用 `saveAsTable` 命令行可以将 DataFrame 作为持久表到 Hive 元数据; 一个已存在的 Hive 部署并需要这个特性; Spark 将创建一个默认的本地 Hive 元数据库 (使用 Derby) 给你; 不像 `createOrReplaceTempView` 命令, `saveAsTable` 会实例化 DataFrame 中的数据, 并会创建一个指向 Hive 元数据库中的数据的指针; 持久表在你重启 Spark 应用后仍存在, 只要你可以获取到同样元数库连; 对一个持久化表的 DataFrame 可以通过调用在 `SparkSession` 上的 `table` 方法带表名的参数创建  
对基于文件的数据源, 例如: text, parquet, json 等; 你可以通过 `path` 选项指定一个自定义的表路径, 例如: `df.write.option("path", "/some/path").saveAsTable("t")`; 当表被删除时, 自定义表路径不会被移除且表数据仍然存在; 如果没有指定自定义的表路径, Spark 将会把数据写入仓库路径下的默认路径; 当表别删除时, 默认表路径也会被移除  
从 Spark 2.1 开始, 持久数据源表有存储在 Hive 元数据库中的每个分区表信息, 这可以带来几个益处
- 因为元数据库仅会返回查询的必要分区, 找到查询表中的所有分区就不再需要
- Hive DDLs 例如 `ALTER TABLE PATITION ... SET LOCATION` 现在对于 Datasource API 创建的表是可用的

注意, 当创建外部数据源表 (有 `path` 选项) 时, 表分区信息默认没有被收集; 同步这些分区信息到元数据库中, 你可以调用 `MSCK REPAIR TABLE`

##### 分桶, 排序和分区
对于基于文件的数据源, 可能对输出进行分桶, 排序或者分区, 分桶和排序仅能应用于持久表
- Scala
- Python
- R
- Java
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java 可以找到
peopleDF.write().bucketBy(42, "name").sortBy("age").saveAsTable("people_bucketed");
```
当使用 Dataset API 时, 可以在 `save` 和 `saveAsTable` 进行分区
- Scala
- Python
- R
- Java
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java 可以找到
usersDF.write().partitionBy("favorite_color").format("parquet").save("namesPartByColor.parquet");
```
对单表同时进行分区和分桶时可行的
- Scala
- Python
- R
- Java
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java 可以找到
peopleDF.write().partitionBy("favorite_color").bucketBy(42, "name").saveAsTable("people_partitioned_bucketed");
```
`partitionBy` 会创建一个目录结构, 如在 [Partition Discovery](https://spark.apache.org/docs/latest/sql-programming-guide.html#partition-discovery) 部分中描述; 因此, 它对具有高基数的列的适用性很有限; 相反, `bucketBy` 在固定数量的桶中分布数据, 当一些独特的值不受限制时可以使用它

#### Parquet 文件
[Parquet](http://parquet.io/) 是一个被许多数据处理系统支持的列式存储格式; Spark SQL 支持对 Parquet 文件的读写, Parquet 文件会自动的将模式保存在元数据中; 当写 Parquet 文件时, 由于兼容性的原因所有的列会被自动的转换为 nullable

##### 编程加载数据
使用以上例子中的数据
- Scala
- Python
- R
- Sql
- Java
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java 可以找到
import org.apache.spark.api.java.function.MapFunction;
import org.apache.spark.sql.Encoders;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;

Dataset<Row> peopleDF = spark.read().json("examples/src/main/resources/people.json");

// DataFrames can be saved as Parquet files, maintaining the schema information
peopleDF.write().parquet("people.parquet");

// Read in the Parquet file created above.
// Parquet files are self-describing so the schema is preserved
// The result of loading a parquet file is also a DataFrame
Dataset<Row> parquetFileDF = spark.read().parquet("people.parquet");

// Parquet files can also be used to create a temporary view and then used in SQL statements
parquetFileDF.createOrReplaceTempView("parquetFile");
Dataset<Row> namesDF = spark.sql("SELECT name FROM parquetFile WHERE age BETWEEN 13 AND 19");
Dataset<String> namesDS = namesDF.map(
    (MapFunction<Row, String>) row -> "Name: " + row.getString(0),
    Encoders.STRING());
namesDS.show();
// +------------+
// |       value|
// +------------+
// |Name: Justin|
// +------------+
```

##### 分区发现
TODO...
##### 模式合并
TODO...
##### Hive 元数据 Parquet 表转换
TODO...
###### Hive/Parquet 表调制
TODO...
###### 元数据刷新
TODO...
##### 配置
TODO...

#### ORC 文件
TODO...
#### JSON Datasets
TODO...
#### Hive 表
Spark SQL 支持读写存储在 [Apache Hive](http://hive.apache.org/) 中的数据; 然而, 由于 Hive 有大量的依赖, 这些依赖并不包括在默认的 Spark 发布包中; 如果 Hive 的依赖在 classpath 中, Spark 将会自动的加载它们; 重要的是这些 Hive 依赖必须在所有的工作节点中, 因为为了能够访问存储在 Hive 中的数据必须要访问到序列化和反序列化的包  
Hive 的配置文件 hive-site.xml, core-site.xml (为了安全配置) 和 hdfs-site.xml (HDFS 的配置) 需要放置在 conf/ 目录下  
使用 Hive 时, 必须初始化为有 Hive 支持的 `SparkSession`, 包括连接到 Hive 元数据库, 支持 Hive serdes, 以及 Hive 的 UDF; 用户不需要有一个已经部署的 Hive 也能开启 Hive 支持; 当没有通过 `hive-site.xml` 来配置时, 上下文会在当前的文件夹中自动创建 `metastore_db` 并且创建一个由 `spark.sql.warehouse.dir` 配置的目录, 默认配置的 `spark-warehouse` 目录是在 Spark 应用启动的当前目录; 从 Spark 2.0.0 起, `hive-site.xml` 中的 `hive.metastore.warehouse.dir` 属性就以过时, 而用 `spark.sql.warehouse.dir` 中去指定在仓库中数据库的默认位置, 你需要为启动 Spark 应用的启动用户授予写权限
- Scala
- Python
- R
- Java
```
// 全部代码在 Spark 仓库中的 examples/src/main/java/org/apache/spark/examples/sql/hive/JavaSparkHiveExample.java 可以找到
import java.io.File;
import java.io.Serializable;
import java.util.ArrayList;
import java.util.List;

import org.apache.spark.api.java.function.MapFunction;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Encoders;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SparkSession;

public static class Record implements Serializable {
  private int key;
  private String value;

  public int getKey() {
    return key;
  }

  public void setKey(int key) {
    this.key = key;
  }

  public String getValue() {
    return value;
  }

  public void setValue(String value) {
    this.value = value;
  }
}

// warehouseLocation points to the default location for managed databases and tables
String warehouseLocation = new File("spark-warehouse").getAbsolutePath();
SparkSession spark = SparkSession
  .builder()
  .appName("Java Spark Hive Example")
  .config("spark.sql.warehouse.dir", warehouseLocation)
  .enableHiveSupport()
  .getOrCreate();

spark.sql("CREATE TABLE IF NOT EXISTS src (key INT, value STRING) USING hive");
spark.sql("LOAD DATA LOCAL INPATH 'examples/src/main/resources/kv1.txt' INTO TABLE src");

// Queries are expressed in HiveQL
spark.sql("SELECT * FROM src").show();
// +---+-------+
// |key|  value|
// +---+-------+
// |238|val_238|
// | 86| val_86|
// |311|val_311|
// ...

// Aggregation queries are also supported.
spark.sql("SELECT COUNT(*) FROM src").show();
// +--------+
// |count(1)|
// +--------+
// |    500 |
// +--------+

// The results of SQL queries are themselves DataFrames and support all normal functions.
Dataset<Row> sqlDF = spark.sql("SELECT key, value FROM src WHERE key < 10 ORDER BY key");

// The items in DataFrames are of type Row, which lets you to access each column by ordinal.
Dataset<String> stringsDS = sqlDF.map(
    (MapFunction<Row, String>) row -> "Key: " + row.get(0) + ", Value: " + row.get(1),
    Encoders.STRING());
stringsDS.show();
// +--------------------+
// |               value|
// +--------------------+
// |Key: 0, Value: val_0|
// |Key: 0, Value: val_0|
// |Key: 0, Value: val_0|
// ...

// You can also use DataFrames to create temporary views within a SparkSession.
List<Record> records = new ArrayList<>();
for (int key = 1; key < 100; key++) {
  Record record = new Record();
  record.setKey(key);
  record.setValue("val_" + key);
  records.add(record);
}
Dataset<Row> recordsDF = spark.createDataFrame(records, Record.class);
recordsDF.createOrReplaceTempView("records");

// Queries can then join DataFrames data with data stored in Hive.
spark.sql("SELECT * FROM records r JOIN src s ON r.key = s.key").show();
// +---+------+---+------+
// |key| value|key| value|
// +---+------+---+------+
// |  2| val_2|  2| val_2|
// |  2| val_2|  2| val_2|
// |  4| val_4|  4| val_4|
// ...
```

##### 为 Hive 表指定存储格式
当你创建一个 Hive 表时, 需要定义这个表如何从文件系统中读写数据, 即输入输出格式; 你也需要指定此表时如何反序列化数据到行, 或者序列化行到数据, 即 serde; 以下选项可以用于指定存储格式 ("serder, "input format", "output format"), 例如 `CREATE TABLE src(id int) USING hive OPTIONS(fileFormat 'parquet')`; 默认的, 我们可以将表文件作为文本读取, 但是创建 Hive 存储处理目前不允许存储为文本, 你可以使用 Hive 的存储处理器创建表, 然后使用 Spark SQL 去读取它
|属性名|意义|
|-|-|
|fileFormat|文件格式是存储格式规格的包, 包括 "serde", "input format", "output format"; 目前我们支持 6 种文件格: "sequencefile"， "rcfile", "orc", "parquet", "textfile", "avro"|
|inputFormat, outputFormat|这两个选项指定的是 "InputForamt" 和 "OutputFormat" 对应类的全限定名, 例如: "org.apache.hadoop.hive.ql.io.orc.OrcInputFormat"; 这两个选项必须成对出现, 如何你已经指定了 "fileFormat" 选项则不能再指定它们|
|serde|此选项指定 serde 的类名; 当 fileFormat 选项被指定时, 如果给定的 fileFormat 中包含了 serde 的信息则不需再指定; 当前 "sequencefile", "rcfile", "textFile" 不包括 serde 信息, 你可以在这三种文件格式中指定此选项|
|fieldDelim, escapeDelim, collectionDelim, mapkeyDelim, lineDelim|这些选项仅可以在 "textfile" 文件格式中使用, 这些定义了如何读取文件到行|
所有使用 `OPTIONS` 定义的其他属性定义都会被视为 Hive serde 的属性

##### 和不同版本的 Hive Metastore 交互
TODO...

#### JDBC 到其他数据库
TODO...

#### 排错
- JDBC 驱动器类对于所有执行节点的客户端的原生类加载器都必须是可见的; 这是因为 Java 的 DriverManager 类会做安全检查, 当在去打开一个连接时会导致它忽略所有不可见的驱动器; 一个简单的方式的修改所有工作节点上的 compute_classpath.sh 已包含你的驱动器 JARs
- 一些数据库, 例如 H2, 需要将所有命名转换为大写, 你需要使用大写去引用在 Spark SQL 中的命名

### 性能调优
通过缓存数据到内存中或者打开一些经验选项可能提升一些工作的执行性能

#### 缓存数据到内存中
TODO...

#### 其他配置选项
TODO...

#### 为 SQL 查询广播提示
TODO...

### 分布式 SQL 引擎
Spark SQL 可以作为使用 JDBC/ODBC 或 命令行接口时的分布式引擎; 在这个模式下, 终端用户和应用可以直接和 Spark SQL q去运行 SQL 查询, 而不需要去写任何代码

#### 运行 Thrift JDBC/ODBC 服务器
Thrift JDBC/ODBC 服务器实现对应的是 Hive 1.2.1 中的 [HiveServer2](https://cwiki.apache.org/confluence/display/Hive/Setting+Up+HiveServer2), 你可以使用 Spark 或 Hive 1.2.1 的 beeline 脚本测试 JDBC 服务器  
运行 Spark 目录中以下脚本启动 JDBC/ODBC 服务器
```
./sbin/start-thriftserver.sh
```
此脚本可以接受 `bin/spark-submit` 所有的命令行选项, 外加一个 `--hiveconf` 选项去指定 Hive 的属性; 你可以运行 `./sbin/start-thriftserver.sh --help` 查看完整的可用选项; 默认的, 服务器会监听 localhost:10000; 你可以通过环境变量修改, 例如
```
export HIVE_SERVER2_THRIFT_PORT=<listening-port>
export HIVE_SERVER2_THRIFT_BIND_HOST=<listening-host>
./sbin/start-thriftserver.sh \
  --master <master-uri> \
  ...
```
或者系统变量
```
./sbin/start-thriftserver.sh \
  --hiveconf hive.server2.thrift.port=<listening-port> \
  --hiveconf hive.server2.thrift.bind.host=<listening-host> \
  --master <master-uri>
  ...
```
然后可以使用 beeline 测试 Thrift JDBC/ODBC 服务器
```
./bin/beeline
```
使用 beeline 连接到 JDBC/ODBC 服务器
```
beeline> !connect jdbc:hive2://localhost:10000
```
Beeline 会向你询问用户名和密码; 在非安全模式下, 简单的输入机器上的用户名和空白密码; 在安全模式下, 需遵循 [beeline 文档](https://cwiki.apache.org/confluence/display/Hive/HiveServer2+Clients) 中的规定  
Hive 的配置文件 hive-site.xml, core-site.xml (为了安全配置) 和 hdfs-site.xml (HDFS 的配置) 需要放置在 conf/ 目录下  
你也可以使用 Hive 中带的 beeline 脚本  
Thrift JDBC  服务器也支持通过 HTPP 传输发送 thrift RPC 消息; 使用以下属性设置开启 HTTP 模式, 作为系统变量或者在 conf/ 目录下的 hive-site.xml
```
hive.server2.transport.mode - Set this to value: http
hive.server2.thrift.http.port - HTTP port number to listen on; default is 10001
hive.server2.http.endpoint - HTTP endpoint; default is cliservice
```
使用 beeline 连接 http 模式的 JDBC/ODBC 服务器
```
beeline> !connect jdbc:hive2://<host>:<port>/<database>?hive.server2.transport.mode=http;hive.server2.thrift.http.path=<http_endpoint>
```

#### 运行 Spark SQL 命令行
TODO...

>**参考:**
[Spark SQL, DataFrames and Datasets Guide](https://spark.apache.org/docs/latest/sql-programming-guide.html)
