---
layout: post
title: "Curator Recipes 中的 Caches"
date: "2018-12-07"
description: "Curator Recipes 中的 Caches"
tag: [zookeeper, curator]
---

### 为什么使用 Caches
原生的 ZooKeeper 使用 Watcher 对节点事件监听非常不方便，一个 Watcher 只能生效一次, 也就是说每次进行监听回调之后需要重新的设置监听才能达到永久监听的效果; Curator Recipes 中提供了 Caches 来实现了对 ZooKeeper 服务端的事件监听, 对其事件监听可以看做是一个本地缓存和远程 ZooKeeper 视图进行对比的过程, 不需要重复注册就可以永久监听

### Caches 分类
- Path Cache
Path Caches 用于监听一个 ZNode, 无论是子节点的添加, 更新, 移除; Path Caches 将会更改它的状态来包含子节点集合, 子节点的数据, 子节点的状态; 在 Curator 框架中提对 Path Caches, 提供了 PathChildrenCache class; 路径的改变将会传递给注册的 PathChildrenCacheListener 实例
- Node Cache  
试图维持本地缓存的节点数据工具; 这个类将会监控节点, 对 update/create/delete 事件响应, 拉取数据; 你可以注册一个监听器, 在变化发生时会获得通知
- Tree Cache
视图维持本地缓存的 ZK 路径下所有子节点数据的工具; 这个类将会监控 ZK 路径, 对 update/create/delete 事件响应, 拉取数据; 可以注册一个监听器, 在变化发生时会获得通知

### 使用 Path Cache 监听
#### 依赖的 Jar 包
```
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>4.0.1</version>
</dependency>
```
#### 使用示例
```
public static void main(String[] args) {
        String connectString = "192.168.0.1:2181";
        String zNodePath =  "/hiveserver2";
        try (CuratorFramework client = CuratorFrameworkFactory.newClient(connectString, new ExponentialBackoffRetry(1000, 3))) {
            client.start();
            // cacheDate 设置为 true 可以获取变化的数据
            PathChildrenCache cache = new PathChildrenCache(client, zNodePath , true);
            cache.getListenable().addListener((curatorFramework, event) -> {
                String data = new String(event.getData().getData(), Charset.forName("UTF-8"));
                System.out.println("Type: " + event.getType() + ", Data: " + data);
            });
            // 启动方式有 NORMAL, BUILD_INITIAL_CACHE, POST_INITIALIZED_EVENT
            cache.start();
            // 此处睡眠以触发监听事件
            TimeUnit.SECONDS.sleep(Long.MAX_VALUE);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```
需要注意的一点是, 监听到的事件顺序不一定和真实发生顺序一致, 可以通过数据自身带的序列号做事件排序

>**参考:**
[Recipes](https://curator.apache.org/curator-recipes/index.html)  
[Path Cache](https://curator.apache.org/curator-recipes/path-cache.html)  
[Curator的三种缓存](https://blog.csdn.net/Leafage_M/article/details/78735485)
