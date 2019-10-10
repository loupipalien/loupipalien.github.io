---
layout: post
title: "Guava 用户指南之缓存"
date: "2019-09-10"
description: "Guava 用户指南之缓存"
tag: [java, guava]
---

### 缓存
```Java
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .removalListener(MY_LISTENER)
        .build(
            new CacheLoader<Key, Graph>() {
                public Graph load(Key key) throws AnyException {
                    return createExpensiveGraph(key);
                }
        });
```

#### 适用性
缓存在很多场景下非常有用; 例如, 计算或检索一个值的代价很高, 并且对同样的输入需要不止一次获取值时, 就应当考虑使用缓存  
Guava Cache 与 ConcurrentMap 很相似, 但不完全一致; 最基本的区别是 ConcurrentMap 会一直保存所有添加的元素, 直到显式地移除; 而 Guava Cache 为了限制内存占用, 通常都设定为自动回收元素; 在某些场景下, 尽管 LoadingCache 不回收元素, 它也是很有用的, 因为它会自动加载缓存; 通常来说, Cache 适用于
- 愿意消耗一些内存空间来提升速度
- 可以预料到某些键会被查询多次
- 缓存中存放的数据量不会超过内存 (Guava Cache 是本地运行时缓存, 如果需要外置缓存数据, 请考虑 Memcached, Redis 等)

如同一开始的示例, Cache 实例通过 CacheBuilder 生成器模式获取 (如果不需要 Cache 中的特性, 使用 ConcurrentHashMap 有更好的内存效率)

#### 加载
在使用缓存前, 考虑有没有合理的默认方法来加载或计算与键关联的值?
- 如果有的, 应当使用 CacheLoader
- 如果没有，或者想要覆盖默认的加载运算, 同时保留 "获取缓存-如果没有-则计算" `[get-if-absent-compute]` 的原子语义, 应该在调用 get 时传入一个 Callable 实例; 缓存元素也可以通过 Cache.put 方法直接插入, 但自动加载是首选的, 因为它可以更容易地推断所有缓存内容的一致性

##### CacheLoader
LoadingCache 是基于 CacheLoader 构建而成的缓存实现; 自定义创建 CacheLoader 通常只需要简单地实现 `V load(K key) throw Exception` 方法; 例如使用以下代码构建 LoadingCache
```Java
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
        .maximumSize(1000)
        .build(
            new CacheLoader<Key, Graph>() {
                public Graph load(Key key) throws AnyException {
                    return createExpensiveGraph(key);
                }
            });
...
try {
    return graphs.get(key);
} catch (ExecutionException e) {
    throw new OtherException(e.getCause());
}
```
从 LoadingCache 查询的方式是使用 get(K) 方法; 这个方法要么返回已经缓存的值, 要么使用 CacheLoader 向缓存原子地加载新值; 由于 CacheLoader 可能抛出异常, LoadingCache.get(K) 也声明为抛出 ExecutionException 异常; 如果定义的 CacheLoader 没有声明任何检查型异常，则可以通过 getUnchecked(K) 查找缓存; 但必须注意，一旦 CacheLoader 声明了检查型异常就不要调用 getUnchecked(K)
```Java
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
        .expireAfterAccess(10, TimeUnit.MINUTES)
        .build(
            new CacheLoader<Key, Graph>() {
                public Graph load(Key key) { // no checked exception
                    return createExpensiveGraph(key);
                }
            });
...
return graphs.getUnchecked(key);
```
`getAll(Iterable<? extends K>)` 方法用来执行批量查询; 默认情况下, 对每个不在缓存中的键, getAll 方法会单独调用 CacheLoader.load 来加载缓存项; 如果批量的加载比多个单独加载更高效, 可以重载 CacheLoader.loadAll 来利用这一点; getAll(Iterable) 的性能也会相应提升

##### Callable
所有类型的 Guava Cache, 不管有没有自动加载功能, 都支持 get(K, Callable<V>) 方法; 这个方法返回缓存中相应的值, 或者用给定的 Callable 运算并把结果加入到缓存中; 在整个加载方法完成前, 缓存项相关的可观察状态都不会更改; 这个方法简便地实现了模式 "如果有缓存则返回; 否则运算, 缓存, 然后返回"
```Java
Cache<Key, Graph> cache = CacheBuilder.newBuilder()
        .maximumSize(1000)
        .build(); // look Ma, no CacheLoader
...
try {
    // If the key wasn't in the "easy to compute" group, we need to
    // do things the hard way.
    cache.get(key, new Callable<Key, Graph>() {
        @Override
        public Value call() throws AnyException {
            return doThingsTheHardWay(key);
        }
    });
} catch (ExecutionException e) {
    throw new OtherException(e.getCause());
}
```

##### 显示插入
使用 cache.put(key, value) 方法可以直接向缓存中插入值, 这会直接覆盖掉给定键之前映射的值; 使用 Cache.asMap() 视图提供的任何方法也能修改缓存; 但请注意, asMap 视图的任何方法都不能保证缓存项被原子地加载到缓存中; 即 asMap 视图的原子运算在 Guava Cache 的原子加载范畴之外, 所以相比于 `Cache.asMap().putIfAbsent(K, V)` 方法, `Cache.get(K, Callable<V>)` 方法应该总是优先使用

#### 缓存回收
Guava Cache 提供了三种基本的缓存回收方式
- 基于容量回收
- 定时回收
- 基于引用回收

##### 基于容量回收 (size-based eviction)
如果要规定缓存项的数目不超过固定值, 只需使用 `CacheBuilder.maximumSize(long)`; 缓存将尝试回收最近没有使用或总体上很少使用的缓存项 (在缓存项的数目达到限定值之前, 缓存就可能进行回收操作, 通常来说这种情况发生在缓存项的数目逼近限定值时)  
另外, 不同的缓存项可以有不同的权重, 如果缓存值完全不占据内存空间, 可以使用 `CacgeBuilder.weight(Weighter)` 指定一个权重函数, 并且使用 `CacheBuilder.maximumWeighe(long)` 指定最大总重; 在权重限定场景中, 除了要注意回收也是在重量逼近限定值时就进行了, 还要知道重量是在缓存创建时计算的, 因此要考虑重量计算的复杂度
```Java
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
        .maximumWeight(100000)
        .weigher(new Weigher<Key, Graph>() {
            public int weigh(Key k, Graph g) {
                return g.vertices().size();
            }
        })
        .build(
            new CacheLoader<Key, Graph>() {
                public Graph load(Key key) { // no checked exception
                    return createExpensiveGraph(key);
                }
            });
```

##### 定时回收 (Timed Eviction)
CacheBuilder 提供两种定时回收的方法
- expireAfterAccess(long, TimeUnit): 缓存项在给定时间内没有被读/写访问则回收 (请注意这种缓存的回收顺序和基于大小回收一样)
- expireAfterWrite(long, TimeUnit): 缓存项在给定时间内没有被写访问 (创建或覆盖) 则回收; 如果认为缓存数据总是在固定时候后变得陈旧不可用, 这种回收方式是可取的
###### 测试定时回收
对定时回收进行测试时, 不一定非得花费两秒钟去测试两秒的过期; 可以使用 `Ticker` 接口和 `CacheBuilder.ticker(Ticker)` 方法在缓存中自定义一个时间源, 而不是非得用系统时钟

##### 基于引用的回收 (Reference-based Eviction)
通过使用弱引用的键, 或弱引用的值, 或软引用的值, Guava Cache 可以把缓存设置为允许垃圾回收
- CacheBuilder.weakKeys(): 使用弱引用存储键; 当键没有其它 (强或软) 引用时, 缓存项可以被垃圾回收; 因为垃圾回收仅依赖恒等式 (==)，使用弱引用键的缓存用 `==` 而不是 `equals` 比较键
- CacheBuilder.weakValues(): 使用弱引用存储值; 当值没有其它 (强或软) 引用时, 缓存项可以被垃圾回收; 因为垃圾回收仅依赖恒等式 (==), 使用弱引用值的缓存用 `==` 而不是 `equals` 比较值
- CacheBuilder.softValues(): 使用软引用存储值; 软引用只有在响应内存需要时, 按照全局最近最少使用的顺序回收; 考虑到使用软引用的性能影响, 通常建议使用更有性能预测性的缓存大小限定 (基于容量回收); 使用软引用值的缓存同样用 `==` 而不是 `equals` 比较值

##### 显示清除
任何时候都可以显式地清除缓存项, 而不是等到它被回收
- 单个清除: Cache.invalidate(key)
- 批量清除: Cache.invalidateAll(keys)
- 所有清除: Cache.invalidateAll()

##### 移除监听器
通过 `CacheBuilder.removalListener(RemovalListener)`, 可以声明一个监听器, 以便缓存项被移除时做一些额外操作; 缓存项被移除时, `RemovalListener` 会获取移除通知 (RemovalNotification), 其中包含移除原因 (RemovalCause), 键, 值; 需要注意的是, `RemovalListener` 抛出的任何异常都会在记录到日志后被丢弃
```Java
CacheLoader<Key, DatabaseConnection> loader = new CacheLoader<Key, DatabaseConnection> () {
    public DatabaseConnection load(Key key) throws Exception {
        return openConnection(key);
    }
};
RemovalListener<Key, DatabaseConnection> removalListener = new RemovalListener<Key, DatabaseConnection>() {
    public void onRemoval(RemovalNotification<Key, DatabaseConnection> removal) {
        DatabaseConnection conn = removal.getValue();
        conn.close(); // tear down properly
    }
};
return CacheBuilder.newBuilder()
    .expireAfterWrite(2, TimeUnit.MINUTES)
    .removalListener(removalListener)
    .build(loader);
```
默认情况下, 监听器方法是在移除缓存时同步调用的; 因为缓存的维护和请求响应通常是同时进行的, 代价高昂的监听器方法在同步模式下会拖慢正常的缓存请求; 在这种情况下, 可以使用 `RemovalListeners.asynchronous(RemovalListener, Executor)` 把监听器装饰为异步操作

##### 清理何时发生
使用 `CacheBuilder` 构建的缓存不会 "自动" 执行清理和回收工作, 也不会在某个缓存项过期后马上清理, 也没有诸如此类的清理机制; 相反, 它会在写操作时顺带做少量的维护工作, 或者偶尔在读操作时做 (如果写操作实在太少的话)  
这样做的原因在于: 如果要自动地持续清理缓存, 就必须有一个线程, 这个线程会和用户操作竞争共享锁; 此外某些环境下线程创建可能受限制, 这样 `CacheBuilder` 就不可用了; 如果缓存是高吞吐的，那就无需担心缓存的维护和清理等工作; 如果缓存只会偶尔有写操作, 而又不想清理工作阻碍了读操作, 那么可以创建自己的维护线程, 以固定的时间间隔调用 `Cache.cleanUp()`

##### 刷新
刷新和回收不太一样; 正如 `LoadingCache.refresh(K)` 所声明, 刷新表示为键加载新值, 这个过程可以是异步的; 在刷新操作进行时, 缓存仍然可以向其他线程返回旧值; 而不像回收操作, 读缓存的线程必须等待新值加载完成
```Java
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
        .maximumSize(1000)
        .refreshAfterWrite(1, TimeUnit.MINUTES)
        .build(
            new CacheLoader<Key, Graph>() {
                public Graph load(Key key) { // no checked exception
                    return getGraphFromDatabase(key);
                }
                public ListenableFuture<Key, Graph> reload(final Key key, Graph prevGraph) {
                    if (neverNeedsRefresh(key)) {
                        return Futures.immediateFuture(prevGraph);
                    }else{
                        // asynchronous!
                        ListenableFutureTask<Key, Graph> task=ListenableFutureTask.create(new Callable<Key, Graph>() {
                            public Graph call() {
                                return getGraphFromDatabase(key);
                            }
                        });
                        executor.execute(task);
                        return task;
                    }
                }
            });
```
`CacheBuilder.refreshAfterWrite(long, TimeUnit)` 可以为缓存增加自动定时刷新功能; 和 expireAfterWrite 相反, `refreshAfterWrite` 通过定时刷新可以让缓存项保持可用; 但请注意: 缓存项只有在被检索时才会真正刷新 (如果 `CacheLoader.refresh` 实现为异步, 那么检索不会被刷新拖慢) 因此, 如果在缓存上同时声明 `expireAfterWrite` 和 `refreshAfterWrite`, 缓存并不会因为刷新盲目地定时重置, 如果缓存项没有被检索, 那刷新就不会真的发生, 缓存项在过期时间后也变得可以回收

#### 其他特性
##### 统计
`CacheBuilder.recordStats()` 用来开启 Guava Cache 的统计功能; 统计打开后, `Cache.stats()` 方法会返回 `CacheStats` 对象以提供很多统计信息, 举例如下
- hitRate(): 缓存命中率
- averageLoadPenalty(): 加载新值的平均时间, 单位为乃秒
- evivtCount(): 缓存项被回收的总数, 不包括显示清除

##### asMap 视图
asMap 视图提供了缓存的 `ConcurrentMap` 形式, 但 asMap 视图与缓存的交互需要注意
- cache.asMap() 包含当前所有加载到缓存的项; 因此相应地 cache.asMap().keySet() 包含当前所有已加载键
- asMap().get(key) 实质上等同于 `cache.getIfPresent(key)`, 而且不会引起缓存项的加载
- 所有读写操作都会重置相关缓存项的访问时间, 包括 `Cache.asMap().get(Object)` 和 `Cache.asMap().put(K, V)`方法; 但其他 asMap 视图的方法不会重置缓存项时间

##### 中断
TODO

>**参考:**
[Caches](https://github.com/google/guava/wiki/GraphsExplained)  
[缓存](http://ifeve.com/google-guava-cachesexplained/)
