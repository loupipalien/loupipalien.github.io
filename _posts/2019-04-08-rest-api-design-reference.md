---
layout: post
title: "Rest Api 设计参考"
date: "2019-04-08"
description: "Rest Api 设计参考"
tag: [rest]
---

### REST 是什么
REST 即 Representational State Transfer 的缩写, 这个词组的翻译是 "表现层的状态转化"; 其实这个词组省略了主语 --- 资源
- 资源 (Resources)
对象的单个实例; 例如: 一只动物, 一段文本, 一张图片, 一首歌, 一种服务等等, 总之就是一个具体的实在; 通常使用一个 URI (统一资源定位符) 指向它
- 表现层 (Representation)
把资源呈现出来的形式; 资源是一个具体的实体, 它可以有多种外在的表现方式; 例如: 文本可以用 txt, html, json, xml 等格式表示, 图片可以用 jpg, png 等格式展现
- 状态转化 (State Transfer)
发起一个请求, 代表了客户端和服务器的一个互动过程; 在这个过程中会涉及到数据和状态的变化; 由于 HTTP 协议是一个无状态协议, 这意味着所有的状态都保存在服务器端; 因此, 如果客户端想要通知服务器改变数据和状态的变化, 必须通过某种方式让服务器端发生 "状态转化" (State Transfer); 这种转化是建立在表现层之上的

### RESTful 是什么
RESTful 表示符合 REST 交互方式的一种架构风格
- 每个 URI 代表一种资源
- 客户端和服务器之间, 传递这中资源的某种表现层
- 客户端通过 HTTP 的几种请求方法, 对服务端的资源进行操作, 实现 "表现层状态转化"

### REST API 设计
#### URI 格式规范
- URI 中尽量使用连字符 `-` 来代替下划线 `_` 的使用
- URI 中统一使用小写字母
- URI 中不要包含文件的扩展名

#### 资源的原型
- 文档 (Document): 文档是资源的表现形式, 可以理解为一个对象, 或者数据库中的一条记录
```
https://api.example.com/users/will
https://api.example.com/posts/1
https://api.example.com/posts/1/comments/1
```
- 集合 (Collection): 集合可以理解为是资源的一个容器
```
https://api.example.com/users
https://api.example.com/posts
https://api.example.com/posts/1/comments
```
- 仓库 (Store): 仓库可以理解为集合的一个容器, 可以放置多种资源的集合
```
https://api.example.com/users/1234/favorites/posts/1
```
- 控制器 (Controller): 是为了除了增删改查以外的一些逻辑操作, 控制器 (方法) 一般定义在 URI 末尾, 且不会有子资源 (控制器)
```
https://api.example.com/alters/1234/resend
https://api.example.com/posts/1/pulish
```

#### 请求方法
- GET (R): 从服务器获取一个资源或一些资源
- POST (C): 在服务器上创建一个新的资源
- PUT (U): 更新服务器上的一个资源
- DELETE (D): 删除服务器上的一个资源

另外, HTTP 还有 HEAD 和 OPTIONS 请求方法, 但并不常用

#### URI 命名规范
- 文档 (Document) 类型的资源用名词 (短语) 单数命名
- 集合 (Collection) 类型的资源用名词 (短语) 复数命名
- 仓库 (Store) 类型的资源用名词 (短语) 复数命名
- 控制器 (Controller) 类型的资源用动词 (短语) 命名
- CRUD 的操作不要体现在 URI 中, 首先 URI 代表一个资源, 其次 HTTP 协议中的请求方法已经对 CRUD 做了映射
```
# 正确示例
DELETE /users/1234
# 错误示例
GET /deleteUser?id=1234  
GET /deleteUser/1234  
DELETE /deleteUser/1234  
POST /users/1234/delete
```

**参考:**
[REST接口设计规范总结](http://www.maogx.win/posts/3/)
[restful接口设计规范总结](https://www.jianshu.com/p/8b769356ee67)
