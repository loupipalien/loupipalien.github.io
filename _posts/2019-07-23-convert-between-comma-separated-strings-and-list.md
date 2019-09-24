---
layout: post
title: "逗号分隔字符串和列表之间的转换"
date: "2019-07-23"
description: "逗号分隔字符串和列表之间的转换"
tag: [java]
---

### 逗号分隔字符串和列表之间的转换

#### 逗号分隔符字符串转为列表
- 利用 JDK 的 Arrays 类
```
String str = "a,b,c";
List<String> list = Arrays.asList(str.split(","))
```
要注意的是, 返回的 list 是 Arrays 类中的 ArrayList 实现

- 利用 Guava 的 Splitter 类
需要引入 Guava 包的依赖
```
String str = "a,b,c";
List<String> list = Splitter.on(",").trimResults().splitToList(str);
```
注意的是, 返回的 list 是不可修改的列表

#### 列表转换为逗号分隔字符串
- 利用 JDK 的 String 类
```
List<String> list = new ArrayList() {{
   add("a");
   add("b");
   add("c");
}};
String str = String.join(",", list);
```
String.join() 方法实际上是借助 StringJoiner 类实现的

- 利用 JDK 的 Stream 类和 Collectors 类
```
List<String> list = new ArrayList() {{
    add("a");
    add("b");
    add("c");
}};
String str = list.stream().collect(Collectors.joining(","));
```
实际上也是借助 StringJoiner 类实现的

- 利用 Guava 的 Joiner 类
```
List<String> list = new ArrayList() {{
    add("a");
    add("b");
    add("c");
}};
String str = Joiner.on(",").join(list);
```

- 利用 Spring Framework 的 StringUtils 类
```
List<String> list = new ArrayList() {{
    add("a");
    add("b");
    add("c");
}};
String str = StringUtils.collectionToDelimitedString(list, ",");
```
StringUtils.collectionToDelimitedString() 方法实际上是借助 StringBuilder 实现的

>**参考:**  
[如何相互转换逗号分隔的字符串和List](https://blog.csdn.net/jicahoo/article/details/44105109)
[Java8-如何将List转变为逗号分隔的字符串](https://blog.csdn.net/benjaminlee1/article/details/72860845)
