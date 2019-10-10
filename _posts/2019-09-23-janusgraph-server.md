---
layout: post
title: "Guava 用户指南之字符串"
date: "2019-09-23"
description: "Guava 用户指南之字符串"
tag: [janusgraph]
---

### JanusGraph Server
JanusGraph 使用 [Gremlin Server](https://tinkerpop.apache.org/docs/3.4.1/reference/#gremlin-server) 引擎作为服务器组件来处理和响应客户单请求; 当打包在 JanusGraph 中时, Gremlin Server 被称为 JanusGraph Server   
为了使用 JanusGraph Server 必须手动启动它; JanusGraph Server 提供了一种对托管在其中的一个或多个 JanusGraph 实例远程执行 Gremlin 遍历的方法; 本节将叙述如何使用 WebSocket 配置, 以及如何配置 JanusGraph Server 处理 HTTP 端点交互; 更多关于用不同语言连接一个 JanusGraph Server 见 [Connecting to JanusGraph](https://docs.janusgraph.org/connecting/)

#### 快速开始
##### 使用预发包的发行版
JanusGraph 版本已预先提供 Cassandra 和 Elasticsearch 的样本配置, 可以开箱即用地运行 JanusGraph Server, 以允许用户快速开始使用 JanusGraph Server; 这个配置默认客户端应用可以通过自定义子协议 WebSocket 连接 JanusGraph; 有许多使用不同语言开发的客户端可以帮助支持该子协议, 使用 WebSocket 接口最熟悉的客户端是 Gremlin Console; 快速入门包并非表示生产安装, 而是提供了一种使用 JanusGraph Server 进行开发, 运行测试以及查看组件如何连接在一起的方法; 使用此默认配置
- 从 [Release page](https://github.com/JanusGraph/janusgraph/releases) 下载当前版本的 `janusgraph-$VERSION.zip` 文件
- 解压文件并进入 `janusgraph-$VERSION` 目录
- 运行 `bin/janusgraph.sh start`, 这一步将会启动 Gremlin Server 以及 Fork 到单独进程的 Cassandra/ES; 注意, 由于 Elasticsearch 的安全原因, `janusgraph.sh` 必须使用非 root 用户运行
```
$ bin/janusgraph.sh start
Forking Cassandra...
Running `nodetool statusthrift`.. OK (returned exit status 0 and printed string "running").
Forking Elasticsearch...
Connecting to Elasticsearch (127.0.0.1:9300)... OK (connected to 127.0.0.1:9300).
Forking Gremlin-Server...
Connecting to Gremlin-Server (127.0.0.1:8182)... OK (connected to 127.0.0.1:8182).
Run gremlin.sh to connect.
```
##### 连接到 Gremlin Server
在运行 `janusgraph.sh` 后, Gremlin Server 将会准备好监听 WebSocket 连接; 最容易测试连接的方式是 Gremlin Console  
使用 `bin.gremlin.sh` 启动 Gremlin Console, 并且使用 `:remote` 和 `:>` 命令将 Gremlin 发送给 Gremlin Server
```
$  bin/gremlin.sh
         \,,,/
         (o o)
-----oOOo-(3)-oOOo-----
plugin activated: tinkerpop.server
plugin activated: tinkerpop.hadoop
plugin activated: tinkerpop.utilities
plugin activated: janusgraph.imports
plugin activated: tinkerpop.tinkergraph
gremlin> :remote connect tinkerpop.server conf/remote.yaml
==>Connected - localhost/127.0.0.1:8182
gremlin> :> graph.addVertex("name", "stephen")
==>v[256]
gremlin> :> g.V().values('name')
==>stephen
```
`:remote` 命令告知控制台使用 `conf/remote.yaml` 文件配置一个远程连接到 Gremlin Server; 这个文件指向运行在 `localhost` 上的一个 Gremlin Server 实例; `:>` 是提交命令, 该命令将该行上的 Gremlin 发送到当前活动的远程连接; 远程连接默认是无会话的, 这意味着在控制台发送的每一行都会解释成单个请求; 多语句可以使用分号作为分隔符作为单行发送; 可选的, 在建立连接时, 你可以通过指定 `session` 建立一个带会话的控制台; 会话态的控制台允许你在几行输入中重用变量
```
gremlin> :remote connect tinkerpop.server conf/remote.yaml
==>Configured localhost/127.0.0.1:8182
gremlin> graph
==>standardjanusgraph[cql:[127.0.0.1]]
gremlin> g
==>graphtraversalsource[standardjanusgraph[cql:[127.0.0.1]], standard]
gremlin> g.V()
gremlin> user = "Chris"
==>Chris
gremlin> graph.addVertex("name", user)
No such property: user for class: Script21
Type ':help' or ':h' for help.
Display stack trace? [yN]
gremlin> :remote connect tinkerpop.server conf/remote.yaml session
==>Configured localhost/127.0.0.1:8182-[9acf239e-a3ed-4301-b33f-55c911e04052]
gremlin> g.V()
gremlin> user = "Chris"
==>Chris
gremlin> user
==>Chris
gremlin> graph.addVertex("name", user)
==>v[4344]
gremlin> g.V().values('name')
==>Chris
```
#### 使用预发包的发行版后清理
如果你想重新开始并且移除数据库和日志, 你可以使用 `janusgraph.sh` 的 clean 命令; 在运行清理操作前需先停止服务器
```
$ cd /Path/to/janusgraph/janusgraph-0.2.0-hadoop2/
$ ./bin/janusgraph.sh stop
Killing Gremlin-Server (pid 91505)...
Killing Elasticsearch (pid 91402)...
Killing Cassandra (pid 91219)...
$ ./bin/janusgraph.sh clean
Are you sure you want to delete all stored data and logs? [y/N] y
Deleted data in /Path/to/janusgraph/janusgraph-0.2.0-hadoop2/db
Deleted logs in /Path/to/janusgraph/janusgraph-0.2.0-hadoop2/log
```

#### JanusGraph Server 作为一个 WebSocket 端
在 [Gettign Started](https://docs.janusgraph.org/basics/server/#getting-started) 中叙述的默认配置就是一个 WebSocket 配置; 如果你想要更改默认配置以使其与自己的 Cassandra 或 HBase 环境一起使用, 而不是使用快速启动环境, 请按照以下步骤操作
##### 为 WebSocket 配置 JanusGraph Server
- 首先测试 JanusGraph 数据库的本地连接; 无论使用 Gremlin Console 来测试连接, 还是从程序代码进行连接, 都适用此步骤; 针对你的环境, 在 `./conf` 目录中的对相应的属性文件做适当的修改; 例如, 编辑 `./conf/janusgraph-hbase.proeprties` 并且确保 `storage.backend, storage.hostname, storage.habse.table` 参数指定正确; 对于各种存储后端的 JanusGraph 配置见 [Storage Backends](https://docs.janusgraph.org/storage-backend/); 确保数据文件包含以下配置
```
gremlin.graph=org.janusgraph.core.JanusGraphFactory
```
- 一旦本地配置以测试并你有一个能生效的配置文件, 则将这个配置文件从 `./conf` 目录拷贝到 `./conf/gremlin-server` 目录
```
cp conf/janusgraph-hbase.properties conf/gremlin-server/socket-janusgraph-hbase-server.properties
```
- 将 `/conf/gremlin-server/gremlin-server.yaml` 拷贝成名为 `socket-gremlin-server.yaml` 的文件; 以防你需要参考原始版的文件
```
cp conf/gremlin-server/gremlin-server.yaml conf/gremlin-server/socket-gremlin-server.yaml
```
- 编辑 `socket-gremlin-server.yaml` 文件并做以下更新
  - 如果你打算从其他 IP 而不是本地连接到 JanusGraph, 更新 host 为 IP 地址
```
host: 10.10.10.100
```
  - 更新 graphs 章节指定你的新属性文件, 这样 JanusGraph Server 可以发现并连接到你的 JanusGraph 实例
```
graphs: { graph:
    conf/gremlin-server/socket-janusgraph-hbase-server.properties
}
```
- 启动 JanusGraph Server, 指定你配置的那个 yaml 文件
```
bin/gremlin-server.sh ./conf/gremlin-server/socket-gremlin-server.yaml
```
- JanusGraph Server 现在运行在 WebSocket 模式中, 可以参照 [Connecting to Gremlin Server](https://docs.janusgraph.org/basics/server/#first-example-connecting-gremlin-server) 的说明测试

>不要使用 `bin/janusgraph.sh`, 那将会使用默认的配置启动, 并且启动一个单独的 Cassandra/Elasticsearch 环境

#### JanusGraph Server 作为一个 HTTP 端
在 [Gettign Started](https://docs.janusgraph.org/basics/server/#getting-started) 中叙述的默认配置就是一个 WebSocket 配置; 如果你想修改默认配置, 使你的 JanusGraph 数据库使用 JanusGraph Server 作为 HTTP 端, 可以按照以下步骤
- 首先测试 JanusGraph 数据库的本地连接;  无论使用 Gremlin Console 来测试连接, 还是从程序代码进行连接, 都适用此步骤; 针对你的环境, 在 `./conf` 目录中的对相应的属性文件做适当的修改; 例如, 编辑 `./conf/janusgraph-hbase.proeprties` 并且确保 `storage.backend, storage.hostname, storage.habse.table` 参数指定正确; 对于各种存储后端的 JanusGraph 配置见 [Storage Backends](https://docs.janusgraph.org/storage-backend/); 确保数据文件包含以下配置
```
gremlin.graph=org.janusgraph.core.JanusGraphFactory
```
- 一旦本地配置以测试并你有一个能生效的配置文件, 则将这个配置文件从 `./conf` 目录拷贝到 `./conf/gremlin-server` 目录
```
cp conf/janusgraph-hbase.properties conf/gremlin-server/http-janusgraph-hbase-server.properties
```
- 将 `/conf/gremlin-server/gremlin-server.yaml` 拷贝成名为 `http-gremlin-server.yaml` 的文件; 以防你需要参考原始版的文件
```
cp conf/gremlin-server/gremlin-server.yaml conf/gremlin-server/http-gremlin-server.yaml
```
- 编辑 `http-gremlin-server.yaml` 文件并做以下更新
  - 如果你打算从其他 IP 而不是本地连接到 JanusGraph, 更新 host 为 IP 地址
```
host: 10.10.10.100
```
  - 更新 channelizer 设置指定为 HttpChannelizer
```
channelizer: org.apache.tinkerpop.gremlin.server.channel.HttpChannelizer
```
  - 更新 graphs 章节指定你的新属性文件, 这样 JanusGraph Server 可以发现并连接到你的 JanusGraph 实例
```
graphs: { graph:
    conf/gremlin-server/http-janusgraph-hbase-server.properties
}
```
- 启动 JanusGraph Server, 指定你配置的那个 yaml 文件
```
bin/gremlin-server.sh ./conf/gremlin-server/http-gremlin-server.yaml
```
- JanusGraph Server 现在运行在 HTTP 模式中, 并且是可测试的; `curl` 可以用来验证服务是否工作
```
curl -XPOST -Hcontent-type:application/json -d *{"gremlin":"g.V().count()"}* [IP for JanusGraph server host](http://):8182
```
#### JanusGraph Server 作为一个 WebSocket 和 HTTP 端
JanusGraph 0.2.0 开始, 你可以配置你的 `gremlin-server.yaml` 在一个端口上同时接收 WebSocket 和 HTTP 连接; 这个可以通过修改之前实例中的 channelizer 实现
```
channelizer: org.apache.tinkerpop.gremlin.server.channel.WsAndHttpChannelizer
```

#### 高级 JanusGraph Server 配置
##### 通过 HTTP 认证
TODO
##### 通过 WebSocket 认证
TODO
##### ##### 通过 HTTP 和 WebSocket 认证
TODO

##### TinkerPop Gremlin Server 和 JanusGraph 一起使用
因为 JanusGraph Server 是一个打包了 JanusGraph 配置文件的 TinkerPop Gremlin Server; 可以单独下载版本兼容的 TinkerPop Gremlin Server 和 JanusGraph 一起使用; 通过下载适当的 Gremlin Server 版本开始使用, 该版本需要与使用中的 JanusGraph 支持的版本相匹配
>除非特别指出, 否则本节中对文件路径的所有引用均指 TinkerPop 分发的 Gremlin Server 下的路径, 而不是 JanusGraph 分发的 JanusGraph Server 下的路径

TODO

#### 拓展 JanusGraph Server
通过实现 Gremlin Server 提供的接口并利用 JanusGraph 进行扩展，可以用其他通信方式扩展 Gremlin Server; 更多详细信息在 TinkerPop 文档中查看
