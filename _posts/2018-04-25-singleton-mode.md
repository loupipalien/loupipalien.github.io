---
layout: post
title: "单例模式"
date: "2018-04-25"
description: "单例模式"
tag: [java, design mode]
---

### 单例模式

#### 懒汉式-线程不安全
```
public class Singleton {
    private static Singleton instance;
    private Singleton (){}
    public static Singleton getInstance() {
     if (instance == null) {
         instance = new Singleton();
     }
     return instance;
    }
}
```
以上代码使用了懒加载模式, 在单线程调用下时可以正常工作的; 当在多线程情景下则会出现致命的问题, 因为多个线程并发可能会导致创建多个实例...

#### 懒汉式-方法级别同步-线程安全
```
public static synchronized Singleton getInstance() {
    if (instance == null) {
        instance = new Singleton();
    }
    return instance;
}
```
为了解决线程不安全的问题, 最简单的办法是将获取实例的方法声明为 synchronized 的, 这样即使在多线程的场景下, 任何时候都只能有一个线程调用此方法; 但是这种方式并不高效, 因为需要同步的操作只有创建实例的语句, 并且创建实例的语句应该只执行一次, 获取实例的语句并不需要同步, 由此引出了双重检验锁

#### 懒汉式-双重检验锁-线程安全
双重检验锁模式 (double checked locking pattern) 是一种使用同步块加锁的方法; 因为会有两次检查 instance == null, 一次是在同步块外, 一次是在同步块内; 在同步块内还要再检验一次是因为可能会有多个线程一起进入同步块外的 if, 如果在同步块内不进行二次检验的话就会生成多个实例了
```
public static Singleton getSingleton() {
    if (instance == null) {                         //Single Checked
        synchronized (Singleton.class) {
            if (instance == null) {                 //Double Checked
                instance = new Singleton();
            }
        }
    }
    return instance ;
}
```
这段代码看起来很完美, 但是它还是有问题... 主要在于 instance = new Singleton() 这并非是一个原子操作...
TODO...

#### 饿汉式-线程安全
将单例的实例被声明为 static 和 final, 在第一次加载类到内存中初始化时生成实例, 所以创建实例本身是线程安全的
```
public class Singleton{
    // 类加载时就初始化
    private static final Singleton instance = new Singleton();

    private Singleton(){}
    public static Singleton getInstance(){
        return instance;
    }
}
```
但是饿汉式的缺点是它不是一种懒加载模式 (lazy initialization), 单例会在加载类后一开始就被初始化, 即使 getInstance()方法不被调用也会实例化; 而且饿汉式的创建方式在一些场景中将无法使用: 譬如 Singleton 实例的创建是依赖参数或者配置文件的, 在 getInstance() 之前必须调用某个方法设置参数, 那样这种单例写法就无法满足要求了

#### 静态内部类
TODO... <<Effective Java>>

#### 枚举

### 总结

>**参考:**
[如何正确地写出单例模式](http://wuchong.me/blog/2014/08/28/how-to-correctly-write-singleton-pattern/)
