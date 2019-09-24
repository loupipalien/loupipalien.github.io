---
layout: post
title: "轮询算法"
date: "2019-01-08"
description: "轮询算法"
tag: [algorithm]
---

### 轮询
轮询调度算法按轮询方式分发作业到所有的服务器; 例如, 在轮询调度中三个服务器 (服务器 A, B, C), 第一个请求将会到服务器 A, 第二个请求将会到服务器 B, 第三个请求会到服务器 C, 第四个请求会到服务器 A, 然后重复这种轮询方式; 用数学方式表示, 即每次按照 `i = (i + 1) mod n` 选取服务器, 其中 n 是服务器的数量  
在 [IPVS](http://kb.linuxvirtualserver.org/wiki/IPVS) 实现中, 有一个额外的条件被引入到轮询调度中, 即检查服务器是否可用, 因为服务器可能因为服务器故障或者系统维护被移出; 轮询调度的处理流程如下
```
# 假定这里有 n 个服务器, 用集合 S 表示, S = {S_0, S1, ..., Sn-1} , 其中 n > 0;
# i 表示上次选中的服务器, i 的初始值为 -1
j = i;
do {
    j = (j + 1) mod n;
    if (Available(Sj)) {
        i = j;
        return Si;
    }
} while (j != i);
return NULL;
```
这里认为所有真实服务器都是相等的, 忽略连接数和每台服务器的响应时间; 它不需要记录每台服务器的状态, 因此这是一个无状态的调度算法  
[IP Virtual Server](http://kb.linuxvirtualserver.org/wiki/IPVS) 的轮询调度相比传统的轮询 DNS 提供了一些优势; 轮询 DNS 解析单个域名到不同的 IP 地址, 调度粒度是基于主机名的; 并且 DNS 的查询缓存也阻碍了基础算法实现, 这些因素在真实主机间导致严重的动态负载不均衡; [IPVS](http://kb.linuxvirtualserver.org/wiki/IPVS) 的调度粒度是每网络连接, 由于更小的调度粒度, 轮询调度比轮询 DNS 更有优越性  
轮询调度算法最大的有优点是在软件中容易实现; 是一个简单的经典调度算法, 被广泛的用于现代调度系统

>**参考:**
[Round-Robin Scheduling](http://kb.linuxvirtualserver.org/wiki/Round-Robin_Scheduling)
