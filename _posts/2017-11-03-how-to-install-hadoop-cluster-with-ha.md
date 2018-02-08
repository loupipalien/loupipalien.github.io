---
layout: post
title: "如何安装 hadoop 高可用集群"
date: "2017-11-03"
description: "如何安装 hadoop 高可用集群"
tag: [hadoop, ha]
---


**前提假设:** 各节点主机名配置, 防火墙处理, ssh配置, jdk和zookeeper安装  
**环境:** CentOS-6.8-x86_64, hadoop-2.7.3

### 部署节点

|节点|软件|进程|
|-|-|-|
|test01|jdk, hadoop|NameNode, ResourceManager, DFSZKFailoverController|
|test02|jdk, hadoop|NameNode, ResourceManager, DFSZKFailoverController|
|test03|jdk, hadoop, zookeeper|DatanNode, NodeManager, JournalNode, QuorumPeerMain|
|test04|jdk, hadoop, zookeeper|DatanNode, NodeManager, JournalNode, QuorumPeerMain|
|test05|jdk, hadoop, zookeeper|DatanNode, NodeManager, JournalNode, QuorumPeerMain|
|...|jdk, hadoop|DatanNode, NodeManager|
|test10|jdk, hadoop|DatanNode, NodeManager|

### 安装步骤
- 解压 hadoop-2.7.3.tar.gz 安装包到欲安装目录 ${HADOOP_HOME}
- 修改配置文件, 配置文件都在 ${HADOOP_HOEM}/etc/hadoop 目录下, 如有对应配置文件可直接修改, 没有则拷贝对应模板文件改名(在 test01 上进行)
  - hadoop-env.sh

  ```
  ...
  # The java implementation to use.
  export JAVA_HOME=/app/java
  ...
  ```
  - core-site.xml

  ```
  ...
  <configuration>
  	<!-- 指定hdfs的nameservice为ns1 -->
  	<property>
  		<name>fs.defaultFS</name>
  		<value>hdfs://ns1</value>
  	</property>
  	<!-- 指定hadoop临时目录 -->
  	<property>
  		<name>hadoop.tmp.dir</name>
  		<value>/app/hadoop/tmp</value>
  	</property>
  	<!-- 指定zookeeper地址 -->
  	<property>
  		<name>ha.zookeeper.quorum</name>
  		<value>test03:2181,test04:2181,test05:2181</value>
      </property>
  </configuration>
  ```
  - hdfs-site.xml

  ```
  ...
  <configuration>
      <!--指定hdfs的nameservice为ns1，需要和core-site.xml中的保持一致 -->
  	<property>
  		<name>dfs.nameservices</name>
  		<value>ns1</value>
  	</property>
  	<!-- ns1下面有两个NameNode，分别是nn1，nn2 -->
  	<property>
  		<name>dfs.ha.namenodes.ns1</name>
  		<value>nn1,nn2</value>
  	</property>
  	<!-- nn1的RPC通信地址 -->
  	<property>
  		<name>dfs.namenode.rpc-address.ns1.nn1</name>
  		<value>test01:8020</value>
  	</property>
  	<!-- nn1的http通信地址 -->
  	<property>
  		<name>dfs.namenode.http-address.ns1.nn1</name>
  		<value>test01:50070</value>
  	</property>
  	<!-- nn2的RPC通信地址 -->
  	<property>
  		<name>dfs.namenode.rpc-address.ns1.nn2</name>
  		<value>test02:8020</value>
  	</property>
  	<!-- nn2的http通信地址 -->
  	<property>
  		<name>dfs.namenode.http-address.ns1.nn2</name>
  		<value>test02:50070</value>
  	</property>
  	<!-- 指定NameNode的元数据在JournalNode上的存放位置 -->
  	<property>
  		<name>dfs.namenode.shared.edits.dir</name>
  		<value>qjournal://test03:8485;test04:8485;test05:8485/ns1</value>
  	</property>
  	<!-- 指定JournalNode在本地磁盘存放数据的位置 -->
  	<property>
  		<name>dfs.journalnode.edits.dir</name>
  		<value>/app/hadoop/journal</value>
  	</property>
  	<!-- 开启NameNode失败自动切换 -->
  	<property>
  		<name>dfs.ha.automatic-failover.enabled</name>
  		<value>true</value>
  	</property>
  	<!-- 配置失败自动切换实现方式 -->
  	<property>
  		<name>dfs.client.failover.proxy.provider.ns1</name>
  		<value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
  	</property>
  	<!-- 配置隔离机制方法，多个机制用换行分割，即每个机制暂用一行-->
  	<property>
  		<name>dfs.ha.fencing.methods</name>
  		<value>
  			sshfence
  			shell(/bin/true)
  		</value>
  	</property>
  	<!-- 使用sshfence隔离机制时需要ssh免登陆 -->
  	<property>
  		<name>dfs.ha.fencing.ssh.private-key-files</name>
  		<value>/home/sfapp/.ssh/id_rsa</value>
  	</property>
  	<!-- 配置sshfence隔离机制超时时间 -->
  	<property>
  		<name>dfs.ha.fencing.ssh.connect-timeout</name>
  		<value>30000</value>
  	</property>
  </configuration>
  ```
  - mapred-site.sml

  ```
  ...
  <configuration>
  	<!-- 指定mr框架为yarn方式 -->
  	<property>
  	    <name>mapreduce.framework.name</name>
  	    <value>yarn</value>
  	</property>
  </configuration>
  ```
  - yarn-site.xml

  ```
  ...
  <configuration>
  	<!-- 开启RM高可靠 -->
  	<property>
  	    <name>yarn.resourcemanager.ha.enabled</name>
  	    <value>true</value>
  	</property>
  	<!-- 指定RM的cluster id -->
  	<property>
  	    <name>yarn.resourcemanager.cluster-id</name>
  	    <value>yrc</value>
  	</property>
  	<!-- 指定RM的名字 -->
  	<property>
  	    <name>yarn.resourcemanager.ha.rm-ids</name>
  	    <value>rm1,rm2</value>
  	</property>
  	<!-- 分别指定RM的地址 -->
  	<property>
  	    <name>yarn.resourcemanager.hostname.rm1</name>
  	    <value>test01</value>
  	</property>
  	<property>
  	    <name>yarn.resourcemanager.hostname.rm2</name>
  	    <value>test02</value>
  	</property>
  	<property>
  	    <name>yarn.resourcemanager.webapp.address.rm1</name>
  	    <value>test01:8088</value>
  	</property>
  	<property>
  	    <name>yarn.resourcemanager.webapp.address.rm2</name>
  	    <value>test02:8088</value>
  	 </property>
  	<!-- 指定zk集群地址 -->
  	<property>
  	    <name>yarn.resourcemanager.zk-address</name>
  	    <value>ltchen03:2181,ltchen04:2181,ltchen05:2181</value>
  	</property>
  	<!-- NodeManager上运行的附属服务,需配置成mapreduce_shuffle,才可运行MapReduce程序 -->
  	<property>
  	    <name>yarn.nodemanager.aux-services</name>
  	    <value>mapreduce_shuffle</value>
  	</property>
  	<!-- 在ad01上配置rm1,在ad02上配置rm2
  	<property>
  	    <name>yarn.resourcemanager.ha.id</name>
  	    <value>rm1</value>
  	    <description>If we want to launch more than one RM in single node, we need this configuration</description>
  	</property>
  	-->
  </configuration>
  ```
  - slaves

  ```
  test03
  test04
  test05
  test06
  test07
  test08
  test09
  test10
  ```
- 将 test01 上的 ${HADOOP_HOME} 拷贝到其他节点

```
#!/bin/bash
HADOOP_HOME="/app/hadoop"
for i in 02 03 04 05 06 07 08 09 10
    do
        scp -r ${HADOOP_HOME} test${i}:${HADOOP_HOME}
    done
```

### 格式化
- 格式化 NameNode
在格式化之前需要启动 JournalNode 服务 (在 test03, test04, test05 上进行), 执行语句 `${HADOOP_HOME}/sbin/hadoop-daemon.sh start journalnode`; 接下来格式化数据, 在 test01 上执行语句 `${HADOOP_HOME}/sbinhdfs namenode -format`,
格式化后会在根据 core-site.xml 中的 hadoop.tmp.dir 配置生成数据, 将对应文件夹拷贝到 test02 对应目录
- 格式化 ZooKeeper
在格式化之前需要启动 ZooKeeper 服务 (在 test03, test04, test05 上进行), 执行语句 `${ZOOKEEPER_HOME}/bin/zkServer.sh start`; 在 test01 上执行语句 `hdfs zkfc -formatZK`, 这是为了在 ZooKeeper 中创建 hadoop-ha 的节点

### 启动服务
- 启动 HDFS
`${HADOOP_HOME}/sbin/start-dfs.sh`, 可启动对应节点的 NameNode, DataNode, JournalNode 服务
- 启动 YARN
`${HADOOP_HOME}/sbin/start-yarn.sh`, 可启动对应节点的 ResourceManager, NodeManager 服务

### 查看服务并验证 HA
- hdfs: http://test01:50070, http://test02:50070
- yarn: http://test01:8088, http://test02:8088
- 验证 HA, kill 掉 active 状态的 NameNode/ResourceManager, standby 状态的 NameNode/ResourceManager 是否会切换到 active

### TODO
- test01 上执行 `${HADOOP_HOME}/sbin/start-yarn.sh` 没有启动 test02 节点上的 ResourceManager 服务, 只好手动启动 ` ${HADOOP_HOME}/sbin/yarn-daemon.sh start resourcemanager` ;尝试了 yarn.resourcemanager.ha.id 配置也未生效, 可能是哪里姿势不对, 后续解决

>**参考:**  
[Setting up a Single Node Cluster](http://hadoop.apache.org/docs/r2.7.3/hadoop-project-dist/hadoop-common/SingleCluster.html)  
[HDFS High Availability Using the Quorum Journal Manager](http://hadoop.apache.org/docs/r2.7.3/hadoop-project-dist/hadoop-hdfs/HDFSHighAvailabilityWithQJM.html)  
[ResourceManager High Availability](http://hadoop.apache.org/docs/r2.7.3/hadoop-yarn/hadoop-yarn-site/ResourceManagerHA.html)
