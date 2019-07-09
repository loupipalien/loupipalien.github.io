---
layout: post
title: "Java 中的断言"
date: "2018-08-02"
description: "Java 中的断言"
tag: [java]
---

### 断言
**断言是为了方便调试程序, 并不是发布程序的组成部分**
C/C++语言中有断言的功能(assert), 在Java SE 1.4 版本以后也增加了断言的特性; 默认情况下 JVM 是关闭断言的, 因此如果想使用断言调试程序, 需要手动打开断言功能; 在命令行模式下运行 Java 程序时可增加参数 -enableassertions 或者 -ea 打开断言; 可通过 -disableassertions 或者 -da 关闭断言 (默认是关闭的)

#### 断言语法
- assert <bool expression>;
```
public class AssertionTest {

	public static void main(String[] args) {

		boolean isSafe = false;
		assert isSafe;
		System.out.println("断言通过!");
	}
}
```
- assert <bool expression> : <message>
```
public class AssertionTest {

	public static void main(String[] args) {

		boolean isSafe = false;
		assert isSafe : "Not safe at all";
		System.out.println("断言通过!");
	}

```

运行时开启前断言: `java -ea AssertionTest`

#### 陷阱
断言只是为了用来调试程序, 切勿将断言写入业务逻辑中, 容易出现问题
```
public class AssertionTest {

	public static void main(String[] args) {

			assert ( args.length > 0);
			System.out.println(args[1]);
	}
}
```
该句断言 `assert (args.length > 0)` 和 `if(args.length > 0)` 基本等效, 但是如果在发布程序的时候 (一般不会开启断言), 所以该句会被忽视, 因此会导致以下
```
Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException: 1
	at AssertionTest.main(AssertionTest.java:7)

```

#### 替代品
- JUnit
在使用 JUnit 写单元测试时, 断言是默认开启的

>**参考:**
[Java 之 assert (断言)](https://blog.csdn.net/tounaobun/article/details/8468401)
