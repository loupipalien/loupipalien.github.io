### 命令行客户端
#### 进入 shell
```
hbase shell
```

### 表管理
#### 查看所有表
```
hbase(main)> list
```
#### 创建表
```
# 语法: create <table>, {NAME => <family>, VERSIONS => <VERSIONS>}
# 例如: 创建表t1, 有两个family name: f1, f2, 且版本数均为2
hbase(main)> create 't1',{NAME => 'f1', VERSIONS => 2},{NAME => 'f2', VERSIONS => 2}
```
####　删除表
```
# 需要先禁止表再删除
# 例如: 删除 t1 表
hbase(main)> disable 't1'
hbase(main)> drop 't1'
```
#### 查看表结构
```
# 语法: describe <table>
# 例如: 查看表 t1 的结构
hbase(main)> describe 't1'
```
#### 修改表结构
```
# 修改表结构需要先 disable 表, 修改后再 enable 表
# 语法: alter 't1', {NAME => 'f1'}, {NAME => 'f2', METHOD => 'delete'}
# 例如: 修改表 t1 的 column family 的 ttl
hbase(main)> disable 't1'
hbase(main)> alter 't1',{NAME=>'f1',TTL=>'1555200'},{NAME=>'f2', TTL=>'15552000'}
hbase(main)> enable 't1'
```

### 权限管理
#### 授予权限
```
# 语法: grant <user>, <permissions> [, <@namespace> [, <table> [, <column family> [, <column qualifier>]]] qualifier> 参数后面用逗号分隔
# 权限用五个字母表示: "RWXCA", READ('R'), WRITE('W'), EXEC('X'), CREATE('C'), ADMIN('A')
# 例如: 授予用户 'test' 对表 t1 的读写权限
hbase(main)> grant 'test','RW','t1'
```
#### 查看表的权限
```
# 语法: user_permission <table>
# 例如: 查看表 t1 的权限列表
hbase(main)> user_permission 't1'
```
#### 回收权限
```
# 语法: revoke <user> [, <@namespace> [, <table> [, <column family> [, <column qualifier>]]]]
# 例如: 收回用户 'test' 对表 t1 的读写权限
hbase(main)> revoke 'test','t1'
```

### 表数据的增删改查
#### 查询行数据
```
# 语法: get <table>,<rowkey>,[<family:column>,....]
# 例如: 查询表 t1, rowkey001 中的 f1 下的 col1 的值
hbase(main)> get 't1','rowkey001','f1:col1'
# 或者:
hbase(main)> get 't1','rowkey001', {COLUMN=>'f1:col1'}
# 查询表t1, rowke001 中的 f1 下的所有列值
hbase(main)> get 't1','rowkey001'
```
#### 扫描表数据
```
# 语法: scan <table>, {COLUMNS => [ <family:column>,.... ], LIMIT => num}
# 另外, 还可以添加 STARTROW, TIMERANGE 和 FITLER 等高级功能
# 例如: 扫描表t1的前5条数据
hbase(main)> scan 't1',{LIMIT=>5}
```
#### 统计表数据
```
# 语法: count <table>, {INTERVAL => intervalNum, CACHE => cacheNum}
# INTERVAL 设置多少行显示一次及对应的 rowkey, 默认1000; CACHE 每次去取的缓存区大小, 默认是10, 调整该参数可提高查询速度
# 例如，查询表t1中的行数, 每100条显示一次, 缓存区为500
hbase(main)> count 't1', {INTERVAL => 100, CACHE => 500}
```
####　新增数据
```
# 语法: put <table>,<rowkey>,<family:column>,<value>,<timestamp>
# 例如: 给表 t1 的添加一行记录: rowkey 是 rowkey001, family name: f1, column name: col1, value:value01, timestamp: 系统默认
hbase(main)> put 't1','rowkey001','f1:col1','value01'
```
#### 删除列数据
```
# 语法: delete <table>, <rowkey>, <family:column>, <timestamp>, 必须指定列名
# 例如: 删除表 t1, rowkey001 中的 f1:col1 的数据
hbase(main)> delete 't1','rowkey001','f1:col1'
```
#### 删除行数据
```
# 语法: delete <table>, <rowkey>, <family:column>, <timestamp>, 可以不指定列名, 删除整行数据
# 例如: 删除表 t1, rowkey001 的数据
hbase(main)> delete 't1','rowkey001'
```
#### 删除表数据
```
# 语法: truncate <table>
# 其具体过程是: disable table -> drop table -> create table
# 例如: 删除表 t1 的所有数据
hbase(main)> truncate 't1'
```

###　Region 管理

### 配置管理及节点重启

> **参考:**
[Hbase 常用 Shell 命令](http://www.cnblogs.com/nexiyi/p/hbase_shell.html)
