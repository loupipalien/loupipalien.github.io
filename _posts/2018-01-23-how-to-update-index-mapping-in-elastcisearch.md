### 在 elasticsearch 中如何修改 index 的 mapping

- 在 es 中默认是不允许修改索引已存在的映射 , 但支持在已存在的索引中添加新字段域的映射
- 如何想修改已存在的映射, 可以通过建立新的索引并设置想定义的映射, 再将原索引中的数据 reindex 到新的索引中
- 如果原索引只需要查询不需要写入, 可以为原索引建立一个别名, 在建立新索引并 reindex 后, 再将原索引的别名赋给新索引, 最后删掉原索引,这样则可以在不停服的情况下, 重建映射;

#### 创建临时索引
```
PUT bdp-map-database-tmp
{
	"settings": {
		"index": {
			"analysis": {
				"analyzer": {
					"ngram_analyzer": {
						"filter": "lowercase",
						"tokenizer": "ngram_analyzer"
					}
				},
				"tokenizer": {
					"ngram_analyzer": {
						"type": "nGram",
						"min_gram": "1",
						"max_gram": "10"
					}
				}
			}
		}
	},
	"mappings": {
		"database": {
			"properties": {
				"banReason": {
					"type": "keyword"
				},
				"businessTags": {
					"type": "keyword"
				},
				"database": {
					"type": "keyword"
				},
				"desc": {
					"type": "text"
				},
				"lastFullSyncTime": {
					"type": "date",
					"format": "yyyy-MM-dd HH:mm:ss||strict_date_optional_time||epoch_millis"
				},
				"layerTags": {
					"type": "keyword"
				},
				"location": {
					"type": "text"
				},
				"owners": {
					"type": "keyword"
				},
				"permissions": {
					"type": "keyword"
				},
				"remark": {
					"type": "text",
					"analyzer": "ngram_analyzer"
				},
				"topicTags": {
					"type": "keyword"
				}
			}
		}
	}
}
```
#### 将原索引数据迁移到临时索引
```
POST _reindex
{
	"source": {
		"index": "bdp-map-database",
		"type": "database"
	},
	"dest": {
		"index": "bdp-map-database-tmp"
	}
}
```
#### 删除原索引
```
DELETE /bdp-map-database
```
#### 创建原索引
```
PUT bdp-map-database
{
	"settings": {
		"index": {
			"analysis": {
				"analyzer": {
					"ngram_analyzer": {
						"filter": "lowercase",
						"tokenizer": "ngram_analyzer"
					}
				},
				"tokenizer": {
					"ngram_analyzer": {
						"type": "nGram",
						"min_gram": "1",
						"max_gram": "10"
					}
				}
			}
		}
	},
	"mappings": {
		"database": {
			"properties": {
				"banReason": {
					"type": "keyword"
				},
				"businessTags": {
					"type": "keyword"
				},
				"database": {
					"type": "keyword"
				},
				"desc": {
					"type": "text"
				},
				"lastFullSyncTime": {
					"type": "date",
					"format": "yyyy-MM-dd HH:mm:ss||strict_date_optional_time||epoch_millis"
				},
				"layerTags": {
					"type": "keyword"
				},
				"location": {
					"type": "text"
				},
				"owners": {
					"type": "keyword"
				},
				"permissions": {
					"type": "keyword"
				},
				"remark": {
					"type": "text",
					"analyzer": "ngram_analyzer"
				},
				"topicTags": {
					"type": "keyword"
				}
			}
		}
	}
}
```
#### 将临时索引数据迁移到原索引
```
POST _reindex
{
	"source": {
		"index": "bdp-map-database-tmp",
		"type": "database"
	},
	"dest": {
		"index": "bdp-map-database"
	}
}
```
#### 删除临时索引
```
DELETE /bdp-map-database-tmp
```
