---
layout: post
title: "Commons CLI 使用场景"
date: "2018-02-01"
description: "Commons CLI 使用场景"
tag: [commons-cli]
---

接下来章节中会在一些实例中讲述如何在应用中使用 CLI

### 使用布尔选项
布尔选项在命令行中以其选择的存在表示, 即如果找到该选项则表示选项为 `true`, 否则值为 `false`; `DateApp` 工具类可以打印当前日期到标准输出中, 如果 `-t` 选项存在则当前时间也被打印

### 创建 Options
必须创建一个 `Options` 对象, 并且 `Option` 必须添加到其中去
```
// create Options object
Options options = new Options();

// add t option
options.addOption("t", false, "display current time");
```
`addOption` 方法有三个参数, 第一个参数是一个 `java.lang.String` 类型, 用于表示这个选项; 第二个参数是一个 `boolean`类型, 用于表示这个参数是否需要一个参数值, 在布尔选项 (有时被引用为一个标记) 中, 参数值不存在所以传递一个 `false`; 第三个参数是这个选项的描述, 描述可以被用于应用的使用说明

### 解析命令行参数
