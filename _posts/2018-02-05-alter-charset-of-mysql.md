---
layout: post
title: "Mysql 的字符集"
date: "2018-02-05"
description: "Mysql 的字符集"
tag: [mysql]
---

### 查看语法
- 查看数据库编码
```
SHOW CREATE DATABASE db_name;
```
- 查看表编码
```
SHOW CREATE TABLE tbl_name;
```
- 查看字段编码
```
SHOW FULL COLUMNS FROM tbl_name;
```

### 修改语法
- 修改数据库字符集：
```
ALTER DATABASE db_name DEFAULT CHARACTER SET character_name [COLLATE ...];
```
会把表默认的字符集和所有字符列 (CHAR,VARCHAR,TEXT) 改为新的字符集：
- 修改数据表字符集
```
ALTER TABLE tbl_name CONVERT TO CHARACTER SET character_name [COLLATE ...];
```
- 修改数据字段的字符集
```
ALTER TABLE tbl_name DEFAULT CHARACTER SET character_name [COLLATE...];
```

>**参考:**
[mysql修改表、字段、库的字符集](http://fatkun.com/2011/05/mysql-alter-charset.html)
