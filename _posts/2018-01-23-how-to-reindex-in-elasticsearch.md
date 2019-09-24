---
layout: post
title: "Elasticsearch 如何 reindex"
date: "2018-01-23"
description: "Elasticsearch 如何 reindex"
tag: [elasticsearch]
---

### Reindex API
> **重要**
reindex 并不尝试建立目标索引, 它并不会拷贝源索引的设置; 所以在运行 `_reindex` 操作之前应该先建立目标索引, 包括建立映射, 分片数, 副本数, 等等

`_reindex` 最基本的形式是将文档从一个索引中拷贝到另一个索引中, 以下语句会将 `twitter` 索引中的文档拷贝到 `new_twitter` 索引中
```
POST_reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter"
  }
}
```
返回信息大致如下
```
{
  "took" : 147,
  "timed_out": false,
  "created": 120,
  "updated": 0,
  "deleted": 0,
  "batches": 1,
  "version_conflicts": 0,
  "noops": 0,
  "retries": {
    "bulk": 0,
    "search": 0
  },
  "throttled_millis": 0,
  "requests_per_second": -1.0,
  "throttled_until_millis": 0,
  "total": 120,
  "failures" : [ ]
}
```
类似于 [_update_by_query](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-update-by-query.html),  `_reindex` 获取源索引的快照并且目标必须是不同的索引, 所以不太可能有版本冲突; `dest` 元素可以像索引 API 配置乐观并发控制; 不配置 `version_type` (像上面一样) 或者将它设置为 `internal`, 这会让 elasticsearch 只一味的将文档转存到目标索引, 如果有任何相同的类型或 ID 会被覆写
```
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "version_type": "internal"
  }
}
```
设置 `version_type` 为 `external` 会使 elasticsearch 保留源索引文档的版本号, 目标索引中不存在的文档会被创建, 目标索引中的文档比源索引中的文档版本号更老, 文档将会被更新 (源索引文档比目标索引文档版本号更老是不是会冲突?)
```
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "version_type": "external"
  }
}
```
将 `op_type` 设置为 `create` 会使 `_reindex` 仅创建在目标索引中不存在的文档, 所有已已存在的文档都会造成一个版本冲突
```
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "op_type": "create"
  }
}
```
默认的, 如果有版本冲突会放弃 `_reindex` 进程, 但是你可以通过在请求体中设置 `"conflicts": "proceed"` 来统计, 从而继续 `_reindex` 进程
```
POST _reindex
{
  "conflicts": "proceed",
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "op_type": "create"
  }
}
```
你可以通过添加类型到 `source` 中 或者添加一个查询限制要复制的文档, 以下语句将只拷贝 `tweet` 类型中由 `kimchy` 创建的文档到 `new_twitter` 索引中
```
POST _reindex
{
  "source": {
    "index": "twitter",
    "type": "tweet",
    "query": {
      "term": {
        "user": "kimchy"
      }
    }
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

>**参考:**
[Reindex API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html)
