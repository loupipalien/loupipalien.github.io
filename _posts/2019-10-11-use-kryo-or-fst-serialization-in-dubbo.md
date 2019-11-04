---
layout: post
title: "在 Dubbo 中使用 Kryo 或 FST 序列化"
date: "2019-10-11"
description: "在 Dubbo 中使用 Kryo 或 FST 序列化"
tag: [dubbo, java]
---

### 在 Dubbo 中使用 Kryo 或 FST 序列化
在使用 Dubbo 时, Dubbo 协议默认使用的序列化为 Hession2, 对应大多数应用都是适用的; 但如果想使用更高效的序列化器, 或者由于其他原因不能使用 Hession2 序列化 (例如 `org.springframework.data.domain.PageImpl` 类)

#### 启用 Kryo 或 FST
启用方式非常简单, 在 Dubbo 提供者的 `dubbo:protocol` 配置中添加 `serialization` 属性, 然后在消费者和提供者两方加入对应的依赖
- Kryo
```
# Dubbo
<dubbo:protocol name="dubbo" serialization="kryo"/>

# Maven
<dependency>
    <groupId>com.esotericsoftware.kryo</groupId>
    <artifactId>kryo</artifactId>
    <version>2.24.0</version>
</dependency>
<dependency>
    <groupId>de.javakaffee</groupId>
    <artifactId>kryo-serializers</artifactId>
    <version>0.26</version>
</dependency>
```
- FST
```
# Dubbo
<dubbo:protocol name="dubbo" serialization="fst"/>

# Maven
<dependency>
    <groupId>de.ruedigermoeller</groupId>
    <artifactId>fst</artifactId>
    <version>1.55</version>
</dependency>
```

#### 注册被序列化类
要让 Kryo 和 FST 完全发挥出高性能, 最好将那些需要被序列化的类注册到 Dubbo 中; 由于注册被序列化的类仅仅是出于性能优化的目的, 所以即使忘记注册某些类也没有关系; 事实上即使不注册任何类, Kryo 和 FST 的性能依然普遍优于 Hessian 和 Dubbo 序列化; 具体操作见参考中的文章  
在使用 Kryo 和 FST 序列化时, 另外有几点需要注意
- 如果被序列化的类中不包含无参的构造函数, 则在 Kryo 的序列化中, 性能将会大打折扣, 因为此时在底层将用 Java 的序列化来透明的取代 Kryo 序列化
- Kryo 和 FST 本来都不需要被序列化都类实现 Serializable 接口, 但还是建议每个被序列化类都去实现它, 因为这样可以保持和 Java 序列化以及 Dubbo 序列化的兼容性

>**参考**:
[Alibaba-在Dubbo中使用高效的Java序列化（Kryo和FST）](http://dubbo.apache.org/zh-cn/docs/user/demos/serialization.html)  
[DangDang-在Dubbo中使用高效的Java序列化（Kryo和FST）](https://dangdangdotcom.github.io/dubbox/serialization.html)  
['org.springframework.data.domain.PageImpl' could not be instantiated](https://blog.csdn.net/weixin_44142296/article/details/87884288)
