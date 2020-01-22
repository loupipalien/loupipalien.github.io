---
layout: post
title: "Apache Avro 入门"
date: "2020-01-08"
description: "Apache Avro 入门"
tag: [Avro]
---

### Apache Avro 入门
Apache Avro 是一个数据序列化系统, 有以下特性
- 丰富的数据结构
- 紧凑的二进制数据格式
- 存储持久数据的容器文件
- 远程程序调用 (RPC)
- 可以与动态语言轻松整合; 使用或实现 RPC 协议读取或写入数据文件均不需要代码生成, 代码生成作为可选的优化, 仅对于静态类型的语言值得实现

#### 下载 Avro 的实现包
这里使用 Maven 构建 Demo, 添加以下以依赖在 POM 文件中
```
<dependency>
  <groupId>org.apache.avro</groupId>
  <artifactId>avro</artifactId>
  <version>1.9.1</version>
</dependency>
```
为了方便的生成代码可添加以下 Maven plugin
```
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
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
```

#### 定义 schema
Avro scheama 使用 JSON 格式定义, schema 由 [primitive types](http://avro.apache.org/docs/current/spec.html#schema_primitive) (null, boolean, int, long, float, double, bytes, string) 和 [complex types](http://avro.apache.org/docs/current/spec.html#schema_complex) (record, enum, array, map, union, fixed) 组成; 以下是一个简单的 schema 示例: `user.avro`
```
{"namespace": "example.avro",
 "type": "record",
 "name": "User",
 "fields": [
     {"name": "name", "type": "string"},
     {"name": "favorite_number",  "type": ["int", "null"]},
     {"name": "favorite_color", "type": ["string", "null"]}
 ]
}
```
本 schema 定义了一个假想的 user 记录 (需要注意的是一个 schema 文件只能包含一个 schema 的定义); 最简约的情况下一个记录的定义必须包含它的类型 ` ("type": "record")`, 名称 `("name": "User")` 以及字段, 此示例中字段为 `name, favorite_number, favorite_color`; 我们也可以定义一个命名空间 `("namespace": "example.avro")`, 它将和名称属性定义的一起定义 schema 的全限定名 `(example.avro.User)`  
字段被一组对象定义, 其中包括名称和类型 (其他属性是可选的, 更多细节见 [record specification](http://avro.apache.org/docs/current/spec.html#schema_record)); 字段的类型属性是另一个 schema 对象, 它可以是一个原生类型或者复杂类型; 例如, User schema 的名称字段是原生类型 `string`, 然而 `favorite_number` 和 `favorite_color` 字段是 `unions` 类型, 用 JSON 数组表示; `unions` 是一个复杂类型, 可以是其中包含的类型列表的任意一个; `favorite_number` 可以是 `int` 或者 `null`, 使其可以成为可选字段

#### 使用代码生成进行序列化和反序列化
因为引入了对应的 Maven plugin, 在编译时可以按照 `${project.basedir}/src/main/avro/` 目录中的 Avro schema 文件(例如 `user.avsc`) 生成对应的 Java 类; 以下是使用生成的代码做序列化反序列化的过程
```Java
public class App {
    public static void main(String[] args) throws IOException {
        App app = new App();
        app.serdeSpecificRecordClass();
    }

    private void serdeSpecificRecordClass() throws IOException {
        User user1 = new User();
        user1.setName("Alyssa");
        user1.setFavoriteNumber(256);
        // Leave favorite color null

        // Alternate constructor
        User user2 = new User("Ben", 7, "red");

        // Construct via builder
        User user3 = User.newBuilder()
                .setName("Charlie")
                .setFavoriteColor("green")
                .setFavoriteNumber(null)
                .build();

        // Serialize user1, user2 and user3 to disk
        DatumWriter<User> userDatumWriter = new SpecificDatumWriter<>(User.class);
        DataFileWriter<User> dataFileWriter = new DataFileWriter<>(userDatumWriter);
        dataFileWriter.create(user1.getSchema(), new File("users.avro"));
        dataFileWriter.append(user1);
        dataFileWriter.append(user2);
        dataFileWriter.append(user3);
        dataFileWriter.close();

        // Deserialize Users from disk
        DatumReader<User> userDatumReader = new SpecificDatumReader<>(User.class);
        DataFileReader<User> dataFileReader = new DataFileReader<>(new File(dir + "/users.avro"), userDatumReader);
        User user = null;
        while (dataFileReader.hasNext()) {
            // Reuse user object by passing it to next(). This saves us from
            // allocating and garbage collecting many objects for files with
            // many items.
            user = dataFileReader.next(user);
            System.out.println(user);
        }
    }
}    
```

#### 不使用代码生成进行序列化和反序列化
```Java
public class App {
    public static void main(String[] args) throws IOException {
        App app = new App();
        app.serdeGenericRecordClass();
    }

    private void serdeGenericRecordClass() throws IOException {
        Schema schema = new Schema.Parser().parse(new File("/src/main/avro/user.avsc"));

        GenericRecord user1 = new GenericData.Record(schema);
        user1.put("name", "Alyssa");
        user1.put("favorite_number", 256);
        // Leave favorite color null

        GenericRecord user2 = new GenericData.Record(schema);
        user2.put("name", "Ben");
        user2.put("favorite_number", 7);
        user2.put("favorite_color", "red");

        String dir = this.getClass().getClassLoader().getResource("").getPath();

        // Serialize user1, user2 and user3 to disk
        DatumWriter<GenericRecord> userDatumWriter = new GenericDatumWriter<>(schema);
        DataFileWriter<GenericRecord> dataFileWriter = new DataFileWriter<>(userDatumWriter);
        dataFileWriter.create(schema, new File("users.avro"));
        dataFileWriter.append(user1);
        dataFileWriter.append(user2);
        dataFileWriter.close();

        // Deserialize Users from disk
        DatumReader<GenericRecord> userDatumReader = new GenericDatumReader<>(schema);
        DataFileReader<GenericRecord> dataFileReader = new DataFileReader<>(new File("users.avro"), userDatumReader);
        GenericRecord record = null;
        while (dataFileReader.hasNext()) {
            // Reuse user object by passing it to next(). This saves us from
            // allocating and garbage collecting many objects for files with
            // many items.
            record = dataFileReader.next(record);
            System.out.println(record);
        }
    }
}
```

>**参考:**
- [Apache Avro™ 1.9.1 Getting Started (Java)](http://avro.apache.org/docs/current/gettingstartedjava.html)
