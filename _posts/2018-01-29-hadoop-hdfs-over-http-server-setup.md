---
layout: post
title: "Hadoop 的 HttpFS 安装"
date: "2018-01-29"
description: "Hadoop 的 HttpFS 安装"
tag: [Hadoop, httpfs]
---

> 本文主要叙述在伪认证的 Hadoop 集群上快速安装伪认证的 HttpFS  

### 安装 HttpFS
```
$ tar -zxvf httpfs-2.7.5.tar.gz
```
如果安装了 hadoop, hadoop 中默认带了 httpfs, 只需要配置启动服务即可

### 配置 HttpFS
默认的, HttpFS 的配置文件是和 Hadoop 的配置文件 (core-site.xml 和 hdfs-site.xml) 在同一个目录中; 如果不是这样, 那么需要在 httpfs-site.xml 文件添加 httpfs.hadoop.config.dir 的属性, 设置其值为 Hadoop 配置文件目录地址

### 配置 Hadoop
编辑 Hadoop 的 core-site.xml 文件, 定义一个 Unix 系统用户作为代理用户运行 HttpFS, 例如:
```
...
<property>
    <name>hadoop.proxyuser.$HTTPFSUSER.hosts</name>
    <value>httpfs-hostname</value>
</property>
<property>
    <name>hadoop.proxyuser.$HTTPFSUSER.groups</name>
    <value>*</value>
</property>
...
```
注意: $HTTPFSUSER 变量需要被替换为 Unix 系统用户

### 重启 Hadoop
为了激活代理用户配置, 需要重启 Hadoop

### 启动 / 停止 HttpFS
使用 HttpFS 的 脚本去起停 HttpFS, 例如
```
$ sbin/httpfs.sh start
```
注意: 没有任何参数运行脚本会列出所有可能的参数 (start, stop, run, etc); httpfs.sh 是 Tomcat 中 catalian.sh 的一个包装, 这个脚本设置了 HttpFS 服务器运行需要的一些环境变量和 Java 系统变量

### 测试 HttpFS
```
$ curl -sS 'http://<HTTPFSHOSTNAME>:14000/webhdfs/v1?op=gethomedirectory&user.name=hdfs'

{"Path":"\/user\/hdfs"}
```

### 内嵌 Tomcat 的配置
内嵌 Tomcat 的配置在 share/hadoop/httpfs/tomcat/conf 目录中; HttpFS 在 Tomcat 的 server.xml 中预配置了 HTTP 和 Admin 的端口, 分别是 14000 和 14001; Tomcat 的日志也被预配置到了 HttpFS 的 log/ 目录; 以下被使用的环境变量可以被修改 (在 etc/hadoop/httpfs-env.sh 脚本中设置)
- HTTPFS_HTTP_PORT
- HTTPFS_ADMIN_PORT
- HTTPFS_LOG

### HttpFS 配置项
|name|value|description|
|-|-|-|
|httpfs.buffer.size|4096|当有 HDFS 的数据流时, 读写请求使用的缓存大小|

### HttpFS 使用 Https (SSL)
为了配置 HttpFS 工作在 SSL 之上, 编辑配置目录中的 httpfs-env.sh 脚本, 将 HTTPFS_SSL_ENABLED 设置为 true; 另外, 以下两个属性也需要被定义 (以下是展示是默认属性):  
- HTTPFS_SSL_KEYSTORE_FILE = $HOME/.keystore
- HTTPFS_SSL_KEYSTORE)PASS = password
在 HttpFS 的 tomcat/conf 目录中, 将使用 ssl-server.xml 文件代替 server.xml 文件; 你需要为 HttpFS 服务器创建一个 SSL 认证, 以 httpfs 的 Unix 系统用户, 使用 Java keytool 命令创建 SSL 证书
```
$ keytool -genkey -ailas tomcat -keyalg RSA
```
你需要在交互窗口中回答一系列的问题, 这将会在 httpfs 用户的家目录下创建名为 .keystore 的 keystore 文件; 对 "keystore password" 输入的密码, 必须和配置目录中的 httpfs-env.sh 文件中设置的 HTTPFS_SSL_KEYSTORE_PASS 环境变量的值相同; 对于 "What is your first and last name?" 的答案必须是 HttpFS 服务器运行所在机器的 hostname; 启动 HttpFS, 将会使用 Https; 在使用 Hadoop 文件系统的 API 或者 Hadoop 文件系统的 shell 时, 需要使用 swedhdfs:// 空间; 如果使用自签名证书, 需要确保 JVM 可以找的到包含 SSL 证书公钥的 truststore  
注意: 一些老的 SSL 客户端可能使用不被 HttpFS 支持的弱密码, 推荐升级 SSL 客户端
