---
layout: post
title: "一致性哈希算法"
date: "2019-01-14"
description: "一致性哈希算法"
tag: [algorithm]
---

### 普通 hash 算法
通常为了数据在多台服务器上能够负载均衡, 会取数据的 hash 值, 然后按机器总数 n 取模后选中一台服务器读写, 这样的方案在机器总数不变的情况下是可以较好的工作的; 但通常会由于服务的扩容或缩容增减服务器, 而由于服务器总数的变化, 几乎大部分数据 hash 按 n 取模后会重新选择机器, 这也会造成大部分数据的迁移读写, 如果是做缓存服务, 这会造成缓存雪崩, 后台服务器请求突高, 可能导致宕机

### 一致性 hash 算法

>****
[Consistent hashing](https://en.wikipedia.org/wiki/Consistent_hashing)
[Consistent Hashing in Java](https://web.archive.org/web/20120605030524/http://weblogs.java.net/blog/tomwhite/archive/2007/11/consistent_hash.html)
[一致性哈希算法的理解与实践](https://yikun.github.io/2016/06/09/%E4%B8%80%E8%87%B4%E6%80%A7%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E7%9A%84%E7%90%86%E8%A7%A3%E4%B8%8E%E5%AE%9E%E8%B7%B5/)
