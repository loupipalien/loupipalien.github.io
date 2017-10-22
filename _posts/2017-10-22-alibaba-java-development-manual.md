---
layout: post
title: "阿里巴巴java开发手册"
date: "2017-10-22"
description: "阿里巴巴java开发手册"
tag: [java,p3c]
---

10月14日上午9:00 阿里巴巴于在杭州云栖大会上, 正式发布<<阿里巴巴Java开发手册>>扫描插件, 该插件在扫描代码后, 将不符合约定的代码按Blocker/Critical/Major三个等级显示在下方。  

### 开发手册
以下是结合自己实际编程经验所认可或推荐的规约, 仍有部分不懂的或没用到的地方, 后续逐步更新  
`标记`: 暂时未遇到相似场景或不知如何处理比较好
#### **1 编程规约**
#### 1.1 命名风格
- [强制] 代码中命名均不能以下划线或美元符号开始或结束
- [强制] 代码中命名严禁使用拼音, 一律使用英文
- [强制] 类名使用UpperCamelCase风格, `通用简写`
- [强制] 方法名, 参数名, 成员变量, 局部变量统一使用lowerCamelCase风格
- [强制] 常量命名全大写, 单词间用下划线间隔, 力求语义完整, 不要嫌命名长
- [强制] 抽象类命名使用Abstract开头; 异常类命名使用Exception结尾; 测试类以要测试的类名称开始, 以Test结尾
- [强制] 中括号是数组类型的一部分, 数组定义如下: String[] args;
- [强制] POJO类中的布尔类型变量都不以is开头, 否则部分框架解析会引起序列化错误
- [强制] 包名统一使用小写, 点分隔符之间有且仅有一个自然语义的英文单词; 包名统一使用单数形式, 类名如果有复数含义可用复数形式
- [强制] 杜绝完全不规范和不通用缩写, 不要嫌命名长, 避免望文不知义
- [推荐] 为使代码有自解释性, 命名时尽量使用完整单词组合
- [推荐] 接口类的方法和属性不要加任何修饰符号, 只加上有效注释保持代码简洁;`接口中定义变量`
- 接口和实现类的命名规则
  - [强制] 对于Service和Dao接口, 实现类用Impl后缀区别
  - [推荐] 形如能力的接口取对应的形容词做接口名(通常是-able形式)
- [参考] 枚举类名建议带上Enum后缀, 枚举成员名称全部大写, 多个单词组成则用下划线隔开
- [参考] 各层命名约定
  - Service/Dao层方法命名
    - 获取单个对象方法用get做前缀, 后接Object核心名词; 例如: 对象ExportFlow, getFlow
    - 获取多个对象方法用getObjects做前缀; 例如: 对象ExportFlow, getFlows
    - 获取统计值方法用getCount做前缀;例如: getCount, getCountBy..., getCountOf...
    - 插入的方法用add/insert做前缀
    - 删除的方法用remove/delete做前缀
    - 修改的方法用update做前缀
  - `领域模型命名 (暂时领域模型未使用分层, 各层使用统一的bean)`

#### 1.2 常量定义
- [强制] 不允许任何魔法值直接出现在代码中
- [强制] Long或long初始赋值时, 使用大写的L, 不要写小写的l, 容易和1混淆
- [推荐] 不要使用一个常量类维护所有常量, 建议按照常量功能归类分开维护
- [推荐] 常量复用层可分五层
  - 跨应用共享常量: 放置在二方库中, 通常时client.jar, api.jar的constant目录下
  - `应用内共享常量`: 放置在一方库中, 通常是mudules的constant目录下
  - 子工程内部共享变量: 放在当前子工程的constant目录下
  - `包内共享变量`: 当前包下的constant目录下
  - 类内共享变量: 直接在类内部private static final定义
- [推荐] 如果一个变量值仅在一个范围内变化, 且带有名称之外的延伸属性, 定义为枚举类

### 插件使用
- Eclipse ([详细使用说明](https://github.com/alibaba/p3c/blob/master/eclipse-plugin/README_cn.md))  
安装: Help >> Install New Software >> 输入Update Site地址: https://p3c.alibaba.com/plugin/eclipse/update 回车, 然后勾选Ali-CodeAnalysis, 再一直点Next Next...按提示走下去就好, 然后就是提示重启了,安装完毕  
更新: 可以通过 Help >> Check for Udates 进行新版本检测  

>**参考:**  
[p3c](https://github.com/alibaba/p3c)
