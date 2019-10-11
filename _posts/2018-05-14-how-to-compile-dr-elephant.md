---
layout: post
title: "如何编译 Dr.Elephant"
date: "2018-05-12"
description: "如何编译 Dr.Elephant"
tag: [dr-elephant]
---

**环境:** CentOS-7.5-x86_64, Hadoop-2.7.3, Spark-2.1.1

### Dr.Elephant
[Dr.Elephant](https://github.com/linkedin/dr-elephant) 是一个 Hadoop 和 Spark 的性能监控和调优工具, 它能够自动收集所有的度量信息并分析, 将其结果以简单的方式展现; 目标是通过使作业调优更容易以提高开发者的生产率和增加机器的效率; 它使用一系列的可插拔的, 可配置的, 基于规则的启发式算法分析 Hadoop 和 Spark 作业以洞察一个作业是如何执行的, 并且根据结果做出关于如何使调优使作业执行的更有效率的建议

#### 搭建环境要求
- CentOS
可以科学上网的或者配置了国内可靠的 SBT 源
- Java
play 要求 Java 1.8 以上
- Play
[Play](https://www.playframework.com) 版本出到了 2.6.13, 但从来没接触过, 图简单所以安装了发布的编译好的二进制包版本 2.2.6

#### 编译 Dr.Elephant
- 修改编译配置文件 ${DR_ELEPHANT_HOME}/compile.conf, 将 hadoop_version 和 spark_version 修改为自己所使用的版本
```
hadoop_version=2.7.3
spark_version=2.1.1
play_opts="-Dsbt.repository.config=app-conf/resolver.conf"
```
- 在 ${DR_ELEPHANT_HOME} 运行 compile.sh compile.conf 开始编译

#### 编译中遇到的报错
- npm ERR! argv "/usr/bin/node" "/usr/bin/npm" "install"
解决方法, 使用 yum 安装 nodejs, nodejs 中自带了 npm
- npm ERR! Failed at the node-sass@3.13.1 postinstall script 'node scripts/build.js'.
自动安装 node-sass 时报错, 于是使用命令 `npm install -g node-sass` 手动安装
- make: execvp: g++: Permission denied
安装 node-sass 时报错, 是因为没安装 gcc-c++, 使用 yum 安装后再次运行 `npm install -g node-sass` 即可
- Server access Error: connection refused url=http://repo.typesafe.com/typesafe/ivy-releases/org.scala-sbt/incremental-compiler/0.13.2/ivys/ivy.xml
CURL 请求这个路径返回 302, 使用浏览器访问这个链接发现跳转到了 dl.bintray.com, 在 ~/.sbt 目录下创建 repositories 文件写入以下配置
```
[repositories]
local
#暂时用不到 aliyun 的 maven 库
#aliyun-nexus: http://maven.aliyun.com/nexus/content/groups/public/  
typesafe: http://repo.typesafe.com/typesafe/ivy-releases/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext], bootOnly
bintray: http://dl.bintray.com/typesafe/ivy-releases/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext], bootOnly
sonatype-oss-releases
maven-central
sonatype-oss-snapshots
```
- impossible to get artifacts when data has not been loaded. IvyNode = com.fasterxml.jackson.core#jackson-databind;2.5.4
解决完 ivy.xml 文件下载不到的问题, 再次运行命令编译抛错... 查看 [issue#201](https://github.com/linkedin/dr-elephant/issues/201), 将 project/Dependencies.scala 中 jacksonVersion 修改为 2.5.4 仍然报错, mvn 仓库并没有 2.5.4 版本... Dependencies.scala 中 jackson-databind 声明的是 jacksonVersion (2.5.3), 为何编译时变成了 2.5.4 报错? 将 project/build.properties 中的 sbt.version 修改为 0.13.9 版本再编译报一下错
- java.lang.UnsupportedOperationException: Position.start on class scala.reflect.internal.util.OffsetPosition
查看到 [issue#1739](https://github.com/sbt/sbt/issues/1739#issuecomment-63909370) 可将 -Dsbt.parser.simple=true 参数传递给 play/SBT 命令行, 修改 ${DR_ELEPHANT_HOME}/compile.conf 配置文件在 play_opts 后添加这个参数再编译
- /app/dr-elephant/app/org/apache/spark/deploy/history/SparkDataCollection.scala:52: not enough arguments for constructor StorageStatusListener: (conf: org.apache.spark.SparkConf)org.apache.spark.storage.StorageStatusListener.
再遇到这个报错, 怀疑 scala 版本不对; 修改 ${DR_ELEPHANT_HOME}/project/Dependencies.scala 中的 spark-core_2.10 到 spark-core_2.11, 以及 ${DR_ELEPHANT_HOME}/build.sbt 中的 scalaVersion :="2.10.4" 到 scalaVersion :="2.11.8"
- sbt.ResolveException: unresolved dependency: com.typesafe.play#play-java-jdbc_2.11;2.2.6: not found
再编译报错找不到包, 查看 [maven](https://mvnrepository.com/artifact/com.typesafe.play/play-java-jdbc_2.11) 库 scala 2.11 无对应 play-2.2.6 版本的包...
- 放弃编译 spark-2.1.1 版本
Dr-Elephant 目前暂时不支持 Spark-2.x, [issue#327](https://github.com/linkedin/dr-elephant/issues/327) 中将 LinkedIn 内部实现了兼容 spark-2.x, 但是开源代码中仍未放出, SparkDataCollection.scala 中仍然使用的是 spark-1.6.x 接口依赖; 编译中如果测试用例报错, 将 ${DR_ELEPHANT_HOME}/compile.sh 中 `play_command \$OPTS clean test compile dist` 为 `play_command \$OPTS clean compile dist` 再编译即可

>**参考:**
[dr.elephant 环境搭建及使用详解](https://blog.csdn.net/xwc35047/article/details/73614657)  
[Failed at the node-sass@3.13.1 postinstall script](https://github.com/codecombat/codecombat/issues/4430)  
[Centos7 安装Nodejs](https://www.jianshu.com/p/7d3f3fa056e8)  
[make: execvp: g++: Permission denied](https://stackoverflow.com/questions/22316670/make-execvp-g-permission-denied)
[SBT国内源配置](https://www.jianshu.com/p/a867b2a7c3c8)
[Spark version compatibility and compilation error for Spark 2.0.1](https://github.com/linkedin/dr-elephant/issues/201)
