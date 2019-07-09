---
layout: post
title: "Class.forname() vs ClassLoader.loadClass()"
date: "2018-12-29"
description: "Class.forname() vs ClassLoader.loadClass()"
tag: [java]
---

### 动态加载类
在程序中常常需要动态加载类文件, 在 JDK 中提供了两个类支持动态加载, `java.lang.Class` 和 `java.lang.ClassLoader`
```
Class<?> clazz = Class.forname(className);
Class<?> clazz = Class.forname(className, true, classLoader);
...
Class<?> clazz = ClassLoader.loadClass(className);
Class<?> clazz = ClassLoader.loadClass(className, true);
```
虽然这两种方式都可以实现类加载, 但也是有细微差别的

### 类的加载过程
类从被加载到虚拟机内存中, 到卸载出内存为止, 它的整个生命周期包括: 加载 (Loading), 验证 (Verification), 准备 (Preparation), 解析 (Resolution), 初始化 (Initialization), 使用 (Using), 卸载 (Unloading) 7 个阶段; 其中验证, 准备, 解析 3 个部分统称为连接 (Linking); 类的加载阶段则是从加载到初始化这五个步骤

- 加载: 查找和导入类或接口的二进制数据
- 校验: 检查导入类或接口的二进制数据的正确性
- 准备: 给类的静态变量分配并初始化存储空间 (静态变量这一步仅是初始缺省值, 常量会直接赋值)
- 解析: 将符号引用转成直接引用；
- 初始化: 激活类的静态变量的初始化代码和静态代码块

#### Class.forname()
```
public static Class<?> forName(String className) throws ClassNotFoundException {
    ...
    return forName0(className, true, ClassLoader.getClassLoader(caller), caller);
}
...
public static Class<?> forName(String name, boolean initialize, ClassLoader loader) throws ClassNotFoundException {
    ...
    return forName0(name, initialize, loader, caller);
}
```
这里有一个 `initialize` 参数, 即表示是否需要完成类初始化; 使用 `Class.forname(className)` 的方法会默认完成类的初始化再返回类的 `Class` 实例; (另外 `Class.forName(String name, boolean initialize, ClassLoader loader)` 中的 classLoader 传递 null 表示使用启动 `Bootstrap ClassLoader` 加载)

#### ClassLoader.loadClass()
```
public Class<?> loadClass(String name) throws ClassNotFoundException {
    return loadClass(name, false);
}
...
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
    ...
    if (resolve) {
        resolveClass(c);
    }
    return c;
}
```
这里有一个 `resolve` 参数, 即表示是否需要完成类解析; 使用 `ClassLoader.loadClass(className)` 的方法默认不会执行类解析, 更不会执行类的初始化

#### 示例
```
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    public Driver() throws SQLException {}

    static {
        try {
            DriverManager.registerDriver(new Driver());
        } catch (SQLException var1) {
            throw new RuntimeException("Can't register driver!");
        }
    }
}
```
在 JDBC 4.0 之前连接 mysql, 会写 `Class.forname("com.mysql.jdbc.Driver")`, `Driver` 类在静态代码块注册到 `DriverManager`, 如果使用 `ClassLoader.loadClass("com.mysql.jdbc.Driver")` 则不会执行类的初始化, 静态代码块也不会执行, 在 `DriverManager.getConnection()` 时会报错

>**参考:**
[深入理解Java虚拟机 (第2版)](https://book.douban.com/subject/24722612/)  
[Class.forName() vs ClassLoader.loadClass()](https://stackoverflow.com/questions/8100376/class-forname-vs-classloader-loadclass-which-to-use-for-dynamic-loading)  
[ClassLoader.loadClass 和Class.forName的区别](https://www.jianshu.com/p/50e18a563301)  
