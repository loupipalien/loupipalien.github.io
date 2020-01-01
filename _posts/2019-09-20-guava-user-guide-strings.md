---
layout: post
title: "Guava 用户指南之字符串"
date: "2019-09-20"
description: "Guava 用户指南之字符串"
tag: [java, guava]
---

### 字符串

#### 连接器
用分隔符把字符串序列连接起来可能会遇上不必要的麻烦, 例如字符串序列中含有 null; 流式风格的 Joiner 让连接字符串更简单
```Java
Joiner joiner = Joiner.on("; ").skipNulls();
return joiner.join("Harry", null, "Ron", "Hermione");
```
上述代码返回 `Harry; Ron; Hermione`; 另外 `useForNull(String)` 方法可以给定某个字符串来替换 `null`, 而不像 `skipNulls()` 方法是直接忽略 `null`; Joiner 也可以用来连接对象类型, 在这种情况下, 它会把对象的 `toString()` 值连接起来
```java
Joiner.on(",").join(Arrays.asList(1, 5, 7)); // returns "1,5,7"
```
>joiner 实例总是不可变的; 用来定义 joiner 目标语义的配置方法总会返回一个新的 joiner 实例; 这使得 joiner 实例都是线程安全的, 可以将其定义为 static final 常量

#### 拆分器
JDK 内建的字符串拆分工具有一些古怪的特性; String.split 会自动丢弃尾部的分隔符, 如 `",a,,b,".split(",")` 将会返回 `"", "a", "", "b"`;  Splitter 使用流式 API 对这些特性做了完全的掌控
```Java
Splitter.on(',')
        .trimResults()
        .omitEmptyStrings()
        .split("foo,bar,,   qux");
```
上述代码返回 `Iterable<String>`, 其中包含 `"foo", "bar", "qux"`; Splitter可以被设置为按照任何模式, 字符, 字符串或字符匹配器拆分  

##### 拆分器工厂

| 方法 | 描述 | 范例 |
| :--- | :--- | :--- |
| Splitter.on(char) | 按单个字符拆分 | Splitter.on(";") |
| Splitter.on(CharMatcher) | 按字符匹配器拆分 | Splitter.on(CharMatcher.BREAKING_WHITESPACE) |
| Splitter.on(String) | 按字符串拆分 | Splitter.on(", ") |
| Splitter.on(Pattern), Splitter.onPattern(String) | 按字符串拆分 | Splitter.onPattern("\r?\n") |
| Splitter.fixedLength(int) | 按固定长度拆分; 最后一段可能比给定的长度短, 但不会为空 | Splitter.fixedLength(3) |

##### 拆分器修饰符

| 方法 | 描述 |
| :--- | :--- |
| omitEmptyStrings() | 从结果中自动忽略空白字符串 |
| trimResults() | 移除结果字符串的前导空白和尾部空白 |
| trimResults(CharMatcher) | 给定匹配器, 移除结果字符串的前导空白和尾部空白 |
| limit() | 限制拆分出的字符串数量 |

如果想要拆分器返回 List, 只要使用 `Lists.newArrayList(splitter.split(string))` 或类似方法
>splitter 实例总是不可变的; 用来定义 splitter 目标语义的配置方法总会返回一个新的 splitter 实例; 这使得 splitter 实例都是线程安全的, 可以将其定义为 static final 常量

#### 字符匹配器 [CharMatcher]
TODO

##### 字符集 [Charsets]
`Charsets` 针对所有 Java 平台都要保证支持的六种字符集提供了常量引用, 尝试使用这些常量, 而不是通过名称获取字符集实例
```Java
# not recommend
try {
    bytes = string.getBytes("UTF-8");
} catch (UnsupportedEncodingException e) {
    // how can this possibly happen?
    throw new AssertionError(e);
}

# recommend
bytes = string.getBytes(Charsets.UTF_8);
```

#### 大小写格式 [CaseFormat]
`CaseFormat` 被用来方便地在各种 ASCII 大小写规范间转换字符串, 比如编程语言的命名规范; `CaseFormat` 支持的格式如下

| 格式 | 范例 |
| :--- | :--- |
| LOWER_CAMEL | lowerCamel |
| LOWER_HYPHEN | lower-hyphen |
| LOWER_UNDERSCORE | lower_underscore |
| UPPER_CAMEL | UpperCamel |
| UPPER_UNDERSCORE | UPPER_UNDERSCORE |

`CaseFormat` 使用方法相当简便
```Java
CaseFormat.UPPER_UNDERSCORE.to(CaseFormat.LOWER_CAMEL, "CONSTANT_NAME")); // returns "constantName"
```

>**参考:**
[Strings](https://github.com/google/guava/wiki/StringsExplained)  
[字符串](http://ifeve.com/google-guava-strings/)  
