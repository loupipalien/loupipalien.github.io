---
layout: post
title: "Jdk 中的 SPI 服务发现机制"
date: "2018-11-27"
description: "Jdk 中的 SPI 服务发现机制"
tag: [java]
---

### 什么是 SPI
SPI 全称是 Serivce Provider Interface, 是 JDK 内置的一种服务发现机制; SPI 是一种动态替换的发现机制, 主要是被框架的开发人员使, 比如 java.sql.Driver 接口, 其他不同厂商可以针对同一接口做出不同的实现, mysql 和 postgresql 都有不同的实现提供给用户, 而 Java 的 SPI 机制可以为接口寻找服务实现

#### 实现机制
当服务的提供者提供了一种接口的实现之后, 需要在 classpath 下的 META-INF/services/ 目录里创建一个以服务接口命名的文件, 这个文件里的内容就是这个接口的具体的实现类; 当其他的程序需要这个服务的时候, 就可以通过查找这个 jar 包（一般都是以 jar 包做依赖）的 META-INF/services/ 中的配置文件, 配置文件中有接口的具体实现类名, 可以根据这个类名进行加载实例化, 就可以使用该服务了; JDK中查找服务的实现的工具类是: java.util.ServiceLoader

### SPI 实例
在 JDBC 4.0 之前, 连接数据库时通常需要先使用 `Class.forname("com.mysql.jdbc.Driver")` 将数据库驱动类加载到 JVM, 在 JDBC 4.0 之后不再需要显式加载了, 因为 `java.sql.DriverManager` 类会使用 SPI 机制加载各个实现; 可以查看依赖数据库连接包中 META-INF/services/ 中的配置文件查看实现类; 获取连接时使用的 `Connection conn = DriverManager.getConnection(url,username,password);` 时, DriverManager 类中的静态代码块调用了静态方法 `loadInitialDrivers()` 做了加载工作

#### loadInitialDrivers() 方法
```
private static void loadInitialDrivers() {
    String drivers;
    try {
        drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
            public String run() {
                return System.getProperty("jdbc.drivers");
            }
        });
    } catch (Exception ex) {
        drivers = null;
    }
    // If the driver is packaged as a Service Provider, load it.
    // Get all the drivers through the classloader
    // exposed as a java.sql.Driver.class service.
    // ServiceLoader.load() replaces the sun.misc.Providers()

    AccessController.doPrivileged(new PrivilegedAction<Void>() {
        public Void run() {

            ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);  
            Iterator<Driver> driversIterator = loadedDrivers.iterator();             

            /* Load these drivers, so that they can be instantiated.
             * It may be the case that the driver class may not be there
             * i.e. there may be a packaged driver with the service class
             * as implementation of java.sql.Driver but the actual class
             * may be missing. In that case a java.util.ServiceConfigurationError
             * will be thrown at runtime by the VM trying to locate
             * and load the service.
             *
             * Adding a try catch block to catch those runtime errors
             * if driver not available in classpath but it's
             * packaged as service and that service is there in classpath.
             */
            try{
                while(driversIterator.hasNext()) {          
                    driversIterator.next();                 
                }
            } catch(Throwable t) {
            // Do nothing
            }
            return null;
        }
    });

    println("DriverManager.initialize: jdbc.drivers = " + drivers);

    if (drivers == null || drivers.equals("")) {
        return;
    }
    String[] driversList = drivers.split(":");
    println("number of Drivers:" + driversList.length);
    for (String aDriver : driversList) {
        try {
            println("DriverManager.Initialize: loading " + aDriver);
            Class.forName(aDriver, true,
                    ClassLoader.getSystemClassLoader());
        } catch (Exception ex) {
            println("DriverManager.Initialize: load failed: " + ex);
        }
    }
}
```
以上代码的主要步骤
- 从系统变量中获取驱动的信息
- 使用 SPI 获取驱动实现
- 变量使用 SPI 获取到的具体实现并实例化
- 根据第一步获取到的驱动列表实例化具体实现类

在实际使用中主要还是从 SPI 获取具体实现类, `ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);` 这一步主要是构造一个 ServiceLoader 实例, 其中构造了换一个懒加载迭代器; `Iterator<Driver> driversIterator = loadedDrivers.iterator();` 即获取这个迭代器; `driversIterator.hasNext()` 会调用 hasNextService() 方法, 其中使用 `configs = ClassLoader.getSystemResources(fullName);` 约定的配置具体实现类名的文件, 然后解析文件获取实现类名, `driversIterator.next();` 会调用迭代器中的 nextService() 方法实例化实现类

### SPI 流程
- 有关组织和公式定义接口标准
- 第三方提供具体实现: 实现具体方法, 配置 META-INF/services/${interface_name} 文件
- 开发者使用

### SPI 的缺点
不能按需加载, 例如工程中引用了的 mysql, postgresql 等 jar, 但实际只使用了 mysql, ServiceLoader 还是会去加载所有实现了 `java.sql.Driver` 接口的实现类, 即使一些类并不会用到, 以及 Dubbo 文档中提到的加载不到实现类时抛出并不是真正原因的异常; Dubbo 也重新写了一套自己的动态加载扩展实现 SPI 的功能

>**参考:**
[Java中SPI机制深入及源码解析](https://cxis.me/2017/04/17/Java%E4%B8%ADSPI%E6%9C%BA%E5%88%B6%E6%B7%B1%E5%85%A5%E5%8F%8A%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/)  
[Java SPI机制详解](https://juejin.im/post/5af952fdf265da0b9e652de3)  
[SPI Loading](https://dubbo.incubator.apache.org/en-us/docs/dev/SPI.html)
