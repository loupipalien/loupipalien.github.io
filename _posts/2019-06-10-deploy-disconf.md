---
layout: post
title: "部署 Disconf"
date: "2019-06-10"
description: "部署 Disconf"
tag: [java]
---

### 部署 disconf
```
$ cd /app
$ git clone https://github.com/knightliao/disconf.git
```
本文中的相对路径都是相对于 disconf 的目录

#### 部署依赖
disconf 部署依赖以下服务
- Mysql: 用户, 应用, 环境,角色等数据管理
初始化数据, 按照 `disconf-web/sql/readme.md` 说明操作
- Redis: 登录 session 管理
- Zookeeper: 配置信息管理
- Nginx: 处理静态资源请求, 转发动态资源请求到 Tomcat
- Tomcat: 部署 disconf-web 应用


#### 配置文件
这里先将模板文件拷贝到一个目录下, `cp disconf-web/profile/rd/* disconf-rd/online-resources`, 修改其中文件
##### application.properties
这里的 domain 换成访问 web 时的服务器名 (即 nginx 中的 server_name 的配置), 邮箱如不使用可以不用修改
```
# 服务器的 domain
domain=192.168.127.100

# 邮箱设置
EMAIL_MONITOR_ON = true
EMAIL_HOST = smtp.163.com
EMAIL_HOST_PASSWORD = password
EMAIL_HOST_USER = sender@163.com
EMAIL_PORT = 25
DEFAULT_FROM_EMAIL = disconf@163.com

# 定时校验中心的配置与所有客户端配置的一致性
CHECK_CONSISTENCY_ON= true
```
#####　jdbc-mysql.properties
```
jdbc.driverClassName=com.mysql.jdbc.Driver

jdbc.db_0.url=jdbc:mysql://192.168.127.100:3306/disconf?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&rewriteBatchedStatements=false
jdbc.db_0.username=root
jdbc.db_0.password=123456

jdbc.maxPoolSize=20
jdbc.minPoolSize=10
jdbc.initialPoolSize=10
jdbc.idleConnectionTestPeriod=1200
jdbc.maxIdleTime=3600
```
##### redis-config.properties
这里在本机部署两个 redis 实例, 并且都没有设置密码, 所以 password 的配置为空
```
redis.group1.retry.times=2

redis.group1.client1.name=BeidouRedis1
redis.group1.client1.host=192.168.127.100
redis.group1.client1.port=6379
redis.group1.client1.timeout=5000
redis.group1.client1.password=

redis.group1.client2.name=BeidouRedis2
redis.group1.client2.host=192.168.127.100
redis.group1.client2.port=6380
redis.group1.client2.timeout=5000
redis.group1.client2.password=

redis.evictor.delayCheckSeconds=300
redis.evictor.checkPeriodSeconds=30
redis.evictor.failedTimesToBeTickOut=6
```
##### zoo.properties
```
hosts=192.168.127.100:2181

# zookeeper 的前缀路径名
zookeeper_url_prefix=/disconf
```

#### 编译构建
```
cd disconf-web
export ONLINE_CONFIG_PATH=/app/disconf/disconf-rd/online-resources
export WAR_ROOT_PATH=/app/disconf/disconf-rd/war
sh deploy/deploy.sh`
```

#### 配置 tomcat 和 nginx
##### tomcat
修改 tomcat 的 `conf/server.xml` 文件, 在 `<Host>` 元素中添加以下 `<Context>` 元素
```
<Context path="" docBase="/app/disconf/disconf-rd/war"/>
```
然后启动 tomcat (默认使用 8080 端口)
##### nginx
新增文件 `/etc/nginx/conf.d/disconf.conf` (如果安装的 nginx 默认使用 `/etc/nginx/nginx.conf` 文件, 且其中加载了 `conf.d` 目录下的 conf 后缀的文件)
```
upstream disconf {
    server 192.168.127.100:8080;
}

server {

    listen   80;
    server_name 192.168.127.100;
    access_log /home/work/var/logs/disconf/access.log;
    error_log /home/work/var/logs/disconf/error.log;

    location / {
        root /home/work/dsp/disconf-rd/war/html;
        if ($query_string) {
            expires max;
        }
    }

    location ~ ^/(api|export) {
        proxy_pass_header Server;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_pass http://192.168.127.100:8080;
    }
}
```
检验 nginx 配置正确后启动 nginx

#### 访问 Disconf
默认账户: `admin/admin`  
![image](https://ws1.sinaimg.cn/large/d8f31fa4ly1g4ek4f2wxwj20u60he3zc.jpg)

>**参考:**  
[如何在 CentOS 7 上安装 Disconf](https://www.zuojl.com/how-to-install-disconf-on-centos7/)  
[Tomcat里 appBase和docBase的区别](https://blog.csdn.net/liuxuejin/article/details/9104055)    
[disconf-web安装](https://disconf.readthedocs.io/zh_CN/latest/install/src/02.html)    
[Disconf实践指南：安装篇](https://blog.csdn.net/u011116672/article/details/73395707)  
