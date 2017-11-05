---
layout: post
title: "如何部署 spark 集群"
date: "2017-11-05"
description: "如何部署 spark 集群"
tag: [spark]
---
**前提假设:** 各节点主机名配置, 防火墙处理, ssh配置, jdk安装  
**环境:** CentOS-6.8-x86_64, spark-2.1.0-bin-hadoop2.7

### 部署节点

|节点|软件|进程|
|-|-|-|
|test01|jdk, spark|Master|
|test02|jdk, spark|Worker|
|test03|jdk, spark|Worker|
|test04|jdk, spark|Worker|
|test05|jdk, spark|Worker|

### 部署步骤
- 解压 spark-2.1.0-bin-hadoop2.7.tar.gz 安装包到欲安装目录 ${SPARK_HOME}
- 修改配置文件, 配置文件都在 ${SPARK_HOME}/conf 目录下, 如有对应配置文件可直接修改, 没有则拷贝对应模板文件改名(在 test01 上进行)
  - spark-env.sh
```
...
# 不设置 JAVA_HOME 可能会导致 Worker 不能启动
export JAVA_HOME=/app/java
```
  - slaves
```
test02
test03
test04
test05
```

### 启动并查看服务
`${SPARK_HOME}/sbin/start-all.sh`, 可启动对应节点的 Master, Worker 服务  
http://test01:8080, 可查看 Master 以及 Worker 服务信息

### 集群管理器
Spark 集群可以在各种集群管理器上运行, 例如 Spark 自带的 Standalone 集群管理器, Hadoop YARN, Apache Mesos 等
- Standalone  
按照以上部署步骤集群默认使用 Standalone 集群管理器
- YARN  
如果需要 Hadoop YARN 作为集群管理器(Spark on YARN), 那么需要在 ${SPARK_HOME}/conf/spark-env.sh 加入 Hadoop 配置文件夹路径
```
...
export HADOOP_CONF_DIR=/app/hadoop/etc/hadoop
```
- Mesos  
暂无环境

>**参考:**  
[Spark Standalone Mode](https://spark.apache.org/docs/2.1.0/spark-standalone.html)  
[Running Spark on YARN](https://spark.apache.org/docs/2.1.0/running-on-yarn.html)  
[Running Spark on Mesos](https://spark.apache.org/docs/2.1.0/running-on-mesos.html)  
