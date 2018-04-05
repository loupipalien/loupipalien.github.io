---
layout: post
title: "如何部署 livy"
date: "2018-04-03"
description: "如何部署 livy"
tag: [livy, spark]
---
**环境:** CentOS-6.8-x86_64, spark-2.1.0-bin-hadoop2.7

### 安装 Livy
下载 [Livy 安装包](http://livy.apache.org/download)

### 运行 Livy
为了运行 Livy 服务器, 你需要有一个已安装的 Spark; 可以 [这里](https://spark.apache.org/downloads.html) 获取 Spark 发布包; Livy 要求 Spark 至少为 1.6 的版本, 支持 Scala 2.10 和 2.11 构建的 Spark; 为了使用本地会话运行 Livy, 首先要 export 以下变量
```
export SPARK_HOME=/app/spark
export HADOOP_CONF_DIR=/app/hadoop/etc/hadoop
```
然后使用以下命令启动服务器
```
${LIVY_HOME}/bin/livy-server
```
Livy 默认使用 `SPARK_HOME` 下的 Spark 配置; 你可以在启动 Livy 之前通过设置 `SPARK_CONF_DIR` 环境变量来覆盖 Spark 的配置  
强烈推荐配置 Spark 使用 YARN cluster 模式提交应用; 这样确保用户会话在 YARN cluster 模式下正确的使用它们的资源, 当有多用户会话运行时 Livy 所在的机器不会过载

### 配置 Livy
Livy 使用了在配置目录下的一些配置文档, 默认的配置目录在 Livy 安装目录下; 当启动 Livy 时, 可通过设置 `LIVY_CONF_DIR` 环境变量来设置配置目录; Livy 使用的配置文件如下
- livy.conf: 包含服务器配置, Livy 发布包包含一个默认的配置文件末班, 其中列举了一些配置键和对应的默认值
- spark-blacklist.conf: 枚举出不允许用户覆盖的一些 Spark 配置项; 这些选项将被限制为它们的默认值，或者是 Livy 使用的 Spark 配置中设置的值
- log4j.properties: 配置 Livy 的日志; 定义日志级别以及日志信息要写到哪里, 默认的配置模板将打印日志到标准输出流

#### 开始使用 Livy
一旦 Livy 开始运行, 你就可以通过 8998 端口去连接 (这个可以通过 `livy.server.port` 配置项更改); [这里](http://livy.apache.org/examples) 提供了一些快速开始的例子, 或者你可以查看 API 文档
- [REST API](http://livy.apache.org/docs/latest/rest-api.html)
- [Programmatic API](http://livy.apache.org/docs/latest/programmatic-api.html)

>**参考:**
[Livy Getting Started](http://livy.apache.org/get-started/)
