---
layout: post
title: "Spring 自动注入和静态方法"
date: "2018-05-03"
description: "Spring 自动注入和静态方法"
tag: [java, spring]
---

### 问题描述
有一个自动注入的类实例想用于一个静态方法中, 类实例是使用 Spring 框架管理的, 方法也必须是静态的; 类似于以下实现, 但是以下实现是会报错的...
```
@Service
public class Foo {
    public int doStuff() {
        return 1;
    }
}

public class Boo {
    @Autowired
    private Foo foo;

    public static void randomMethod() {
         foo.doStuff();
    }
}
```

### 解决方法
可以使用 @Autowired 注解在构造方法上, 或者使用 @PostConstruct 将自动注入的类实例再赋值给一个静态变量

#### 使用 @Autowired 注解
这种方法需要在构造方法中传递其他的 Beans 作为构造参数, 在构造方式中将传入的构造参数赋值给静态变量
```
@Component
public class Boo {

    private static Foo foo;

    @Autowired
    public Boo(Foo foo) {
        Boo.foo = foo;
    }

    public static void randomMethod() {
         foo.doStuff();
    }
}
```

#### 使用 @PostConstruct 注解
这种方法是将 Spring 在自动注入后调用 @PostConstruct 注解的方法, 将自动注解的类实例赋值给静态变量
```
@Component
public class Boo {

    private static Foo foo;
    @Autowired
    private Foo tFoo;

    @PostConstruct
    public void init() {
        Boo.foo = tFoo;
    }

    public static void randomMethod() {
         foo.doStuff();
    }
}
```
>**参考:**
[@Autowired and static method](https://stackoverflow.com/questions/17659875/autowired-and-static-method/17660150)  
[Spring @PostConstruct和@PreDestroy实例](https://www.jianshu.com/p/18265dd3c38d)  
[Spring 容器中 bean 初始化和销毁前所做的操作定义方式](https://blog.csdn.net/topwqp/article/details/8681497)
