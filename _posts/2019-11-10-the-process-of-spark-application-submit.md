---
layout: post
title: "Spark 应用的提交流程"
date: "2019-11-10"
description: "Spark 应用的提交流程"
tag: [spark]
---

### Spark 应用的提交流程
Spark 提交应用到 Yarn 上运行
```
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master yarn \
  --deploy-mode cluster \  # can be client for client mode
  --executor-memory 20G \
  --num-executors 50 \
  /path/to/examples.jar \
  1000
```

#### 脚本解析过程
- `spark-submit` 脚本中主要中添加了 `org.apache.spark.deploy.SparkSubmit` 参数, 与原参数一起调用 `spark-class` 脚本
- `spark-class` 脚本先将所有参数调用 `org.apache.spark.launcher.Main` 类进行处理, 解析所有的参数后构造为后续执行的命令
- `spark-class` 执行上一步构造出命令: `org.apache.spark.deploy.SparkSubmit ...`

#### SparkSubmit 的执行过程
- 根据提交的参数判定是 `submit, kill, request_status` (这里叙述 submit 过程)
- submit 方法主要有两步
  - 基于集群管理器 (--master) 和部署模式 (--deploy-mode) 为子类 (--class) 准备正确的 `classpath, system properties, application arguments`
  - 使用准备好的发布环境调用子类 (--class) 的 main 函数

#### 在 Yarn 上的运行过程
##### client 模式
client 模式中, Driver 在客户端本地运行, 这种模式可以使得 Spark Application 和客户端进行交互
- SparkYarnClient 向 Yarn 的 RM 申请启动 AM, 同时在 SparkContext 初始化中创精 DAGScheduler 和 TaskScheduler; 由于是 client 模式, 后端实现会选择 YarnClientClusterScheduler 和 YarnClientSchedulerBackend
- RM 接收请求后, 在集群中选择一个 NM, 为 Spark 应用分配第一个 Container, 在这个 Container 中启动应用的 AM; 与 cluster 模式不同的是, AM 上不运行 SparkContext, 只与 SparkContext 进行协商资源的协商和分配
- Client 中的 SparkContext 初始化完毕后, 与 AM 建立通信向 RM 注册, AM 根据应用信息向 RM 申请资源
- 一旦 AM 申请到资源后, 便与对应的 NM 通信, 要求 NM 在获得的 Container 中启动 CoarseGrainedExecutorBackend, 启动后会向 Client 中的 SparkContext 注册并申请 Task
- Client 中的 SparkContext 分配 Task 给 CoarseGrainedExecutorBackend 执行, CoarseGrainedExecutorBackend 运行 Task 并向 Driver 汇报运行的状态和进度, 以便知晓各个任务的状态, 从而在 Task 失败时重试
- 应用完成后, Client 的 SparkContext 向  RM 申请注销并关闭
![client](https://img-blog.csdn.net/20150812152811799)

#### cluster 模式
与 client 不同的是, SparkContext 会在 AM 上启动并运行
![cluster](https://img-blog.csdn.net/20150812152831281)

>**参考:**
- [Submitting Applications](https://spark.apache.org/docs/2.2.0/submitting-applications.html)
- [Spark应用程序提交流程](https://blog.csdn.net/lovehuangjiaju/article/details/49123975)
- [Spark运行架构](https://blog.csdn.net/yirenboy/article/details/47441465)
- [Spark 运行架构基本概念](https://blog.csdn.net/book_mmicky/article/details/25714419)
