---
layout: post
title: "在 Hadoop 中何时开始 reduce  任务"
date: "2018-04-10"
description: "在 Hadoop 中何时开始 reduce  任务"
tag: [hadoop, mapreduce]
---
**环境:** CentOS-6.8-x86_64, hadoop-2.7.3

### reduce 何时开始
reduce 阶段有三个步骤: shuffle, sort, reduce  
shuffle 阶段是 reducer 从每个 mapper 中收集数据, 当 mappers 在生成数据时这有可能发生, 因为这仅仅在传输数据; 另一方面, sort 和 reduce 仅能发生在所有的 mappers 完成后; 你可以通过查看 reducer 的完成度来判定 mapreduce 作业正在做什么: 0 - 33% 意味着正在 shuffle, 34 - 66% 在 sort, 67 - 100% 在 reduce; 这就是为什么 reducer 有时似乎被卡在 33% --- 正是因为在等 mapper 完成  
reducer 基于 mappers 的一个完成度阈值开始 shuffle, 你可以改变参数来使得 recuder 更早或更晚的开始  

#### reduce 早开始的好处
为什么较早开始 reducers 是件好事, 因为它将从 mappers 到 reducer 的数据传输打散到更多的时间处理, 如果你的网络是瓶颈时这将是件好事

#### reduce 早开始的坏处
为什么较早开始 reducers 是件坏事, 因为当占用的 reduce 槽仅用于拷贝用户且等待 mapper 完成时, 另一个较晚开始的真正要用的 reduce 槽的作业反而不占用不到槽

#### 设置参数
你可以通过改变 `mapred-site.xml` 文件中 `mapred.reduce.slowstart.completed.maps` 的默认值来自定义 reducer 启动; 设置为 `1.00` 将会等待所有的 mappers 完成再启动 reducers; 设置为 `0.00` 将会立即启动 reducers; 设置为 `0.50` 将会在 mappers 完成一半时启动 reducers; 你可以在作业级别上修改 `mapred.reduce.slowstart.completed.maps`  参数, 在新版本的 Hadoop (2.4.1 以后) 中参数名也可以为 `mapreduce.job.reduce.slowstart.completedmaps`    
当系统有多个作业同时在运行时, 更偏好将 `mapred.reduce.slowstart.completed.maps` 参数设置为 `0.90` 以上, 当在仅拷贝数据而不做其他事时将不会占用 reducers 槽; 如果在同一时间仅有一个任务在运行, 设置为 `0.10` 大概是合适的

>**参考:**
[When do reduce tasks start in Hadoop?](https://stackoverflow.com/questions/11672676/when-do-reduce-tasks-start-in-hadoop)
