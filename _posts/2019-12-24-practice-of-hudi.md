---
layout: post
title: "Hudi 实践"
date: "2019-12-24"
description: "Hudi 实践"
tag: [Hudi]
---

### Hudi 实践
通过 Debezium, Hudi 等组件构建一个近实时管道的 Demo

#### 环境搭建
使用 4C8G 以上配置的 CentOS 安装 Docker 以及 Docker Compose, 使用 [此 YAML 文件](https://gist.github.com/loupipalien/22f2e6fbe656b4cf62f24b1e770c9f35) 搭建实践环境, 此文件是依据 Hudi 的 [Docker Demo](https://hudi.incubator.apache.org/docker_demo.html) 和 Debezium 的 [Tutorial](https://debezium.io/documentation/reference/0.10/tutorial.html) 和 [Avro Serialization](https://debezium.io/documentation/reference/0.10/configuration/avro.html) 编写的 (建议先阅读这三篇文档并实践后再继续看本文)
- 启动脚本 (start.sh)
```Shell
#!/bin/bash

# constant
export HUDI_WS=/app/hudi
export DEBEZIUM_VERSION=0.10

# script dir
path=`dirname $0`
# yaml file
file="docker-compose-near-real-time-pipeline.yml"

# restart
docker-compose -f $path/$file down
docker-compose -f $path/$file pull
sleep 5
docker-compose -f $path/$file up -d
sleep 60

docker exec -it adhoc-1 /bin/bash /var/hoodie/ws/docker/demo/setup_demo_container.sh
docker exec -it adhoc-2 /bin/bash /var/hoodie/ws/docker/demo/setup_demo_container.sh

# registry monitor
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostnam
e": "mysql", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "184054", "database.server.name": "dbserver1", "database.whitelist": "inventory", "database.history.kafka.bootstrap.servers": "kafkabroker:
9092", "database.history.kafka.topic": "dbhistory.inventory" } }'
```
- 停止脚本 (stop.sh)
```Shell
#!/bin/bash

# constant
export HUDI_WS=/app/hudi
export DEBEZIUM_VERSION=0.10

# script dir
path=`dirname $0`
# yaml file
file=docker-compose-near-real-time-pipeline.yml

# stop
docker-compose -f $path/$file down

# remove mount directory
rm -rf /tmp/hadoop_data
rm -rf /tmp/hadoop_name
```

#### 捕获数据变化
Debezuim 是一个变化数据捕获器 (Change Data Cpature， CDC), 已经搭建好的环境中已经配置好了一个监控 MySQL 实例的 Monitor, 可将 MySQL 的 binlog 数据 (Avro 格式) 发送到 Kafka; 可使用以下语句访问 MySQL 实例  
```
docker run -it --rm --name mysqlterm --network docker_default --link mysql mysql:5.7 sh -c 'exec mysql -hmysql -P3306 -uroot -pdebezium'
```
再启动一个 Kafka 消费者来获取新产生的 binlog 数据
```
docker run -it --rm --name avro-consumer --network docker_default -e KAFKA_LOG4J_OPTS="-Dlog4j.configuration=file:/kafka/config.orig/tools-log4j.properties" --link zookeeper:zookeeper --link kafka:kafka --link mysql:mysql --link schema-registry:schema-registry debezium/connect:0.10 /kafka/bin/kafka-console-consumer.sh --bootstrap-server kafkabroker:9092 --property print.key=true --formatter io.confluent.kafka.formatter.AvroMessageFormatter --property schema.registry.url=http://schema-registry:8081 --topic dbserver1.inventory.customers
```
改动 `inventory.customers` 表的数据, 可实时看到 MySQL 产生的 binlog 数据; 以下是增改删一条记录产生的 binlog 数据
```
{"id":1005}     {"before":null,"after":{"dbserver1.inventory.customers.Value":{"id":1005,"first_name":"Kenneth","last_name":"Anderson","email":"kander@acme.com"}},"source":{"version":"0.10.0.Final","connector":"mysql","name":"dbserver1","ts_ms":1577430917000,"snapshot":{"string":"false"},"db":"inventory","table":{"string":"customers"},"server_id":223344,"gtid":null,"file":"mysql-bin.000003","pos":2245,"row":0,"thread":{"long":7},"query":null},"op":"c","ts_ms":{"long":1577430918380}}
{"id":1005}     {"before":{"dbserver1.inventory.customers.Value":{"id":1005,"first_name":"Kenneth","last_name":"Anderson","email":"kander@acme.com"}},"after":{"dbserver1.inventory.customers.Value":{"id":1005,"first_name":"Kander","last_name":"Anderson","email":"kander@acme.com"}},"source":{"version":"0.10.0.Final","connector":"mysql","name":"dbserver1","ts_ms":1577431358000,"snapshot":{"string":"false"},"db":"inventory","table":{"string":"customers"},"server_id":223344,"gtid":null,"file":"mysql-bin.000003","pos":2559,"row":0,"thread":{"long":7},"query":null},"op":"u","ts_ms":{"long":1577431358733}}
{"id":1005}     {"before":{"dbserver1.inventory.customers.Value":{"id":1005,"first_name":"Kander","last_name":"Anderson","email":"kander@acme.com"}},"after":null,"source":{"version":"0.10.0.Final","connector":"mysql","name":"dbserver1","ts_ms":1577431388000,"snapshot":{"string":"false"},"db":"inventory","table":{"string":"customers"},"server_id":223344,"gtid":null,"file":"mysql-bin.000003","pos":2911,"row":0,"thread":{"long":7},"query":null},"op":"d","ts_ms":{"long":1577431388716}}
{"id":1005}     null
```
本环境中使用 Avro 格式来发送 binlog, 采用 Avro 格式 Debezium 会将 binlog 数据的 schema 信息发送到 Schema-Registry 中, data 信息发送到 Kafka 中, 这样的方式比 Json 格式少发送冗余的 schema 信息; 可使用一下命令查看对应的 schema 信息
```
curl -XGET http://localhost:8081/subjects/dbserver1.inventory.customers-value/versions/latest
```
更详细的操作信息可以查看 Debezium 官网的 [Tutorial](https://debezium.io/documentation/reference/0.10/tutorial.html) 和 [Avro Serialization](https://debezium.io/documentation/reference/0.10/configuration/avro.html), 以及 Schema-Registry 官网的 [Schema Management](https://docs.confluent.io/current/schema-registry/index.html) 和 [Schema-Registry](https://github.com/confluentinc/schema-registry)

#### 使用 Hudi 摄取数据
##### 使用前的填坑
在摄取 binlog 数据之前, 需要先填几个 Hudi 的坑
- 对 `hoodie.datasource.write.recordkey.field` 配置支持 `|` 的配置
例如在摄取 customers 表的 binlog 数据时, 如果想使用表主键 id 作为 recordKey, 会发现无论是配置 `before.id` 还是 `after.id` 都会遇到 null 的情况; 支持 `before.id | after.id` 的配置形式, 取第一个不为 null 的值, 如果都为 null 则抛出异常 (这与原有 Hudi 的逻辑一致)
- 过滤从 Kafka 消费的 `null` 记录
MySQL 删除记录, Debezium 会额外发送一条 `null` (见上述数据) 到 Kafka, 这会导致 Hudi 抛出 NPE
- 使 `TimestampBasedKeyGenerator` 类支持微秒时间戳
原有 `TimestampBasedKeyGenerator` 只支持 `UNIX_TIMESTAMP, DATE_STRING, MIXED` 三种类型, 而 binlog 的 `ts_ms` 为微秒时间戳, 使用 `UNIX_TIMESTAMP` 生成分区时间不正确

##### 创建 Hudi 摄取时的配置
```
# see ${HUDI_HOME}/docker/demo/config/
include=base.properties
# Key fields, for kafka example
hoodie.datasource.write.recordkey.field=before.id|after.id
hoodie.datasource.write.partitionpath.field=ts_ms
# key generator timestamp based
hoodie.datasource.write.keygenerator.class=org.apache.hudi.utilities.keygen.TimestampBasedKeyGenerator
hoodie.deltastreamer.keygen.timebased.timestamp.type=MILLISECONDS_TIMESTAMP
hoodie.deltastreamer.keygen.timebased.output.dateformat=yyyy/MM/dd
# Schema provider props (change to absolute path based on your installation)
hoodie.deltastreamer.schemaprovider.registry.url=http://schema-registry:8081/subjects/dbserver1.inventory.customers-value/versions/latest
# Kafka Source
hoodie.deltastreamer.source.kafka.topic=dbserver1.inventory.customers
#Kafka props
metadata.broker.list=kafkabroker:9092
auto.offset.reset=smallest
schema.registry.url=http://schema-registry:8081
```
使用以上配置文件摄取 Kafka 中的 customers 的 binlog, 使用 `before.id` 或 `after.id` 字段作为 `recordkey`, `ts_ms` 字段作为 `partitionpath`; 在 Hudi 中这两个字段组成 `HoodieKey`, 此类作为 Hudi dataset 中的每行的主键, 如果 `HoodieIndex.isGlobal() == true`, 则 `HoodieKey` 只依赖 `recordkey` 作为主键
##### 使用 HoodieDeltaStreamer 摄取数据
使用以下命令从 Kafka 摄取 binlog 到 HDFS 中
```Shell
spark-submit \
--class org.apache.hudi.utilities.deltastreamer.HoodieDeltaStreamer $HUDI_UTILITIES_BUNDLE \
--storage-type COPY_ON_WRITE \
--source-class org.apache.hudi.utilities.sources.AvroKafkaSource \
--source-ordering-field ts_ms \
--target-base-path /user/hive/warehouse/customers_cow_timestamp \
--target-table customers_cow_timestamp \
--props file:///var/hoodie/ws/docker/demo/config/partition-table-with-schema-registry.properties  \
--schemaprovider-class org.apache.hudi.utilities.schema.SchemaRegistryProvider
```
以上命令会采用写时合并的方式摄取 binlog 数据为 Hudi dataset; 由于使用 `ts_ms` (数据处理时间) 作为分区, 可能会导致一行业务数据产生多行 Hudi dataset 的数据, 虽然 recordkey 相同但 partitionpath 可能不同; 可以使用以下方法来处理这一问题
- 在业务数据中添加 `create_time` 字段, 在 Hudi 摄取时将其作为 partitionpath
- 将 `hoodie.index.type` 设置为 `GLOBAL_BLOOM`
- 将 `hoodie.datasource.write.keygenerator.class` 设置为 `org.apache.hudi.NonpartitionedKeyGenerator`, 将其摄取非分区的 Hudi dataset (推荐)

##### 同步 Hive 元数据
使用以下命令同步 `customers_cow_timestamp` 的元数据到 Hive Metastore 中
```
/var/hoodie/ws/hudi-hive/run_sync_tool.sh  \
--jdbc-url jdbc:hive2://hiveserver:10000 \
--user hive \
--pass hive \
--partitioned-by inc_day \
--base-path /user/hive/warehouse/customers_cow_timestamp \
--database default \
--table customers_cow_timestamp
```
这里将分区列命名为了 `inc_day`, 接下来就可以在 Hiveserver 查询数据了, 可以看到表里有几个 Hudi 的字段数据
```
0: jdbc:hive2://hiveserver:10000> select * from customers_cow_timestamp t limit 1;
+------------------------+-------------------------+-----------------------+---------------------------+------------------------------------------------------------------------+-----------+---------------------------------------------------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------+----------------+-------------+--+
| t._hoodie_commit_time  | t._hoodie_commit_seqno  | t._hoodie_record_key  | t._hoodie_partition_path  |                          t._hoodie_file_name                           | t.before  |                                     t.after                                     |                                                                                                               t.source                                                                                                               | t.op  |    t.ts_ms     |  t.inc_day  |
+------------------------+-------------------------+-----------------------+---------------------------+------------------------------------------------------------------------+-----------+---------------------------------------------------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------+----------------+-------------+--+
| 20191226075718         | 20191226075718_0_1      | 1003                  | 2019/12/26                | 29849273-f119-4994-a8a4-0a2137d4ba58-0_0-21-21_20191226075718.parquet  | NULL      | {"id":1003,"first_name":"Edward","last_name":"Walker","email":"ed@walker.com"}  | {"version":"0.10.0.Final","connector":"mysql","name":"dbserver1","ts_ms":0,"snapshot":"true","db":"inventory","table":"customers","server_id":0,"gtid":null,"file":"mysql-bin.000003","pos":154,"row":0,"thread":null,"query":null}  | c     | 1577346949947  | 2019-12-26  |
+------------------------+-------------------------+-----------------------+---------------------------+------------------------------------------------------------------------+-----------+---------------------------------------------------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------+----------------+-------------+--+
```

##### 将 binlog 的 Hudi dataset 派生为业务数据的 Hudi dataset
以上已经完成了将业务数据的 binlog 增量摄取的过程, 接下来将 binlog 表派生为业务数据表; 在 spark-shell 中将 `customers_cow_timestamp` 的数据处理为想要的 DataFrame, 将 DataFrame 写入一个新的 Hudi dataset
```
# spark/bin/spark-shell --jars /var/hoodie/ws/packaging/hudi-spark-bundle/target/hudi-spark-bundle-0.5.0-incubating.jar --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer'
// 导入类 (如果使用一些配置常量)
scala> import scala.collection.JavaConversions._
scala> import org.apache.spark.sql.SaveMode._
scala> import org.apache.hudi.DataSourceReadOptions._
scala> import org.apache.hudi.DataSourceWriteOptions._
scala> import org.apache.hudi.config.HoodieWriteConfig._
// 设定增量开始时间 ("000" 表示从头开始), 以及数据根路径
scala> val beginTime = "000"
scala> val basePath = "/user/hive/warehouse/customers_cow_timestamp"
// 获取增量 binlog 数据
scala> val df = spark.
     |      read.
     |      format("org.apache.hudi").
     |      option("hoodie.datasource.view.type", "incremental").
     |      option("hoodie.datasource.read.begin.instanttime", beginTime).
     |      load(basePath)
scala>      
// 获取增量 binlog 数据中的 insert 和 update 业务数据     
scala> val upserts = df.filter("op='c' or op = 'u'").select("after.*")    
scala> upserts.show()
+----+----------+---------+--------------------+
|  id|first_name|last_name|               email|
+----+----------+---------+--------------------+
|1003|    Edward|   Walker|       ed@walker.com|
|1001|     Sally|   Thomas|sally.thomas@acme...|
|1004|      Anne|Kretchmar|  annek@noanswer.org|
|1002|    George|   Bailey|  gbailey@foobar.com|
+----+----------+---------+--------------------+
scala>
// 获取增量 binlog 数据中的 delete 业务数据
scala> val temp = df.filter("op='d'").select("before.*")
scala> temp.show()
+----+-------------+-------------+-----------------+
|  id|   first_name|    last_name|            email|
+----+-------------+-------------+-----------------+
|1005|       Kander|     Anderson|  kander@acme.com|
+----+-------------+-------------+-----------------+
// 将其注册为临时表
scala> deletes.createOrReplaceTempView("tbl")
// 将非 id 字段设置为 null, 设置为 null 字段需保持原有字段类型, 否则 deletes 将与 upserts 不兼容
scala> val deletes = spark.sql("select id, cast(null as string) as first_name, cast(null as string) as last_name, cast(null as string) as email from tbl")
scala> deletes.show()
+----+----------+---------+-----+
|  id|first_name|last_name|email|
+----+----------+---------+-----+
|1005|      null|     null| null|
+----+----------+---------+-----+
scala>
// 派生表根路径
scala> val derivedBasePath = "/user/hive/warehouse/customers_cow_derived"
// 写入 upserts
scala> upserts.write
scala> .format("org.apache.hudi")
scala> .option("hoodie.datasource.write.recordkey.field", "id")
scala> .option("hoodie.datasource.write.partitionpath.field", "email")
scala> .option("hoodie.datasource.write.precombine.field", "email")
scala> .option("hoodie.datasource.write.keygenerator.class", "org.apache.hudi.NonpartitionedKeyGenerator")
scala> .option("hoodie.table.name", "customers_cow_derived")
scala> .mode("Overwrite")
scala> .save(derivedBasePath)
// 写入 deletes
scala> deletes.write
scala> .format("org.apache.hudi")
scala> .option("hoodie.datasource.write.recordkey.field", "id")
scala> .option("hoodie.datasource.write.partitionpath.field", "id")
scala> .option("hoodie.datasource.write.precombine.field", "id")
scala> .option("hoodie.datasource.write.keygenerator.class", "org.apache.hudi.NonpartitionedKeyGenerator")
scala> .option("hoodie.table.name", "customers_cow_derived")
scala> .mode("Append")
scala> .save(derivedBasePath)
scala>
// 查看派生表数据
scala> val derived = spark.read.format("org.apache.hudi").load(derivedBasePath)
scala> derived.show()
+-------------------+--------------------+------------------+----------------------+--------------------+----+----------+---------+--------------------+
|_hoodie_commit_time|_hoodie_commit_seqno|_hoodie_record_key|_hoodie_partition_path|   _hoodie_file_name|  id|first_name|last_name|               email|
+-------------------+--------------------+------------------+----------------------+--------------------+----+----------+---------+--------------------+
|     20191226091544|  20191226091544_0_1|              1001|                      |dab6dbba-219a-4aa...|1001|     Sally|   Thomas|sally.thomas@acme...|
|     20191226091544|  20191226091544_0_2|              1002|                      |dab6dbba-219a-4aa...|1002|    George|   Bailey|  gbailey@foobar.com|
|     20191226091544|  20191226091544_0_3|              1003|                      |dab6dbba-219a-4aa...|1003|    Edward|   Walker|       ed@walker.com|
|     20191226091544|  20191226091544_0_4|              1004|                      |dab6dbba-219a-4aa...|1004|      Anne|Kretchmar|  annek@noanswer.org|
|     20191226100319|  20191226100319_0_1|              1005|                      |dab6dbba-219a-4aa...|1005|      null|     null|                null|
+-------------------+--------------------+------------------+----------------------+--------------------+----+----------+---------+--------------------+
```
派生表 `customers_cow_derived` 这里生成为了非分区表

#### 小结
以上展示了使用 Hudi 构建增量 pipeline 的过程, 可以看到 Hudi 基本可以支持以下场景
- 增量单表
- 增量事实表与全量维度表生成增量明细表

除此之外, 对于增量表与增量表的 join, 由于时间窗中的数据不对等可能会产生应该 join 的数据未能成功 join, 此场景暂时尚未找到 Hudi 的实现方案

>**参考:**
- [Debezium](https://debezium.io/)
- [Apache Hudi](https://hudi.apache.org)
- [Schema Registry](https://github.com/confluentinc/schema-registry)
- [Hudi: Uber Engineering’s Incremental Processing Framework on Apache Hadoop](https://eng.uber.com/hoodie/)
- [Building robust CDC pipeline with Apache Hudi and Debezium](https://www.slideshare.net/SyedKather/building-robust-cdc-pipeline-with-apache-hudi-and-debezium)
- [Yotpo构建零延迟数据湖实践](https://mp.weixin.qq.com/s/JMv2wZdRNvxHsU618l6DlQ)
