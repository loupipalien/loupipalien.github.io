---
layout: post
title: "如何使用 ThreadLocal"
date: "2018-02-27"
description: "如何使用 ThreadLocal"
tag: [java]
---

### 介绍
ThreadLocal 一般称为线程本地变量, 它是一种特殊的线程绑定机制, 将变量与线程绑定在一起，为每一个线程维护一个独立的变量副本 (值或引用), 通过ThreadLocal可以将变量的可见范围限制在同一个线程内, 减少同一个线程内多个函数或者组件之间一些公共变量的传递的复杂度


#### 代码实例
```
public class ThreadLocalDemo {

    public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            final Thread t = new Thread() {
                @Override
                public void run() {
                    System.out.println("First Time: Current Thread:" + Thread.currentThread().getName() + ", Thread Id:" + ThreadId.get());
                    try {
                        Thread.sleep(new Random().nextInt(100));
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println("Second Time: Current Thread:" + Thread.currentThread().getName() + ", Thread Id:" + ThreadId.get());
                }
            };
            t.start();
        }
    }

    static class ThreadId {
        /**
         * 一个递增的序列, 使用 AtomicInger 原子变量保证线程安全
         */
        private static final AtomicInteger nextId = new AtomicInteger(0);
        /**
         * 线程本地变量, 为每个线程关联一个唯一的序号
         */
        private static final ThreadLocal<Integer> threadId =
                new ThreadLocal<Integer>() {
                    @Override
                    protected Integer initialValue() {
                        // 相当于 nextId++ ,由于 nextId++ 这种操作是个复合操作而非原子操作，会有线程安全问题(可能在初始化时就获取到相同的ID，所以使用原子变量
                        return nextId.getAndIncrement();
                    }
                };

        /**
         * 返回当前线程的唯一的序列，如果第一次get, 会先调用initialValue
         * @return
         */
        public static int get() {
            return threadId.get();
        }
    }
}
```
执行结果, 可以看到每个线程都分配到了一个唯一的 ID, 同时在此线程范围内的 "任何地点", 都可以通过 ThreadId.get() 这种方式直接获取, 而且 ID 是线程隔离的

#### 设计思想
- ThreadLocal 仅仅是一个变量的访问入口, 并不存放变量的值
- 每个 Thread 对象都有一个 ThreadLocalMap 对象, 这个 ThreadLocalMap 对象持有 ThreadLocal 的引用
- ThreadLocalMap 以 ThreadLocal 对象为 key, 以真正需要存储对象为 value; get 时通过 ThreadLocal 对象先获取当前线程的 ThreadLocalMap 对象, 再以自己为 key 获取 value

其实 ThreadLocal 可以设计一个形如 Map<Thread, V> 的容器形式, 一个线程对应一个存储对象, 但 ThreadLocal 这样设计的好处是
- 保证当前线程结束时相关对象可以尽快被回收
- 保存存储对象的 Map 容器中的元素个数减少, 降低哈希冲突的可能性并且提高性能

#### ThreadLocal 的主要方法
- get()
获取当前线程, 并利用 ThreadLocal 对象获取值; 如果当前线程的 ThreadLocalMap 还未初始化则调用 setInitialValue() 方法
```
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```
- setInitialValue()
调用 initialValue() 获取初始化值设置并返回
```
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```
- initialValue()
此方法是 protected 的, 可由用户重写实现自己的逻辑
```
protected T initialValue() {
    return null;
}
```
- set(T value)
可以直接设置当前线程绑定的 ThreadLocal 对象的值
```
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

#### TODO
- 线程独享变量
- ThreadLocal 散列值
- ThreadLocalMap是使用ThreadLocal的弱引用作为Key

>**参考:**
[谈谈Java中的ThreadLocal](https://www.cnblogs.com/chengxiao/p/6152824.html)
[详解 ThreadLocal](https://www.cnblogs.com/zhangjk1993/archive/2017/03/29/6641745.html)  
[深入剖析ThreadLocal](https://www.cnblogs.com/ysw-go/p/5944837.html)
