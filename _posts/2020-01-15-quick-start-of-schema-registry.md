---
layout: post
title: "Schema Registry 快速入门"
date: "2020-01-15"
description: "Schema Registry 快速入门"
tag: [Kafka, Schema Registry]
---

### Schema Registry 快速入门
Confluent Schema Registry 为你的元数据提供一个服务层, 为存储和查找 Apache Avro schema 提供了 RESTFUL 接口; 它存储基于 [subject name strategy](https://docs.confluent.io/current/schema-registry/serializer-formatter.html#sr-avro-subject-name-strategy) 的 schema 的所有历史版本, 提供多种[兼容性设定](https://docs.confluent.io/current/schema-registry/avro.html#schema-evolution-and-compatibility)并且允许依据配置的兼容性设定进行演进来扩展 Avro 的支持; 它为 Apache Kafka 客户端提供序列化插件, 对用 Avro 格式发送的 Kafka 消息处理 schema 存储和查找  
Schema Registry 与你的 Kafka brokers 是分离开的, 你的生产者和消费者仍然向 Kafka 的 topic 发布以及读取数据; 同时它们也会向 Schema Registry 发送和查找 schema来描述消息的数据格式  
![Confluent Schema Registry for storing and retrieving Avro schemas](https://docs.confluent.io/current/_images/schema-registry-and-kafka.png)  
Schema Registry 是使用 Kafka 作为它的底层存储的一个分布式的 Avro Schemas 存储层; 一些设计点如下
- 为每个 schema 分配全局唯一 ID, 分配的 ID 保证是单调递增的但是不保证是连续的
- Kafka 提供持久后端, 并充当 Schema Registry 及其所包含架构的状态的预写更改日志
- Schema Registry 被设计为单主节点的分布式的架构, ZooKeeper 或 Kafka 协助主节点选举

#### 获取安装包
有以下两种方式可以获取
#### 下载 Confluent Platform
[Confluent Platform 全家桶](https://www.confluent.io/download/)  包含了 Schema Registry 组件,如果已经部署了自己的 Kafka 和 ZooKeeper 可以只启动 Schema Registry 组件
#### 自行编译
编译 Schema Regsitry 需要先编译依赖的 [kafka]( https://github.com/confluentinc/kafka), [common](https://github.com/confluentinc/common), [rest-utils](https://github.com/confluentinc/rest-utils), FAQ 见[这里] (https://github.com/confluentinc/schema-registry/wiki/FAQ); 可能遇到的报错如下
- 编译遇到包下载问题可以设置代理, gradle 编译时下载依赖包到一半时报错 `Read timeout`, 可通过设置 ` -Dorg.gradle.internal.http.connectionTimeout=300000 -Dorg.gradle.internal.http.socketTimeout=300000` 参数重试, 参数定义见使用版本的 [gradle 代码 (v5.6.2)](https://github.com/gradle/gradle/blob/v5.6.2/subprojects/resources-http/src/main/java/org/gradle/internal/resource/transport/http/JavaSystemPropertiesHttpTimeoutSettings.java)  
- 启动时报错缺少 rest-utils 或 common 包中的类, 可将对应版本 Confluent Platform 全家桶中的 `${CONFLUENT}\share\java\rest-utils` 和 `${CONFLUENT}\share\java\confluent-common` 拷贝到 `${SCHEMA_REGISTRY}\share\java` 目录下
- 重启 Schema Registry 时报错 [`The broker does not support DESCRIBE_CONFIGS`]
报错详情如 [issues#814](https://github.com/confluentinc/schema-registry/issues/814) 或 [questions#59203482](https://stackoverflow.com/questions/59203482/why-am-i-unable-to-restart-schema-registry-after-spinning-up-new-pods-in-kuberne), 可以参考 [兼容性列表](https://docs.confluent.io/current/installation/versions-interoperability.html#cp-and-apache-kafka-compatibility) 部署对应版本的 Kafka

#### 配置
Schema Registry 的配置参数见[这里](https://docs.confluent.io/current/schema-registry/installation/config.html), 虽然参数很多但大部分都是 ssl 相关的参数, 如无安全要求配置的核心参数如下
```
listeners=http://0.0.0.0:8081
kafkastore.connection.url=localhost:2181
kafkastore.topic=_schemas
debug=false
# Schema Registry 集群时才需要, 多个实例需要配置成相同值
kafkastore.group.id=schema-registry-cluter
```
除了单机, 单数据中心集群方式还有多数据中心集群部署, 详细可见[这里](https://docs.confluent.io/3.0.0/schema-registry/docs/deployment.html#multi-dc-setup)

#### 读写 Kafka Avro 格式数据
Kafka 生产者写数据到 topic 中, Kafka 消费者重 topic 中读取数据, 这意味着一个约定: 生产者写入数据的 schema 能够被消费者读取, 甚至生产者和消费者能偶演进这个 schema; Schema Registry 能够帮助确保这个约定满足兼容性检查  
将 schema 认做为 API 是非常有用的; 应用依赖于 APIs, 并且期盼着 APIs 做的任何变化都是兼容的能保证应用继续运行; 相同的, 流应用依赖于 schema 并期盼着 schema 所做的变化是兼容的能保证继续运行; Schema 演进要求兼容性检查已确保生产者消费者的约定不被破坏; 这正式 Schemas Registry 所做的: 提供集中化 schema 管理并保证 schema 演进的兼容性检查  

这里使用 [user.avsc](http://avro.apache.org/docs/current/gettingstartedjava.html#Defining+a+schema) 作为数据 的 schema
##### 依赖 jar
```
<-- ... -->
<dependencies>
    <dependency>
        <groupId>org.apache.avro</groupId>
        <artifactId>avro</artifactId>
        <version>1.9.1</version>
    </dependency>
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-clients</artifactId>
        <version>2.1.1</version>
    </dependency>
    <dependency>
        <groupId>io.confluent</groupId>
        <artifactId>kafka-avro-serializer</artifactId>
        <version>5.4.0</version>
    </dependency>
    <-- ... -->
</dependencies>
<-- ... -->
<build>
    <-- ... -->
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.8.0</version>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
            </configuration>
        </plugin>
        <plugin>
            <groupId>org.apache.avro</groupId>
            <artifactId>avro-maven-plugin</artifactId>
            <version>1.9.1</version>
            <executions>
                <execution>
                    <phase>generate-sources</phase>
                    <goals>
                        <goal>schema</goal>
                    </goals>
                    <configuration>
                        <sourceDirectory>${project.basedir}/src/main/avro/</sourceDirectory>
                        <outputDirectory>${project.basedir}/src/main/java/</outputDirectory>
                    </configuration>
                </execution>
            </executions>
        </plugin>
        <-- ... -->
    </plugins>
</build>
```

##### 写 Kafka Avro 格式数据
```Java
public class SchemaConsumer {

    public static void main(String[] args) {
        String brokers = "localhost:9092";
        String registry = "http://localhost:8081";
        String topic = "example.avro.User";

        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, brokers);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, KafkaAvroDeserializer.class);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "users");
        props.put(AbstractKafkaAvroSerDeConfig.SCHEMA_REGISTRY_URL_CONFIG, registry);

        try (final KafkaConsumer<String, GenericRecord> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Collections.singletonList(topic));
            while (true) {
                final ConsumerRecords<String, GenericRecord> records = consumer.poll(100);
                for (final ConsumerRecord<String, GenericRecord> record : records) {
                    final String key = record.key();
                    final GenericRecord value = record.value();
                    System.out.printf("key = %s, value = %s%n", key, value);
                }
            }
        }
    }
}
```
##### 写 Kafka Avro 格式数据
```Java
public class SchemaProducer {

    public static void main(String[] args) throws IOException {
        String brokers = "localhost:9092";
        String registry = "http://localhost:8081";
        String topic = "example.avro.User";

        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, brokers);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class);
        props.put(AbstractKafkaAvroSerDeConfig.SCHEMA_REGISTRY_URL_CONFIG, registry);
        KafkaProducer<String,GenericRecord> producer = new KafkaProducer<>(props);

        String key = "Alyssa";
        Schema schema = new Schema.Parser().parse(new File("./demo-avro/src/main/avro/user.avsc"));
        GenericRecord value = new GenericData.Record(schema);
        value.put("name", "Alyssa");
        value.put("favorite_number", 128);
        // 发送
        ProducerRecord<String,GenericRecord> record = new ProducerRecord<>(topic, key, value);
        producer.send(record);
        producer.flush();
        producer.close();
    }
}
```

此外, Schema Regsitry 还提供了 [Schema Registry Maven Plugin](https://docs.confluent.io/current/schema-registry/develop/maven-plugin.html), 可用于 schema 的下载, 注册, 测试兼容性

>**参考:**
- [Schema Registry](https://github.com/confluentinc/schema-registry)
- [Schema Management](https://docs.confluent.io/current/schema-registry/index.html)
- [Schema Registry Tutorial](https://docs.confluent.io/current/schema-registry/schema_registry_tutorial.html)
- [Kafka Schema Registry 原理](https://zhmin.github.io/2019/04/23/kafka-schema-registry/)
