---
layout: post
title: "加权最少连接算法"
date: "2019-01-11"
description: "加权最少连接算法"
tag: [algorithm]
---

### 加权最少连接算法
加权最少连接算法是最少连接算法的超集, 可以为每台真实服务器分配一个性能权重; 高权重值的服务器将会接收到同一时间的大部分活跃连接; 默认服务器的权重是 1, 并且 [IPVS] 管理者或者监控程序可以分配任何权重给真实服务器; 在加权最少连接调度中, 新的网络连接会被指派到当前活跃连接数与权重比值最低的服务器上; 加权最少连接调度工作流程如下
```
# 假定服务器集合 S = {S0, S1, ..., Sn-1}
# W(Si) 为服务器 Si 的权重
# C(Si) 为服务器的当前连接数
# CSUM = ΣC(Si) (i=0, 1, .. , n-1) 是当前连接数的总和
#
# 新连接会被分配到服务器 j, (C(Sm) / CSUM)/ W(Sm) = min { (C(Si) / CSUM) / W(Si)}  (i=0, 1, . , n-1), 其中 W(Si) 不等于 0
# 因为在一次循环中 CSUM 为常量, 所以不需要再除以 CSUM, 条件可以被优化为 C(Sm) / W(Sm) = min { C(Si) / W(Si)}  (i=0, 1, . , n-1), 其中 W(Si) 不等于 0
# 因为除操作比成操作更吃 CPU 时间片, 并且 Linux 内核中不允许浮点模式, 条件 C(Sm)/W(Sm) > C(Si)/W(Si) 可以被优化为 C(Sm)*W(Si) > C(Si)*W(Sm)
# 调度会保证不会调度到权重为 0 的服务器, 加权最少连接调度算法的伪代码如下

for (m = 0; m < n; m++) {
    if (W(Sm) > 0) {
        for (i = m+1; i < n; i++) {
            if (C(Sm)*W(Si) > C(Si)*W(Sm))
                m = i;
        }
        return Sm;
    }
}
return NULL;
```
加权最少连接调度算法比最少连接算法需要额外的除操作; 在相同处理性能的服务器下, 最少连接调度和加权最少连接调度都实现了最小化的调度开销

>**参考:**
[Weighted Least-Connection Scheduling](http://kb.linuxvirtualserver.org/wiki/Weighted_Least-Connection_Scheduling)
[Use of floating point in the Linux kernel](https://stackoverflow.com/questions/13886338/use-of-floating-point-in-the-linux-kernel)
