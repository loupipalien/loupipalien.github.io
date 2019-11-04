---
layout: post
title: "基于 Zookeeper 的分布式锁"
date: "2019-10-08"
description: "基于 Zookeeper 的分布式锁"
tag: [zookeeper, java]
---

### 基于 Zookeeper 的分布式锁
分布式锁目前有三种主流方案, 分别为基于数据库, Redis, Zookeeper 的方案; 这里主要叙述 Zookeeper 实现分布式锁的机制

#### Zookeeper 分布式锁的流程
假定有两个客户端 (A 和 B) 使用同一个分布式锁
![Image]()  
锁其实是 Zookeeper 上的一个节点 (假定节点路径为 `/lock`), 客户端在竞争锁时首先在此节点下创建临时顺序节点, Zookeeper 会维护节点的顺序与请求顺序一致; 假定客户端 A 先请求, 这时会先创建一个临时顺序节点
![Image]()  
紧接着客户端 A 会获取 `/lock` 这个节点下的所有子节点, 检查第一个节点是否是自己创建的节点, 如果是则可以加锁; 由于客户端 A 是第一个请求的, 所以成功获取了锁
![Image]()  
客户端 B 此时也想要获取锁, 发起请求后也会先创建一个临时顺序节点
![Image]()  
紧接着也获取 `/lock` 这个节点下的所有子节点, 检查第一个节点是否是自己创建的节点; 由于客户端 A 此时是第一个节点, 客户端 B 加锁失败; 失败后客户端 B 会通过 Zookeeper 的 API 对它的临时顺序节点的上一个节点 (就是客户端 A 的临时顺序节点) 添加一个监听器, 监听上一个节点是否被删除
![Image]()  
客户端 A 加锁之后, 处理完业务逻辑后释放了锁, 释放锁就会把之前创建的临时节点删除; Zookeeper 会将此事件通知给监听此节点的监听器, 客户端 B 加的监听器获取了此事件, 接着通知客户端 B 在它之前的客户端释放了锁, 于是客户端 B 重新尝试获取锁, 即获取 `/lock` 这个节点下的所有子节点, 检查第一个节点是否是自己创建的节点; 由于客户端 B 创建的节点排在第一个, 所以成功获取了锁
![Image]()
客户端 B 加锁之后, 处理完处理完业务逻辑后, 锁再次被释放

#### Zookeeper 分布式锁的机制
当多个客户端竞争一个 Zookeeper 分布式锁, 原理都是类似的
- 各个客户端在请求后在锁节点下创建一个临时节点
- 如果创建的临时节点不是第一个, 则对前一个节点加监听器
- 上一个节点释放锁, 自己就可以获得锁, 这相当于一个派对机制

另外, 如果客户端宕机, Zookeeper 会检测到与其之间的心跳超时, 会自动删除对应的临时顺序节点, 这相当于自动释放锁

#### Zookeeper 分布式锁的 Curator 实现
客户端获取锁, 这里使用 Curator 的实现, 需要依赖 `curator-recioes` 包
```
public static void main(String[] args) throws Exception {
    // 创建 Zookeeper 的客户端
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    CuratorFramework client = CuratorFrameworkFactory.newClient("192.169.0.1", retryPolicy);
    client.start();

    // 创建分布式锁, 锁节点路径为 /lock
    InterProcessMutex mutex = new InterProcessMutex(client, "/lock");
    mutex.acquire();
    // 获得了锁, 进行业务流程
    System.out.println("Enter mutex");
    // 完成业务流程, 释放锁
    mutex.release();

    //关闭客户端
    client.close();
}
```
Curator 封装的获取锁释放锁接口非常简洁, 首先来看 `acquire` 方法
```
@Override
public void acquire() throws Exception {
    // 获取锁, 当锁被占用时会阻塞等待, 支持同线程的可重入 (也就是可重入锁)
    if (!internalLock(-1, null)) {
        throw new IOException("Lost connection while trying to acquire lock: " + basePath);
    }
}
```
当与 Zookeeper 通信存在异常时会抛出异常; 未获得锁会永久阻塞等待 (也有超时等待的方法); 接着进入 `internalLock` 方法
```
private boolean internalLock(long time, TimeUnit unit) throws Exception{
    Thread currentThread = Thread.currentThread();

    LockData lockData = threadData.get(currentThread);
    if (lockData != null) {
        // 重入时
        lockData.lockCount.incrementAndGet();
        return true;
    }

    // 获取锁
    String lockPath = internals.attemptLock(time, unit, getLockNodeBytes());
    if (lockPath != null) {
        LockData newLockData = new LockData(currentThread, lockPath);
        // 获得锁之后, 记录在当前线程获得锁的信息
        threadData.put(currentThread, newLockData);
        return true;
    }

    return false;
}
```
这个方法内主要做了可重入锁的支持, 以及调用获取锁的接口; 进入 `attemptLock` 方法来获取锁
```
String attemptLock(long time, TimeUnit unit, byte[] lockNodeBytes) throws Exception {
    final long startMillis = System.currentTimeMillis();
    final Long millisToWait = (unit != null) ? unit.toMillis(time) : null;
    final byte[] localLockNodeBytes = (revocable.get() != null) ? new byte[0] : lockNodeBytes;
    int retryCount = 0;

    String ourPath = null;
    boolean hasTheLock = false;
    boolean isDone = false;
    while (!isDone) {
        isDone = true;

        try {
            // (StandardLockInternalsDriver) 创建临时顺序子节点, 并返回此子节点路径
            ourPath = driver.createsTheLock(client, path, localLockNodeBytes);
            // 判断是否获得锁 (子节点序号最小), 获得锁则直接返回, 否则阻塞等待前一个子节点删除通知
            hasTheLock = internalLockLoop(startMillis, millisToWait, ourPath);
        }
        catch (KeeperException.NoNodeException e) {
            // 当 StandardLockInternalsDriver 没有找到锁节点, 会话过期时会发生; 如果允许重试则再尝试一次
            if (client.getZookeeperClient().getRetryPolicy().allowRetry(retryCount++, System.currentTimeMillis() - startMillis, RetryLoop.getDefaultRetrySleeper())) {
                isDone = false;
            } else {
                throw e;
            }
        }
    }

    if (hasTheLock) {
        return ourPath;
    }

    return null;
}
```
在这个方法中, 会先在 Zookeeper 中创建临时顺序节点, 然后调用 `internalLockLoop` 方法获取锁
```
private boolean internalLockLoop(long startMillis, Long millisToWait, String ourPath) throws Exception {
    boolean haveTheLock = false;
    boolean doDelete = false;
    try {
        if (revocable.get() != null) {
            client.getData().usingWatcher(revocableWatcher).forPath(ourPath);
        }

        // 这里使用对象监视器做线程同步, 当获取不到锁时监听前一个子节点删除消息并且调用 wait, 当前一个子节点删除时, 回调会通过 notifyAll 唤醒此线程, 此线程继续自旋判断是否获得锁
        while ((client.getState() == CuratorFrameworkState.STARTED) && !haveTheLock) {
            // 获取排序的子节点
            List<String> children = getSortedChildren();
            // 获取当前子节点名称
            String sequenceNodeName = ourPath.substring(basePath.length() + 1);
            // 判断是否是第一个子节点, 如果不是则获取需要监听的路径
            PredicateResults predicateResults = driver.getsTheLock(client, children, sequenceNodeName, maxLeases);
            if (predicateResults.getsTheLock()) {
                haveTheLock = true;
            } else {
                String  previousSequencePath = basePath + "/" + predicateResults.getPathToWatch();

                synchronized(this) {
                    try
                    {
                        // use getData() instead of exists() to avoid leaving unneeded watchers which is a type of resource leak
                        // 使用 getData() 而不是 checkExists() 是为了避免留下不必要的监听器, 这对 Zookeeper 来说是一种资源泄露
                        client.getData().usingWatcher(watcher).forPath(previousSequencePath);
                        // 如果设置了超时
                        if (millisToWait != null) {
                            millisToWait -= (System.currentTimeMillis() - startMillis);
                            startMillis = System.currentTimeMillis();
                            if (millisToWait <= 0) {
                                // 超时需删除创建的子节点
                                doDelete = true;   
                                break;
                            }

                            wait(millisToWait);
                        } else {
                            wait();
                        }
                    }
                    catch (KeeperException.NoNodeException e) {
                        // 当前一个节点已经被删除 (锁被释放), 重试一次即可
                    }
                }
            }
        }
    }
    catch (Exception e) {
        ThreadUtils.checkInterrupted(e);
        doDelete = true;
        throw e;
    } finally {
        if (doDelete) {
            deleteOurPath(ourPath);
        }
    }
    return haveTheLock;
}
```
这个方法可以看到, 当没有设置获取超时时间, 会永久自旋等待直到获取了锁

>**参考:**
[七张图彻底讲清楚ZooKeeper分布式锁的实现原理](https://juejin.im/post/5c01532ef265da61362232ed)
[基于Zookeeper的分布式锁](http://www.dengshenyu.com/java/%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F/2017/10/23/zookeeper-distributed-lock.html)  
[基于Zookeeper的分布式锁](http://www.dengshenyu.com/java/%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F/2017/10/23/zookeeper-distributed-lock.html)
