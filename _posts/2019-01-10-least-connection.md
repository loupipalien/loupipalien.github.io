---
layout: post
title: "最少连接算法"
date: "2019-01-10"
description: "最少连接算法"
tag: [algorithm]
---

### 最少连接
最少连接调度算法会将网络连接分发到建立最少连接数的服务器, 这是一个动态调度算法; 因为它需要统计每台服务器的连接数, 动态的估计服务器的负载; [负载均衡器](http://kb.linuxvirtualserver.org/wiki/Load_balancer) 记录了每台服务器的连接数, 当一个新的连接分发到服务器时增加服务器的连接数, 当连接完成或者超时则减少服务器连接数  
在 [IPVS](http://kb.linuxvirtualserver.org/wiki/IPVS) 实现中, 在最少连接调度中引入了一个额外的条件, 即检查服务器是否可用, 因为服务器会被移出服务因为服务宕机或系统维护, 服务器权重为 0 意味着服务器不可用; 最少连接调度的正常处理流程如下
```
# 假定服务器集合 S = {S0, S1, ..., Sn-1}
# W(Si) 为服务器 Si 的权重
# C(Si) 为服务器 Si 的当前连接数
for (m = 0; m < n; m++) {
    if (W(Sm) > 0) {  // 先找到第一台可用的服务器
        for (i = m+1; i < n; i++) {  // 选择连接数最少的服务器
            if (W(Si) <= 0)
                continue;
            if (C(Si) < C(Sm))
                m = i;
        }
        return Sm;
    }
}
return NULL;
```
对于性能相近的一组服务器集合的 [IPVS](http://kb.linuxvirtualserver.org/wiki/IPVS) 集群, 当请求负载非常高时, 最少连接调度有着非常平滑的分布; [负载均衡器](http://kb.linuxvirtualserver.org/wiki/Load_balancer) 会请求到最少连接的真实服务器  
初看起来, 最少连接调度似乎可以执行的很好, 甚至是在服务器处理能力差别很大的情况下, 因为最快的服务器将会获得更多的网络连接; 但事实上, 由于 TCP 的 TIME_WAIT 状态并不能执行的非常好; TCP 的 TIME_WAIT 通常是 2 分钟, 在这 2 分钟可能接收到上千个连接; 例如, 假设服务器 A 是服务器 B 性能的两倍, 服务器 A 处理上千个连接并且保持在了 TIME_WAIT 状态, 但服务器 B 缓慢的完成了上千个连接; 所以, 在服务器处理性能差别较大时, 最少连接调度不能很好的负载均衡

>**参考:**
[Least-Connection Scheduling](http://kb.linuxvirtualserver.org/wiki/Least-Connection_Scheduling)
