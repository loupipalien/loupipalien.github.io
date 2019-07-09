---
layout: post
title: "Elasticsearch 的 Index Api"
date: "2018-02-09"
description: "Elasticsearch 的 Index Api"
tag: [elasticsearch]
---

### Index API
**重要:** 见 [Removal of mapping types](https://www.elastic.co/guide/en/elasticsearch/reference/current/removal-of-types.html)

索引 API 添加或更新一个 JSON 类型文档到一个指定索引中, 使之变成可搜索的; 以下示例插入 JSON 文档到 "twitter" 索引, 在 "\_doc" 类型下, id 为 1
```
PUT twitter/_doc/1
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```
以上索引操作的结果如下
```
{
    "_shards" : {
        "total" : 2,
        "failed" : 0,
        "successful" : 2
    },
    "_index" : "twitter",
    "_type" : "_doc",
    "_id" : "1",
    "_version" : 1,
    "_seq_no" : 0,
    "_primary_term" : 1,
    "result" : "created"
}
```
`_shards` 头提供了关于索引操作的副本处理信息
- `total`: 表示索引操作应该在多少个分片副本 (主分片和副本分片) 执行
- `successful`: 表示索引操作在多少个分片副本上操作成功
- `failed`: 包含副本相关错误的数组, 如果索引操作在一个副本分片上失败。

索引操作成功时 `successful` 的值至少是 1
**注意:**
当一个索引操作成功返回时副本分片可能没有全部启动 (默认的, 只有主分片是必须的, 但是行为可能被 [改变](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-index_.html#index-wait-for-active-shards)); 在这种情况下, `total` 的值等于基于 `number_of_replicas` 设置的总分片数, 并且 `successful` 的值等于已启动的分片数 (主分片加副本分片); 如果没有失败, `failed` 将会等于 0

#### 自动创建索引
如果之前还没有创建索引 (对于手动创建索引查看 [create index API](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html)), 索引操作将自动的创建一个索引, 并且也能为指定的类型自动创建一个动态类型映射, 如果此类型还没有被创建 (对于手动创建类型映射查看 [put mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-put-mapping.html))  
映射是非常灵活且模式自由的; 新的域和对象可以被自动的添加到指定类型的映射定义中; 更多的映射定义信息查看 [mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html) 章节  
自动创建索引可以通过早所有节点的配置文件中设置 `action.auto_create_index` 为 `false` 来禁止; 自动创建映射可以通过为每个索引设置 `index.mapper.dynamic` 为 `false` 作为索引设置来禁止  
自动创建索引包括基于黑白名单的模式匹配, 例如, 设置 `action.auto_create_index` 为 `+aaa*, -bbb*, +ccc*, -*` (+ 意味允许, - 意味禁止)

#### 版本化
每个索引文档都会有一个版本号; 关联的 `version` 号会作为 index API 请求响应的一部分返回; 当 `version` 参数被指定时, index API 对于 [乐观并发控制](http://en.wikipedia.org/wiki/Optimistic_concurrency_control) 可选择性支持; 这将会控制要执行文档的版本; 对于版本化的一个好的示例是执行一个事务性的读写;
指定初始读的文档的 `version`, 同时确保没有任何修改发生 (当读是为了更新, 推荐将 `preference` 设置为 `_primary`); 例如
```
PUT twitter/_doc/1?version=2
{
    "message" : "elasticsearch now has versioning support, double cool!"
}
```
注意: 版本化完全是实时的, 并且不会
