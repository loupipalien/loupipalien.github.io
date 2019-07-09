## hive metastore design

### 数据表
- AUX_TABLE [x]
- BUCKETING_COLS [x]
- CDS 

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|CD_ID|BIGINT|20|-|-|-|

- COLUMNS_V2

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|CD_ID|BIGINT|20|-|-|外键(CDS.CD_ID)|
|COMMENT|VARCHAR|256|[x]|-|注释|
|COLUMN_NAME|VARCHAR|767|-|-|字段名|
|TYPE_NAME|VARCHAR|4000|[x]|-|字段类型|
|INTEGER_IDX|INT|11|-|-|字段索引顺序|

- COMPACTION_QUEUE
- COMPLETED_COMPACTIONS
- COMPLETED_TXN_COMPONENTS
- DATABASE_PARAMS
- DBS

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|DB_ID|BIGINT|20|-|-|-|
|DESC|VARCHAR|4000|[x]|-|描述|
|DB_LOCATION_URI|VARCHAR|4000|-|-|在HDFS的路径|
|NAME|VARCHAR|128|[x]|-|数据表名|
|OWNER_NAME|VARCHAR|128|[x]|-|属主|
|OWNER_TYPE|VARCHAR|10|[x]|-|属主类型|

- DB_PRIVS
- DELEGATION_TOKENS
- FUNCS
- FUNC_RU
- GLOBAL_PRIVS

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|USER_GRANT_ID|BIGINT|20|-|-|-|
|CREATE_TIME|INT|11|[x]|-|描述|
|GRANT_OPTION|SMALLINT|6|-|-|数据库在HDFS的路径连接|
|GRANTOR|VARCHAR|128|[x]|-|授权人|
|GRANTOR_TYPE|VARCHAR|128|[x]|-|授权人类型|
|PRINCIPAL_NAME|VARCHAR|128|[x]|-|当事人|
|PRINCIPAL_TYPE|VARCHAR|128|[x]|-|当事人类型|
|USER_PRIV|VARCHAR|128|[x]|-|用户权限|

- HIVE_LOCKS
- IDXS
- INDEX_PARAMS
- KEY_CONSTRAINTS
- MASTER_KEYS
- NEXT_COMPACTION_QUEUE_ID

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|NCQ_NEXT|BIGINT|20|-|-|-|

- NEXT_LOCK_ID

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|NL_NEXT|BIGINT|20|-|-|-|

- NEXT_TXN_ID

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|NTXN_NEXT|BIGINT|20|-|-|-|

- NOTIFICATION_LOG
- NOTIFIVATION_SEQUENCE
- NUCLEUS_TABLES
- PARTITIONS

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|PART_ID|BIGINT|20|-|-|-|
|CREATE_TIME|INT|11|-|-|创建时间|
|LAST_ACCESS_TIME|INT|11|-|-|最后访问时间|
|PART_NAME|VARCHAR|767|[x]|-|分区名|
|SD_ID|BIGINT|20|[x]|-|外键(SDS.SD_ID)|
|TBL_ID|BIGINT|20|[x]|-|外键(TBLS.TBL_ID)|

- PARTITION_EVENTS
- PARTITION_KYES

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|TBL_ID|BIGINT|20|-|-|外键(TBLS.TBL_ID)|
|PKEY_COMMENT|VARCHAR|4000|[x]|-|分区注释|
|PKEY_NAME|VARCHAR|128|-|-|分区字段名|
|PKEY_TYPE|VARCHAR|767|-|-|分区字段类型|
|INTEGER_IDX|INT|11|-|-|分区字段顺序|

- PARTITION_KEY_VALS

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|PART_ID|BIGINT|20|-|-|外键(PARTITIONS.PART_ID)|
|PART_KEY_VAL|VARCHAR|256|-|-|分区名|
|INTEGER_IDX|INT|11|-|-|分区名顺序|

- PARTITION_PARAMS

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|PART_ID|BIGINT|20|-|-|外键(PARTITIONS.PART_ID)|
|PARAM_KEY|VARCHAR|256|-|-|分区参数名|
|PARAM_VALUE|VARCHAR|4000|-|-|分区参数值|

- PART_COL_PRIVS
- PART_COL_STATS
- PART_PRIVS
- ROLES

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|ROLE_ID|BIGINT|20|-|-|-|
|CREATE_TIME|VARCHAR|11|-|-|创建时间|
|OWNER_NAME|VARCHAR|128|[x]|-|属主名|
|ROLE_NAME|VARCHAR|128|[x]|-|角色名|

- ROLE_MAP
- SDS

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|SD_ID|BIGINT|20|-|-|-|
|CD_ID|BIGINT|20|[x]|-|外键(CDS.CD_ID)|
|INPUT_FORMAT|VARCHAR|4000|[x]|-|输入格式|
|IS_COMPRESSED|BIT|1|-|-|是否压缩|
|IS_STOREDASSUBDIRECTORIES|BIT|1|-|-|是否-|
|LOCATION|VARCHAR|4000|[x]|-|在HDFS的路径|
|NUM_BUCKETS|INT|11|-|-|分桶数|
|OUTPUT_FORMAT|VARCHAR|400|[x]|-|输出格式|
|SERDE_ID|BIGINT|20|[x]|-|外键(SERDES.SERDE_ID)|

- SD_PARAMS
- SEQUENCE_TABLE

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|SEQUENCE_NAME|VARCHAR|256|-|-|序列化器名|
|NEXT_VAL|BIGINT|20|[x]|-|下一个值|

- SERDES

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|SERDE|BIGINT|20|-|-|-|
|NAME|VARCHAR|128|[x]|-|-|
|SLIB|VARCHAR|4000|[x]|-|-|

- SERDE_PARAMS

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|SERDE_ID|BIGINT|20|-|-|外键(SERDES.SERDE_ID)|
|PARAM_KEY|VARCHAR|256|-|-|-|
|PARAM_VALUE|VARCHAR|4000|[x]|-|-|

- SKEWED_COL_NAMES
- SKEWED_COL_VALUE_LOC_MAP
- SKEWED_STRING_LIST
- SKEWED_STRING_LIST_VALUES
- SKEWED_VALUES
- SORT_COLS
- TABLE_PARAMS

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|TBL_ID|BIGINT|20|-|-|外键(TBLS.TBL_ID)|
|PARAM_KEY|VARCHAR|256|-|-|-|
|PARAM_VALUE|VARCHAR|4000|[x]|-|-|

- TAB_COL_STATS
- TBLS

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|TBL_ID|BIGINT|20|-|-|外键(TBLS.TBL_ID)|
|CREATE_TIME|INT|11|-|-|创建时间|
|DB_ID|BIGINT|20|[x]|-|外键(DBS.DB_ID)|
|LAST_ACCESS_TIME|INT|11|-|-|最后一次访问时间|
|OWNER|VARCHAR|767|[x]|-|属主|
|RETENTION|INT|11|-|-|保留|
|SD_ID|BIGINT|20|[x]|-|外键(SDS.SD_ID)|
|TBL_NAME|VARCHAR|128|[x]|-|表名|
|TBL_TYPE|VARCHAR|128|[x]|-|表类型|
|VIEW_EXPANDED_TEXT|MEDIUMTEXT|-|[x]|-|-|
|VIEW_ORIGINAL_TEXT|MEDIUMTEXT|-|[x]|-|-|

- TBL_COL_PRIVS
- TBL_PRIVS

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|TBL_GRANT_ID|BIGINT|20|-|-|-|
|CREATE_TIME|INT|11|-|-|创建时间|
|GRANT_OPTION|SMALLINT|6|-|-|是否可授权他人|
|GRANTOR|VARCHAR|128|[x]|-|授权人|
|GRANTOR_TYPE|VARCHAR|128|[x]|-|授权人类型|
|PRINCIPAL_NAME|VARCHAR|128|[x]|-|当事人|
|PRINCIPAL_TYPE|VARCHAR|128|[x]|-|当事人类型|
|TBL_PRIV|VARCHAR|128|[x]|-|表权限|
|TBL_ID|BIGINT|20|[x]|-|外键(TBLS.TBL_ID)|

- TXNS
- TXN_COMPONENTS
- TYPES
- TYPE_FIELDS
- VERSION

|名称|数据类型|长度|允许NULL|默认值|注释|
|-|-|-|-|-|-|
|VER_ID|BIGINT|20|-|-|-|
|SCHEMA_VERSION|VARCHAR|127|-|-|版本|
|VERSION_COMMENT|VARCHAR|255|[x]|-|版本注释|

- WRITE_SET