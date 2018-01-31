---
layout: post
title: "Commons CLI 快速开始"
date: "2018-01-31"
description: "Commons CLI 入门"
tag: [commons-cli]
---

### 介绍
命令行处理中有三个阶段, 分别是定义, 解析, 询问; 接下来的章节将会依次讨论, 并且讨论如何用 CLI 实现

### 定义阶段
每个命令行必须定义用于定义应用的接口的选项集;CLI 用 `Options` 类作为 `Option` 实例的容器; 在 CLI 中有两种方式创建 `Options`, 一种是通过构造器, 另一种是通过定义在 `Options` 类中的工厂方法; 使用场景文档中会提供如何创建一个 `Options` 对象的例子, 并提供真实的实例; 定义阶段的结果是返回一个 `Options` 实例

### 解析阶段
解析阶段中通过处理命令行将文本传递到应用中, 文本根据解析器实现定义的规则处理; `parse` 方法定义在 `CommmangLineParser` 中, 使用一个 `Options` 实例和一个 `String[]` 作为参数并返回一个 `CommandLine`; 解析阶段的结果是返回一个 `CommandLine` 实例

### 询问阶段
询问阶段是应用查询 `CommandLine` 依据布尔选项和提供给应用的选项值决定执行哪个分支; 这个阶段的实现在用户代码中, `CommandLine` 中的访问器方法给用户代码提供了询问的功能; 询问阶段的结果是用户代码了解命令行上提供的并根据解析器和 `Options` 规则处理的文本

>**参考:**
[Getting started](http://commons.apache.org/proper/commons-cli/introduction.html)
