---
layout: post
title: "Kafka 的生产者配置 (0.10.2.0)"
date: "2017-11-28"
description: "Kafka 的生产者配置"
tag: [kafka]
---

**前提假设:**  版本 0.10.2.0

### 配置参数

|Name|Description|Type|Default|Valid Values|Importance|
|-|-|-|-|-|-|
|bootstrap.servers|用于建立到 kafka 集群初始连接的一组 host/port 对, 客户端将会使用所有的服务器, 而不会为了引导指定某一台服务器-这个列表仅用于发现全部服务器集合的初始主机; 列表应该形如 `host1:port1,host2:port2,...` 因为这些服务区仅用于初始连接去发现全部的集群成员 (是动态变化的), 这个列表并不需要包含全部的服务器集合 (不过可能需要多个, 以防服务器宕机)|list|-|-|高|
|key.serializer|用于键的实现了 `Serializer` 接口的序列化类|class|-|-|高|
|value.serializer|用于值的实现了 `Serializer` 接口的序列化类|class|-|-|高|
|acks|在认为一个请求完成之前生产者要求领导者接收的确认次数, 此控制记录发送的持久性; 以下是被允许的设置: `acks = 0`: 如果设置为 0, 生产者将完全不会等待服务器的任何确认, 记录将会被立刻添加到套接字缓存并且认为已发送, 在这种情况下服务器接收到记录是不做任何保证的, 并且 `retries` 配置将不生效 (因为客户端通常不会知道任何失败), 每个记录返回的偏移量也总会是 -1; `acks = 1`: 这意味着领导者将记录写到自己本地日志就返回响应, 而不会等待所有跟随者全部确认, 在这种情况下如果在领导者确认记录之后立即失败, 但又在跟随者复制记录之前那么此记录就会丢失; `acks = all`: 这意味着领导者就会等待此记录的全部同步复制确认集合, 只要有一个同步复制存活就能保证此记录将不会丢失, 这是最有力的保证, 此设置值等同于 `acks = -1`|string|1|[all,-1,0,1]|高|
|buffer.memory|生产者可用于缓存等待发送给服务器的记录的总内存字节, 如果记录发送的速度快于发送到服务器的速度, 则生产者在阻塞 `max.block.ms` 之后抛出一个异常; 设置值仅是大致的对应生产者将使用的内存, 这不是一个硬性的界限因为不是生产者用的所有内存都被用于缓存, 一些额外的内存将会被用于压缩 (如果开启了压缩) 为了维持飞速的请求 |long|33554432|[0,...]|高|
|compression.type|对于生产者产生的所有数据的压缩方式; 默认值是 none (无压缩), 有效值是 `none`, `gzip`, `snappy`, `lz4`; 压缩是对于批次的数据, 批次的效率也会影响压缩率 (更多的批次意味着着更好的压缩)|string|none|-|高|
|retries|设置值大于 0 意味着有着潜在传输错误的记录被发送失败时会重发; 注意如果客户端重发记录收到了错误那么重试时没有区别的, 允许重试而没有设置 `max.in.flight.requests.per.connection` 为 1 将会有潜在的改变记录顺序的问题, 因为如果两个批次被发送到同一个分区, 当第一个失败然后重试但是第二个成功, 则第二批次中的记录可能会先出现|int|0|[0,...,2147485647]|高|
|ssl.key.password|在 key store 文件中私匙地密码, 对于客户端时可选项|password|null|-|高|
|ssl.keystore.location|key store 文件的位置, 对于客户端是可选项并且此项可用于客户端的双向认证|string|null|-|高|
|ssl.keystore.password|key store 文件的存储密码, 对于客户端是可选的并且只有在 `ssl.keystore.location` 被配置时才需要|password|null|-|高|
|ssl.truststore.location|trust store 文件的位置|string|null|-|高|
|ssl.truststore.password|trust store 文件的存储密码|password|null|-|高|
