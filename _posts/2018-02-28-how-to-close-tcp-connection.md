---
layout: post
title: "如何关闭 Tcp 链接"
date: "2018-02-27"
description: "如何关闭 Tcp 链接"
tag: [linux, tcp]
---

**环境:** CentOS-6.8-x86_64

在做 Hive 的浏览器客户端中, 常常会因为服务器断开了客户端获取的数据库连接, 客户端再次提交 Sql 时报错; 为了重现让服务端断开某个连接, 所以找到连接对应的 Tcp 链接将其断开; Google 一波发现果然有人也有类似的诉求, 并有大神开发了工具 `linux_tcp_drop`

### 编译安装 linux_tcp_drop
- 下载解压 & 进入目录 & 运行 make 命令, make 时可能会遇到如下问题
  ```
  make -C /lib/modules/2.6.32-504.el6.x86_64/build M=/app/linux-tcp-drop modules
  make: *** /lib/modules/2.6.32-504.el6.x86_64/build: No such file or directory.  Stop.
  make: *** [default] Error 2
  ```
  这是由于未安装 kernel_devel, 使用 yum 安装即可
  ```
  make -C /lib/modules/2.6.32-504.el6.x86_64/build M=/app/linux-tcp-drop modules
  expr: syntax error
  make[1]: Entering directory `/usr/src/kernels/2.6.32-504.el6.x86_64'
  /usr/src/kernels/2.6.32-504.el6.x86_64/arch/x86/Makefile:81: stack protector enabled but no compiler support
  make[1]: gcc: Command not found
    CC [M]  /app/linux-tcp-drop/tcp_drop.o
  /bin/sh: gcc: command not found
  make[2]: *** [/app/linux-tcp-drop/tcp_drop.o] Error 127
  make[1]: *** [_module_/app/linux-tcp-drop] Error 2
  make[1]: Leaving directory `/usr/src/kernels/2.6.32-504.el6.x86_64'
  make: *** [default] Error 2
  ```
  这是由于未安装 gcc, 同样使用 yum 安装即可
- 加载模块到内核中
  ```
  $ sudo insmod ./tcp_drop.ko
  ```
- 从内核中卸载 (如果不想再用)
  ```
  $ sudo rmmod tcp_drop
  ```
### 使用示例
使用 netstat 命令查出想要断开的 Tcp 链接
```
$ netstat -atuln | grep 10000

tcp        0      0 0.0.0.0:10000               0.0.0.0:*                   LISTEN      
tcp        0      0 10.202.120.12:10000         10.202.120.12:49599         ESTABLISHED
tcp        0      0 10.202.120.12:48116         10.202.120.12:10000         ESTABLISHED
tcp        0      0 10.202.120.12:49860         10.202.120.12:10000         ESTABLISHED
tcp        0      0 10.202.120.12:10000         10.202.120.12:49860         ESTABLISHED
tcp        0      0 10.202.120.12:10000         10.202.120.12:48116         ESTABLISHED
tcp        0      0 10.202.120.12:49599         10.202.120.12:10000         ESTABLISHED
```
将想断开的 Tcp 链接, 把 Local Address 和 Foreign Address 写入 /proc/net/tcp_drop 文件中即可
```
echo "10.202.120.12:49599         10.202.120.12:10000" > /proc/net/tcp_drop
```

### Kernel 版本大于 3.9 时编译报错
```
make -C /lib/modules/3.10.0-327.el7.x86_64/build M=/app/linux-tcp-drop modules
make[1]: Entering directory `/usr/src/kernels/3.10.0-327.el7.x86_64'
  CC [M]  /app/linux-tcp-drop/tcp_drop.o
/app/linux-tcp-drop/tcp_drop.c:178:8: error: expected ‘=’ before ‘:’ token
 .write : tcp_drop_write_proc,
        ^
/app/linux-tcp-drop/tcp_drop.c:149:1: warning: ‘tcp_drop_write_proc’ defined but not used [-Wunused-function]
 tcp_drop_write_proc(struct file *file, const char __user *buffer,
 ^
make[2]: *** [/app/linux-tcp-drop/tcp_drop.o] Error 1
make[1]: *** [_module_/app/linux-tcp-drop] Error 2
make[1]: Leaving directory `/usr/src/kernels/3.10.0-327.el7.x86_64'
make: *** [default] Error 2
```
因为 Kernal-3.9 版本后, create_proc_entry()函数已经被proc_create()函数取代, 导致编译报错; 解决方法见 [issue 5](https://github.com/arut/linux-tcp-drop/issues/5)

>**参考:**
[linux不重启服务的情况下关闭某条链路的tcp连接](http://blog.csdn.net/fox_hacker/article/details/53115583)  
[linux-tcp-drop](https://github.com/arut/linux-tcp-drop)
