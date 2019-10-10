---
layout: post
title: "Guava 用户指南"
date: "2019-09-10"
description: "Guava 用户指南"
tag: [java, guava]
---

### [Guava](https://github.com/google/guava)
- Guava 是什么
Guava 是来自 Google 的工具类库集合, 包含了 collections, caching, primitives support, concurrency libraries, common annotations, string processing, I/O, Math 等等
- 为什么要使用 Guava
Guava 提供了非常友好工具包, 与 Apache-Commons 等包相比, 设计更优秀, 文档更齐全, 代码质量更高, 社区更活跃等等; 与 Apache-Commons 的具体区别见 [这里](https://stackoverflow.com/questions/4542550/what-are-the-big-improvements-between-guava-and-apache-equivalent-libraries)

#### 使用和避免 NULL
轻率的使用 NULL 可能会导致很多令人错愕的问题, Guava 认为相比默默的接受 NULLL, 使用快速失败拒绝 NULL 值对开发者更有帮助; 除此之外, NULL 很少可以明确表示某种语义, 例如 Map.get(key) 返回 null, 可能表示 Map 中的值是 NULL, 也可能是 Map 中没有 key 对应的值; 调用返回的 NULL 可以表示失败或者成功; 由于 NULL 的模糊语义导致了许多问题, Guava 提供了 ·`Optional` 工具类来解决这一问题  
Guava 提供 `Optional<T>` 表示可能为 NULL 的 T 类型引用; 一个 Optional 实例可能包含非 NULL 的引用 (称之为引用存在), 也可能什么也不包括 (引用缺失); 它从不说包含的是 NULL, 而是用存在或缺失表示
```Java
Optional<Integer> possible = Optional.of(5);
possible.isPresent(); // returns true
possible.get(); // returns 5
```
使用 Optional 除了赋予 NULL 语义, 增加了可读性, 最大的优点在于傻瓜式的防护; Optional 迫使开发者思考引用缺失的情况, 因为必须显式的从 Optional 获取引用, 直接使用 NULL 容易让人忘记这些情形; 所以将方法的返回类型指定为 Optional, 也可以迫使调用者思考返回时引用缺失的情形

#### 前置条件
Guava 在 `Preconditions` 类中提供了若干前置条件判断的实用方法, 强烈建议静态导入这些方法; 每个方法都有三个变种
- 没有额外参数: 抛出的异常中没有错误消息
- 有一个 Object 对象作为额外参数: 抛出的异常使用 Object.toString() 作为错误消息
- 有一个 String 对象作为额外参数, 并且有一组任意数量的附加 Object 对象: 这个变种处理异常消息的方式类似 printf, 但只支持 `%s` 替换符
```java
checkArgument(i >= 0, "Argument was %s but expected nonnegative", i);
checkArgument(i < j, "Expected i < j, but %s > %s", i, j);
```
相比 Apache Commons 提供的类似方法, Guava 的 Preconditions 更加优秀; Piotr Jagielski 在他的博客中简要地列举了一些理由
- 在静态导入后, Guava 方法非常清楚明晰, checkNotNull 清楚的描述做了什么, 会抛出什么异常
- checkNotNull 直接返回检查的参数, 这可以在构造函数中保持字段的单行赋值风格: `this.field = checkNotNull(field)`
- 简单的, 参数可变的 printf 风格异常信息; 鉴于这个优点, 在 JDK7 已经引入 `Objects.requireNonNull` 的情况下, 仍然建议使用 checkNotNull

#### 排序
`Ordering` 是流式风格比较器的实现, 用来构建较为复杂的比较器, 以完成集合排序的功能; 从实现上来说, Ordering 实例就是一个特殊的 Comparator 实例, Ordering 把很多机遇 Comparator 的静态方法 (如 Collections.max) 包装为自己的实例方法, 并且提供了链式调用来定制和增强现有的比较器

##### 创建比较器: 常见的比较器可以由下面的静态方法创建

| 方法 | 描述 |
| :--- | :--- |
| natural() | 对排序类型做自然排序 |
| usingToString() | 按对象的字符串形式做字典排序 [lexicographical ordering]|
| from(Comparator) | 把给定的 Comparator 转化为排序器 |

实现自定义的排序器时, 除了可以使用上面的 from 方法, 也可以跳过实现 Comparator, 直接继承 Ordering
```Java
Ordering<String> byLengthOrdering = new Ordering<String>() {
    public int compare(String left, String right) {
        return Ints.compare(left.length(), right.length());
    }
};
```
##### 链式调用方法: 通过链式调用, 可以由给定的排序器衍生出其他排序器

| 方法 | 描述 |
| :--- | :--- |
| reverse() | 获取语义相反的排序器 |
| nullsFirst() | 使用当前排序器, 但额外把 NULL 值排到最前面 |
| nullsLast() | 使用当前排序器, 但额外把 NULL 值排到最后面 |
| compound(Comparator) | 合成另一个比较器, 已处理当前排序器中相等的情况 |
| lexicographical() | 基于处理类型 T 的排序器, 返回改类型的可迭代对象 Iterable<T> 的排序器 |
| onResultOf(Function) | 对集合元素调用 Function, 再按返回值用当前排序器排序 |

以下是使用链式调用合成的排序器
```Java
class Foo {
    @Nullable String sortedBy;
    int notSortedBy;
}

Ordering<Foo> ordering = Ordering.natural().nullsFirst().onResultOf(new Function<Foo, String>() {
    public String apply(Foo foo) {
        return foo.sortedBy;
    }
});
```
超过一定长度的链式调用可能会带来于都和理解上的难度, 建议一个链式中最多使用三个方法, 此外还可以吧 Function 抽离为中间对象, 让链式调用更加简洁紧凑

##### 运用排序器: 排序器实现了若干操作集合或元素的方法

| 方法 | 描述 | 另见 |
| :--- | :--- | :--- |
| greatestOf(Iterable iterable, int k) | 获取了迭代对象中的最大 k 个元素 | leastOf |
| isOrdered(Iterable) | 判断可迭代对象是否已按排序器排序: 允许有排序值相等的元素 | isStrictlyOrdered |
| sortedCopy(Iterable) | 返回可迭代对象迭代元素的列表 | immutableSortedCopy |
| min(E, E) | 返回两个参数中最小的那个; 如果相等, 则返回第一个参数 | max(E,E) |
| min(E, E, E, E...) | 返回多个参数中最小的那个; 如果有超过一个参数都最小, 则返回第一个最小的参数 | max(E, E, E, E...) |
| min(Iterable) | 返回迭代器中最小的元素; 如果可迭代对象中没有元素, 则抛出 NoSuchElementException | max(Iterable) |

#### 常见 Object 方法
##### equals
当一个对象中的字段可以为 NULL 时, 实现 Object.equals 方法会很痛苦, 因为不得不进行 NULL 检查, 使用 `Objects.equal` 可帮助执行 NULL 敏感的 equals 判断, 从而避免抛出 NPE
```Java
Objects.equal("a", "a"); // returns true
Objects.equal(null, "a"); // returns false
Objects.equal("a", null); // returns false
Objects.equal(null, null); // returns true
```
JDK7 引入的 Objects 类提供了一样的方法 Objects.equals
##### hashCode
用对象的所有子弹作散列运算较为简单; Guava 的 `Objects.hashCode(Object...)` 会对传入的字段序列计算计算出合理的, 顺序敏感的散列值; 可以使用 `Objects.hashCode(field1, field2, ..., fieldn)` 来代替后动计算散列值  
JDK7 引入的 Objects 类提供了一样的方法 Objects.hash(Object...)
##### toString
使用 `Objects.toStringHelper` 可以轻松编写有用的 toString 方法
```Java
// Returns "ClassName{x=1}"
Objects.toStringHelper(this).add("x", 1).toString();
// Returns "MyObject{x=1}"
Objects.toStringHelper("MyObject").add("x", 1).toString();
```
##### compare/compareTo
通常实现一个比较器的代码如下
```Java
class Person implements Comparable<Person> {
    private String lastName;
    private String firstName;
    private int zipCode;
    public int compareTo(Person other) {
    int cmp = lastName.compareTo(other.lastName);
    if (cmp != 0) {
      return cmp;
    }
    cmp = firstName.compareTo(other.firstName);
    if (cmp != 0) {
      return cmp;
    }
    return Integer.compare(zipCode, other.zipCode);
  }
}
```
但这部分代码太琐碎了, 容易乱也难以调试; Guava 提供了 `ComparisonChain`; ComparisonChain 执行一种懒比较, 执行比较操作直到发现非零的结果, 在那之后的比较输入将被忽略
```Java
public int compareTo(Foo that) {
    return ComparisonChain.start()
            .compare(this.aString, that.aString)
            .compare(this.anInt, that.anInt)
            .compare(this.anEnum, that.anEnum, Ordering.natural().nullsLast())
            .result();
}
```

#### 异常传播
TODO

>**参考:**
[What are the big improvements between guava and apache equivalent libraries?](https://stackoverflow.com/questions/4542550/what-are-the-big-improvements-between-guava-and-apache-equivalent-libraries)  
[User Guide](https://github.com/google/guava/wiki)
[Google Guava官方教程](http://ifeve.com/google-guava/)
