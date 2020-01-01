---
layout: post
title: "Java 中的 ThreadLocal"
date: "2019-10-31"
description: "Java 中的 ThreadLocal"
tag: [java]
---

### Java 中的 ThreadLocal

#### ThreadLocal 是什么
官方文档如下
```
这个类提供了线程本地变量; 这些变量不同于普通变量, 每个线程都可以访问一个, 每个线程有它自己的, 独立初始化的变量副本; ThreadLocal 实例通常是希望状态与线程关联的类中的私有静态字段 (例如 userId 或 TransactionId)  
只要线程是存活的, ThreadLocal 实例是可访问的, 那么每个线程都持有一个它的线程本地变量副本的隐式引用; 在一个线程结束, 它所有的线程本地副本实例都将被垃圾回收 (除非存在这些副本的引用)
```
`ThreadLocal` 适用于每个线程需要自己独立的实例, 且线程如何创建不被开发者所掌控时 (如果开发者可以掌控线程的创建, 则可以为线程设置字段来代替 `ThreadLocal` 达到相同的目的; 有时使用 ThreadLocal 实现更加优雅)

#### ThreadLocal 使用示例
```Java
public class ThreadLocalDemo {
    /**
     * 线程次序: 记录当前线程是第几个使用此 ThreadLocalDemo 实例的
     */
    private static final ThreadLocal<Integer> THREAD_ORDER = new ThreadLocal<Integer>() {
        private AtomicInteger integer = new AtomicInteger();
        @Override
        protected Integer initialValue() {
            return integer.getAndIncrement();
        }
    };

    public void  printThreadOrder() {
        System.out.println(Thread.currentThread().getName() + ": " + THREAD_ORDER.get());
    }

    public static void main(String[] args) {
        ThreadLocalDemo demo = new ThreadLocalDemo();
        for (int i = 0; i < 10; i++) {
            new Thread(() -> demo.printThreadOrder()).start();
        }
    }
}
```

#### ThreadLocal 原理
##### ThreadLocal 维护 Thread 与线程本地变量副本的映射
既然每个访问 `ThreadLocal` 的线程都有自己的线程本地变量副本, 一个可能实现方案是 `ThreadLocal` 维护一个 Map, 见是 Thread, 值是该线程对应的线程本地变量副本; 此方案可以满足每个线程有独立的线程本地变量副本的要求, 每个新线程访问 `ThreadLocal` 时需要向 Map 中添加一个映射, 每个线程结束时移除映射  
![ThreadLocal维护映射.png](https://i.loli.net/2019/10/31/Hb1Gg6Uj4nLhcow.png)  
但这个方案需要注意以下问题
- 增加映射和减少映射都需要写 Map, 多个线程则需要 Map 是线程安全的; 无论采用何种方式保证线程安全, 都有锁开销
- 线程结束后, 需要保证此线程访问的所有 `ThreadLocal` 中对应的映射被删除, 否则会引起内存泄露
##### Thread 维护 ThreadLocal 与线程本地变量副本的映射 (JDK 方案)
上述方案由于有多线程访问同一个 Map, 会有锁开销; 将 Map 由 Thread 维护, 从而使得每个 Thread 只访问自己的 Map, 这样就避开了多线程写 Map 的问题, 也就不再需要 Map 是线程安全的, 也就没有了锁开销  
![Thread维护映射.png](https://i.loli.net/2019/10/31/7hPaDwHtWGfSMkc.png)   
该方案虽然避免了锁的开销的, 但由于线程访问了 `ThreadLocal` 后都会在自己的 Map 内维护 `ThreadLocal` 与线程本地变量副本的映射, 所以线程结束后, 需要保证删除对应的映射, 否则仍有内存泄露问题

#### ThreadLocal 实现 (JDK8)
JDK 中使用 Thread 维护 ThreadLocal 与线程本地变量副本的映射; 存储映射的 Map 是 `ThreadLocal` 类的静态内部类 `ThreadLocalMap`; 与常用的 `HashMap` 不同的是, `ThreadLocalMap` 的每个 Entry 有一个对键的弱引用, 一个对值的强引用
```Java
/**
 * The entries in this hash map extend WeakReference, using
 * its main ref field as the key (which is always a
 * ThreadLocal object).  Note that null keys (i.e. entry.get()
 * == null) mean that the key is no longer referenced, so the
 * entry can be expunged from table.  Such entries are referred to
 * as "stale entries" in the code that follows.
 */
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

TODO

>**参考:**
- [正确理解Thread Local的原理与适用场景](http://www.jasongj.com/java/threadlocal/)
