---
layout: post
title: "Spring AOP"
date: "2019-04-29"
description: "Spring AOP"
tag: [spring, java]
---

### 概念
- joinPoint (连接点): 指被拦截的目标函数
- pointcut (切入点): 对连接点的抽象
- advice (通知): 在切入点上需要执行的动作
- aspect (切面): 切入点和通知的组合
- weaving (织入): 把切面代码编译到连接点代码的过程

#### 切入点指示符
##### 通配符
- `..`: 匹配方法定义中的任意数量的参数, 或者匹配类定义中任意数量的包
- `+`: 匹配给定类的任意子类
- `*`: 匹配任意数量的字符

##### 类型签名表达式
```
# type name: 包名或者类名 (可带通配符)
within(<type name>)
```

##### 方法签名表达式
```
# scope: 方法作用域, 如 public, private, protected
# return-type: 方法返回值类型
# fully-qualified-class-name: 方法所在类的完全限定名
# parameters: 方法参数
execution(<scope> <return-type> <fully-qualified-class-name>.*(parameters))
```

##### 其他表达式
- bean: Spring AOP 扩展的; AspectJ 没有对于指示符, 用于匹配特定名称的 Bean 对象的执行方法
- this: 用于匹配当前 AOP 代理对象类型的执行方法
- target: 用于匹配当前目标对象类型的执行方法
- @within: 用于匹配持有指定注解类型内的方法
- @annotation: 用于匹配持有指定注解的方法

最后, 切入点指示符支持运算符表达式, `and, or, not` 和 `&&, ||, !`

#### 五种通知类型
- 前置通知 (`@Before`)
- 后置通知 (`@AfterReturning`)
- 环绕通知 (`@Around`)
- 异常通知 (`@AfterThrowing`)
- 最终通知 (`@After`)

#### 通知传递参数
TODO

#### aspect 优先级
如果有多个通知需要在同一切入点指定的连接点上执行, 那么在连接点目标函数执行 ("进入") 的通知函数, 最高优先级的通知将会先执行, 在连接点目标函数执行 ("退出) 的通知函数, 最高优先级的通知将会后执行
- 对于同一个切面定义的通知函数, 在类中越先声明顺序越高
- 对于不同切面定义的通知函数, 可以实现 `org.springframework.core.Ordered` 接口, 重写 getOrder() 方法定制返回值, 返回值越小优先级越大

#### Spring AOP 的实现原理
AspectJ 基于静态织入, Spring AOP 基于动态织入; 动态织入又分为 JDK 动态代理和 CGLIB 动态代理
- JDK 动态代理的先决条件是目标对象必须是接口的实现类
- CGLIB 动态代理不需要目标对象为接口实现类, 因为 CGLIB 动态代理时通过继承的方式实现的

>**参考:**
[关于 Spring AOP (AspectJ) 你该知晓的一切](https://blog.csdn.net/javazejian/article/details/56267036)
