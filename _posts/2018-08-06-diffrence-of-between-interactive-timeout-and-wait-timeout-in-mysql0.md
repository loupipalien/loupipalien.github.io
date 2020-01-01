---
layout: post
title: "Mysql 中 interactive_timeout 和 wait_timeout 的区别"
date: "2018-08-06"
description: "Mysql 中 interactive_timeout 和 wait_timeout 的区别"
tag: [mysql]
---

### interactive_timeout
在关闭一个交互式连接执之前, 服务器等待活动的秒数; 客户端在 [mysql_real_connection](https://dev.mysql.com/doc/refman/8.0/en/mysql-real-connect.html) 中使用 CLIENT_INTERACTIVE 选项, 即表明是交互式客户端; 参见 [wait_timeout](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_wait_timeout)

|属性|属性值|
|-|-|
|命令行格式|--interactive-timeout=#|
|系统变量名|interactive_timeout|
|范围|全局, 会话|
|动态的|是|
|SET_VAR 提示应用|否|
|类型|Integer|
|默认值|28800|
|最小值|1|

### wait_timeout
在关闭一个非交互式连接执之前, 服务器等待活动的秒数; 在线程启动时, 会话的 wait_timeout 会从全局的 wait_timeout 值或从全局的 interactive_timeout 值继承, 其取决于客户端的类型 (在 [mysql_real_connection](https://dev.mysql.com/doc/refman/8.0/en/mysql-real-connect.html) 中使用 CLIENT_INTERACTIVE 选项定义); 参见 [interactive_timeout](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_interactive_timeout)

|属性|属性值|
|-|-|
|命令行格式|--wait-timeout=#|
|系统变量名|wait_timeout|
|范围|全局, 会话|
|动态的|是|
|SET_VAR 提示应用|否|
|类型|Integer|
|默认值|28800|
|最小值|1|
|最大值 (Other)|31536000|
|最小值 (Windows)|2147483|

#### 查看与设置
- 查看
```
mysql> select variable_name,variable_value from information_schema.global_variables where variable_name in ('interactive_timeout','wait_timeout');
+---------------------+----------------+
| variable_name       | variable_value |
+---------------------+----------------+
| INTERACTIVE_TIMEOUT | 28800          |
| WAIT_TIMEOUT        | 28800          |
+---------------------+----------------+
2 rows in set (0.13 sec)
```
- 设置
```
set global interactive_timeout=864000;
set global wait_timeout=864000;
```
- show processlist
```
mysql> show processlist;
+----+------+----------------------+------+---------+------+-------+------------------+
| Id | User | Host                 | db   | Command | Time | State | Info             |
+----+------+----------------------+------+---------+------+-------+------------------+
|  2 | root | localhost            | NULL | Query   |    0 | init  | show processlist |
|  6 | repl | 192.168.244.20:44641 | NULL | Sleep   | 1154 |       | NULL             |
+----+------+----------------------+------+---------+------+-------+------------------+
2 rows in set (0.03 sec)
```

>**参考:**
[Server System Variables](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html)
[MySQL中interactive_timeout和wait_timeout的区别](http://www.cnblogs.com/ivictor/p/5979731.html)
