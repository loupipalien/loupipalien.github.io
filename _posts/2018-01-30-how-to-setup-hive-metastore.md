---
layout: post
title: "如何安装 Hive Metastore"
date: "2018-01-30"
description: "如何安装 Hive Metastore"
tag: [hive]
---
**前提:** 了解 Hadoop 和 Hive, 并搭建了 Hadoop 集群和安装了 Mysql
**环境:** CentOS-6.8-x86_64, hadoop-2.7.3, hive-2.1.1

### 修改配置文件 hive-env.sh
```
...
HADOOP_HOME=/app/hadoop
...
```
### 修改配置文件 hive-site.xml (从将 hive-default.xml.tempalate 拷贝并重命名)
```
...
<!-- 元数据存储数据库的密码 -->
<property>
  <name>javax.jdo.option.ConnectionPassword</name>
  <value>123456</value>
  <description>password to use against metastore database</description>
</property>
...
<!-- 元数据的数据库链接 -->
<property>
  <name>javax.jdo.option.ConnectionURL</name>
  <value>jdbc:mysql://ltchen03:3306/hive?createDatabaseIfNotExist=true&amp;useSSL=true</value>
  <description>
    JDBC connect string for a JDBC metastore.
    To use SSL to encrypt/authenticate the connection, provide database-specific SSL flag in the connection URL.
    For example, jdbc:postgresql://myhost/db?ssl=true for postgres database.
  </description>
</property>
...
<!-- 元数据连接驱动名 -->
<property>
  <name>javax.jdo.option.ConnectionDriverName</name>
  <value>com.mysql.jdbc.Driver</value>
  <description>Driver class name for a JDBC metastore</description>
</property>
...
<!-- 元数据连接用户名 -->
<property>
  <name>javax.jdo.option.ConnectionUserName</name>
  <value>root</value>
  <description>Username to use against metastore database</description>
</property>
...
```

### 添加链接数据库驱动 Jar 包
下载与安装的 Mysql 兼容版本的 mysql-connector-java-x.x.xx-bin.jar 文件, 拷贝到 lib 目录下

### 初始化元数据库
```
$ bin/schematool -initSchema -dbType mysql
```
初始化完成后有相应提示, 也可查看数据库中初始化的 hive 库

### 启动 Hive metastore (用 nohup 命令放置后台运行)
```
$ nohup bin/hive --service metastore &
```

### 启动 CLI 客户端验证服务
```
$ bin/hive
```
可能会报错 ${system:java.io.tmpdir} 路径找不到, 这是因为环境变量不能被解析, 将客户端上的 hive-site.xml 中以下两个配置修改为对应值,后再次启动 cli 客户端即可
```
...
<property>
  <name>hive.exec.local.scratchdir</name>
  <value>${system:java.io.tmpdir}/${system:user.name}</value>
  <description>Local scratch space for Hive jobs</description>
</property>
<property>
  <name>hive.downloaded.resources.dir</name>
  <value>${system:java.io.tmpdir}/${hive.session.id}_resources</value>
  <description>Temporary local directory for added resources in the remote file system.</description>
</property>
...
```

### 初始化元数据库时可能会遇到的警告
```
Sta Jan 8 15:59:32 CST 2018 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
```
原因是 Mysql 在高版本需要指明是否进行 SSL 连接, 所以在连接数据库的URL后加上 useSSL=true, 修改为
```
jdbc:mysql://ltchen03:3306/hive?createDatabaseIfNotExist=true&useSSL=true
```
删掉数据库中的元数据再次初始化结果出现
```
[Fatal Error] hive-site.xml:496:83: The reference to entity "useSSL" must end with the ';' delimiter.Exception in thread "main" java.lang.RuntimeException: org.xml.sax.SAXParseException; systemId: file:/usr/local/hive-2.1.1/conf/hive-site.xml; lineNumber: 496; columnNumber: 83; The reference to entity "useSSL" must end with the ';' delimiter.
```
原因是在 xml 文件中 `&` 字符要进行转义替换为 `&amp;`, 最后将 URL 修改为
```
jdbc:mysql://ltchen03:3306/hive?createDatabaseIfNotExist=true&amp;useSSL=true
```

>**参考:**
[Hadoop集群之Hive安装配置](http://yanliu.org/2015/08/13/Hadoop%E9%9B%86%E7%BE%A4%E4%B9%8BHive%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE/)  
[hive2.1.0安装(hadoop2.7.2环境)](https://kaimingwan.com/post/da-shu-ju/hive2.1.0an-zhuang-hadoop2.7.2huan-jing)
