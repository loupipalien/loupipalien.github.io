---
layout: post
title: "Kafka 的消费者配置 (0.10.2.0)"
date: "2017-11-28"
description: "Kafka 的消费者配置"
tag: [kafka]
---

**前提假设:** 版本 0.10.2.0

|Name|Description|Type|Default|Valid Values|Importance|
|-|-|-|-|-|-|
|bootstrap.servers|用于建立到 kafka 集群初始连接的一组 host/port 对, 客户端将会使用所有的服务器, 而不会为了引导指定某一台服务器-这个列表仅用于发现全部服务器集合的初始主机; 列表应该形如 `host1:port1,host2:port2,...` 因为这些服务区仅用于初始连接去发现全部的集群成员 (是动态变化的), 这个列表并不需要包含全部的服务器集合 (不过可能需要多个, 以防服务器宕机)|list|-|-|高|
|key.deserializer|用于键的实现了 `Deserializer` 接口的反序列化类|class|-|-|高|
|value.deserializer|用于值的实现了 `Deserializer` 接口的反序列化类|class|-|-|高|
|fetch.min.bytes|服务器对于一次获取请求的最小数据量, 如果可获取的数据量不足, 将会等待积累更多的数据量后响应请求; 默认值是 1 个字节, 这意味着只要有一个字节可用获取请求就会被响应, 或者获取请求在数据到来的等待中超时; 将此设置为比 1 大的值会引起服务器等待更大的数据量积累, 这能提高服务器的吞吐率而增加额外的延迟|int|1|[0,...]|高|
|group.id|用于区分此消费者属于那以消费组的唯一字符串; 如果消费者通过 `subscribe(topic)` 使用了组管理功能或者基于 kafka 的品阿姨管理策略, 则需要此属性|string|""|-|高|
|heartbeat.interval.ms|当使用 kafka 的组管理功能时心跳到消费者协调器的期望时间, 心跳用于确认此消费者会话仍处于活跃, 并且在当有新的消费者加入或离开组时重新负载均衡; 这个值的设置必须小于 `session.timeout.ms`, 并且通常设置不高于此值的 1/3, 甚至可以调整到更低以控制正常负载均衡的期望时间 |int|3000|-|高|
|max.partition.fetch.bytes|服务器返回的每个分区的最大数据量; 如果在第一个非空分区获取的第一条消息大于此限制, 此消息将会被返回以确保消费者能够继续; 被 broker 可接收的最大消息的大小通过 `message.max.bytes` (broker 配置) 或 `max.message.bytes` (topic 配置); 参考消费者请求限制大小设置 `fetch.max.bytes`|int|1048576|[0,...]|高|
|session.timeout.ms|当使用 kafka 的组管理时此值用于探测消费者是否失败; 此消费者周期性地给 broker 发送心跳表示活跃, 如果 broker 在会话超时过期之前没有收到心跳, 接下来 broker 将会从组中移除此消费者并初始化一次负载均衡; 注意此值必须在 broker 配置项 `group.min.session.timeout.ms` 和 `group.max.session.timeout,ms` 配置值所允许地范围内|int|10000|-|高|
|ssl.key.password|在 key store 文件中私匙地密码, 对于客户端时可选项|password|null|-|高|
|ssl.keystore.location|key store 文件的位置, 对于客户端是可选项并且此项可用于客户端的双向认证|string|null|-|高|
|ssl.keystore.password|key store 文件的存储密码, 对于客户端是可选的并且只有在 ssl.keystore.location 被配置时才需要|password|null|-|高|
|ssl.truststore.location|trust store 文件的位置|string|null|-|高|
