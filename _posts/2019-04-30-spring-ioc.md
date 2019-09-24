---
layout: post
title: "Spring IOC"
date: "2019-04-30"
description: "Spring IOC"
tag: [spring, java]
---

### Spring IOC 实现原理
反射

#### 装配 Bean 的方式
XML 配置和注解配置

#### Spring 依赖注入
- Setter 注入
- 构造函数注入 (注意循环依赖问题)
- 注解注入 (自动装配, `<context:component-scan/> 和 <context:annotation-config/>`)
  - @Autowired (默认按类型匹配)
  - @Resource (默认按名称匹配)
  - @Value (读取 properties 文件, 占位符方式和 SpEL 方式)

#### IOC 容器管理 Bean

##### Bean 的命名
TODO

##### Bean 实例化方法
- 默认构造函数方法
- 实例工厂方法
- 静态工厂方法

Bean 的重写机制, 即重复便覆盖机制

##### Bean 的作用域
- singleton 作用域
- prototype 作用域
- request 作用域
- session 作用域

另外, Bean 的懒加载

#### IOC 与依赖注入的区别
- IOC (控制反转) 即将对象的创建权由 Spring 管理
- DI (依赖注入) 即在 Spring 创建对象的过程中, 把对象依赖的属性注入到类中

>**参考:**
[关于Spring IOC (DI-依赖注入)你需要知道的一切](https://blog.csdn.net/javazejian/article/details/54561302)
