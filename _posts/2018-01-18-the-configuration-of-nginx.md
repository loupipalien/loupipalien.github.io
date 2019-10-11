---
layout: post
title: "nginx 的配置项"
date: "2018-01-18"
description: "nginx 的配置项"
tag: [nginx]
---

###　nginx 命令
```
用法:
nginx [-?hvVtq] [-s signal] [-c filename] [-p prefix] [-g directives]

选项:
-?,-h         : 帮助
-v            : 显示版本后退出
-V            : 显示版本和配置项后退出
-t            : 校验配置项后退出
-q            : 检验配置时无错误时不显示信息
-s signal     : 发送信号给 master 进程: stop, quit, reopen, reload
-p prefix     : 设置前缀路径 (默认: /usr/share/nginx/)
-c filename   : 设置配置文件路径 (默认: /etc/nginx/nginx.conf)
-g directives : 在配置文件外设置全局配置项
```
### upstream 块
```
http {
    ...
    # 后端服务器组
    upstream ltchen-ide-web {
        # 客户端 ip 分配规则
        ip_hash;
        # 后端服务器, 可配置最大失败次数, 权重等
        server 192.168.0.1:8080 max_fails = 0 weight = 1;  
    }
    ...
}
```

### server 块
```
http {
    ...
    # 虚拟服务器
    server {
        # 监听端口
        listen       80;
        # 服务器名
        server_name  dev.ltchen.com;
        # 过滤器, / 表示匹配所有请求
        location / {
            root  html;
            index index.html index.htm;
        }   
    }
    ...
}
```

### location 块
```
http {
    ...
    server {
        ...
        # 过滤器
        location /ltchen-ide-web {
            #
            proxy_pass http://ltchen-ide-web;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Cookie $http_cookie;
            log_subrequest on;
        }
        ...
    }
    ...
}
```

>**参考:**
[Nginx详细安装部署教程](https://www.cnblogs.com/taiyonghai/p/6728707.html)
