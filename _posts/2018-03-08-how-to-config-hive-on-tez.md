---
layout: post
title: "如何配置 Hive On Tez"
date: "2018-03-08"
description: "如何配置 Hive On Tez"
tag: [hive, tez]
---

**环境:** CentOS-6.8-x86_64, hive-2.1.1, tez-0.9.0

### 为 Tez 部署底层应用
对于 Tez-0.9.0 以及更高版本, Tez 需要 Apache Hadoop 版本为 2.7.0 或更高
#### 安装 Apache Hadoop 2.7.0 或更高版本
- 需要
```
$ hadoop version
```
#### 使用 `mvn clean package -DskipTests=true -Dmaven.javadoc.skip=true` 构建 tez
- 假定你已经安装了 JDK8 或更高版本和 Maven3 或更高版本
- Tez 要求安装 Protocol Buffers 2.5.0, 包括 protoc-compiler
  - 可以从 [protobuf releases](https://github.com/google/protobuf/tags) 获得
  - Mac...
  - rpm-based...
- 如果想要执行单元测试, 移除上述命令中的 skipTests 参数即可
-  如果使用 Eclipse IDE ...

####  拷贝 tez 相关 tarball 到 HDFS, 并且配置 tez-site.xml
1. 在 tez-dist/target/tez-x.y.z-SNAPSHOT.tar.gz 中可以找到一个包含 tez 和 hadoop 函数库的 tarball
  - 假定这个 tez 的 jar 放置在 HDFS 的 /app/ 目录下, 命令行如下
  ```
  hadoop fs -mkdir /apps/tez-x.y.z-SNAPSHOT
  hadoop fs -copyFromLocal tez-dist/target/tez-x.y.z-SNAPSHOT.tar.gz /apps/tez-x.y.z-SNAPSHOT/
  ```
2. tez-site.xml 配置
  - 设置 tez.lib.uris 指向 tar.gz 上传的 HDFS 路径, 假定遵循上一步提及设定, 设置 tez.lib.uris 为 `${fs.defaultFS}/apps/tez-x.y.z-SNAPSHOT/tez-x.y.z-SNAPSHOT.tar.gz`
  - 确保 tez.use.cluster.hadoop-libs 不被设置在 tez-site.xml 中, 如果设置了那么值应该为 false
3. 注意当提交 Tez 作业到集群时, tarball 的版本应该匹配客户端 jar 的版本, 更多的版本兼容和探测失配请参考 [Version Compatibility Guide](https://cwiki.apache.org/confluence/display/TEZ/Version+Compatibility)
4. 可选: 如果想要已有的 MapReduce 运行在 Tez 上, 修改 mapred-site.xml 配置文件, 将 "mapreduce.framework.name" 属性从默认的 "yarn" 改为 "yarn-tez"
5. 配置客户端节点的 hadoop classpath 包含 tez-libraries 到 hadoop classpath 中
  - 从第二步中生成的本地文件夹中找到生成的 tez minimal tarball 解压 (既定 TEZ_JARS 是为下一步解压的路径)
    ```
    tar -xvzf tez-dist/target/tez-x.y.z-minimal.tar.gz -C $TEZ_JARS
    ```
  - 在 tez-site.xml 中设置 TEZ_CONF_DIR
  - 添加 ${TEZ_CONF_DIR}, ${TEZ_JARS}/* 和 ${TEZ_JARS}/lib/* 到应用 classpath; 使用以下语句通过标准 Hadoop 工具链设置应用 classpath
    ```
    export HADOOP_CLASSPATH=${TEZ_CONF_DIR}:${TEZ_JARS}/*:${TEZ_JARS}/lib/*
    ```
  - 当设置 classpath 路径中有包含 jars 的目录, 注意 "\*" 是非常重要的
6. 在 tez-examples.jar 中有使用 MRR 的基础示例, 参照源码的 OrderedWordCount.java, 运行示例:
...
7. 可以使用类似以下示例提交 MR 作业
...

#### 配置 tez.lib.uris 的多种方式
`tez.lib.uris` 配置属性支持逗号分隔值列表; 支持的值类型: 文件路径, 目录路径, 压缩文档 (tarball, zip, etc)  
对于简单文件和目录, Tez 将会添加这些文件和目录中顶层中的项到 Tez 运行时的工作目录, 会被自动的包含到 classpath 中; 对于文档, 已知的常见压缩文件类型后缀, 例如 "tgz, tar.gz, zip" 等, 也会被解压到容器工作目录; 然而, Tez 框架不知道给予的文档结构, 用户则需要配置 `tez.lib.uris.classpath` 确保文档的内部目录结构添加到 classpath; classpath 的值应该使用以 "./" 开头的相对路径

#### Hadoop 安装的依赖安装部署说明
以上的安装说明使用包含有预编译的 Hadoop 函数库 Tez 的包, 也是推荐的安装方式; 使用有所有依赖的 tarball 是一种比较好的方式, 可以确保在集群滚动升级时运行已存在的任务  
但 `tez.lib.uris` 配置项可以扩展使用方式, 这里有两种可选的方式安装来支持框架
- Mode A: 使用在 HDFS 上的 tez tarball 配合集群上可用的 Hadoop 函数库
- Mode B: 使用有 Hadoop tarball 的 tez tarball

以上模式都要求编译无 Hadoop 函数库的 tez, 在  tez-dist/target/tez-x.y.z-minimal.tar.gz 中可以找到

#### Mode A: 通过利用 "yarn.application.classpath", Tez tarball 使用已存在集群上的 Hadoop 函数库
这种模式对于使用滚动升级的集群不推荐使用; 另外, 用户需要确认使用的 tez 版本和集群运行的 Hadoop 版本兼容; 此模式前三步如以上描述所写, 后续步骤应该将 tez-dist/target/tez-x.y.z.tar.gz 替换为 tez-dist/target/tez-x.y.z-minimal.tar.gz
- 没有 Hadoop 依赖编译的 tez 可在 tez-dist/target/tez-x.y.z-minimal.tar.gz 获得, 假定 tez jars 放在 HDFS 的 /apps 目录下, 命令大致如下
  ```
  "hadoop fs -mkdir /apps/tez-x.y.z"
  "hadoop fs -copyFromLocal tez-dist/target/tez-x.y.z-minimal.tar.gz /apps/tez-x.y.z"
  ```
- tez-site.xml 配置
  - 设置 tez.lib.uris 指向包含 tez jars 的 HDFS 路径, 假定遵循以上提及的目录, 设置 tez.lib.uris 为 `${fs.defaultFS}/apps/tez-x.y.z/tez-x.y.z-minimal.tar.gz`
  - 设置 tez.use.cluster.hadoop-libs 为 true

#### Mode B: 有 Hadoop tarball 的 Tez tarball
...

### 安装中遇到的报错
在 Windows 下使用 Idea 编译困难重重, 遇到了以下报错
- windows 缺少 protoc 环境, 根据参考文章方法解决 (应该不用重启就行)
- ember 从 github 上拉取失败, 根据官方参考文章中方法解决, 将 https:// 换为 git:// 再编译即可
- (Bower install) on project tez-ui 失败, 暂时解决不掉...弃坑, 从 apache 下载已编译好的包使用了算
- 因为集群稳定不常升级, 使用 Mode A 安装; 安装后使用 Cli 跑 Sql 报错
虚拟内存超出, 根据参考文章中的设置不检查虚拟内存或者调大与物理内存比例解决

>**参考:**
[Install/Deploy Instructions for Tez](http://tez.apache.org/install.html)  
[Build errors and solutions](https://cwiki.apache.org/confluence/display/TEZ/Build+errors+and+solutions)  
[windows 环境下的 protoc 安装](http://blog.csdn.net/pzw_0612/article/details/51798402)
[轻松配置Hive On Tez](http://lxw1234.com/archives/2017/06/860.htm)  
[hadoop任务运行错误之container is running beyond virtual memory limits](https://mowblog.com/hadoop%E4%BB%BB%E5%8A%A1%E8%BF%90%E8%A1%8C%E9%94%99%E8%AF%AF%E4%B9%8Bcontainer-is-running-beyond-virtual-memory-limits/)
