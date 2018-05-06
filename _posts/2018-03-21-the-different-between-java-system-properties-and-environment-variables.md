---
layout: post
title: "Jave 系统属性和环境变量的区别"
date: "2018-03-21"
description: "Jave 系统属性和环境变量的区别"
tag: [java]
---

### System Properties
可以使用 Java 命令行 `-Dpropertyname=value` 语法设置; 可以使用 `System.setProperty(String key, String value)` 或通过 `System.getProperties().load()` 的各种方法在运行时添加

### Environment Variables
在操作系统中设置, 例如在 Linux 中 `export HOME=/Users/myusername` 或在 Windows 上 `SET WINDIR=C:\Windows` 等, 不像 properties 可以在运行时设置; 可以使用 `System.getenv(String naem)` 获取一个指定的环境变量

>**参考:**
[Java system properties and environment variables](https://stackoverflow.com/questions/7054972/java-system-properties-and-environment-variables)
