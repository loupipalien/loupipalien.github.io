---
layout: post
title: "代理模式"
date: "2019-05-29"
description: "代理模式"
tag: [java, design mode]
---

### 问题
通常在写一个功能函数时, 经常需要在其中写入一些与功能无关但有必要的代码; 例如: 日志记录, 安全审计, 调用统计, 事务支持等等; 这些代码是非功能性的, 又不可或缺, 但又想保持代码干净单一低耦合

### 模式定义
代理模式 (Proxy): 为其他对象提供一种代理以控制对这个对象的访问  

### 模式结构
代理模式包含如下角色
- Subject: 抽象主题角色 (接口或者父类)
- RealSubject: 真实主题角色 (被代理对象)
- Proxy: 代理主题角色 (代理对象)

![代理模式.png](http://ww1.sinaimg.cn/large/d8f31fa4ly1g3ipb3un0vj20t00htgm6.jpg)

### 常见应用
- 远程代理: 为一个对象在不同的地址空间提供局部代表, 这样可以隐藏一个对象存在与不同地址空间的事实
- 虚拟代理: 根据需要创建开销很大的对象, 通过它来存放实例化需要很长时间的真实对象
- 安全代理: 用来控制真实对象访问时额权限
- 智能引用: 当调用真实对象时, 代理处理额外一些事

### 代理实现
代理模式的实现, 可分为静态代理和动态代理, 其中动态代理有分为 JDK 动态代理和 CGLIB 动态代理

#### 静态代理
静态代理在使用时, 需要定义接口或者父类, 被代理对象和代理对象实现相同的接口或者继承相同的父类, 其中代理对象持有被代理对象的引用 (在编译期生成代理类的 class 文件)

##### 代码示例
```
// 接口
public interface UserService {

    public void save();
}

// 目标
public class UserServiceImpl implements UserService {

    @Override
    public void save() {
        System.out.println("save.");
    }
}

// 代理
public class UserServiceProxy implements UserService {
    // 目标对象
    private UserService target;

    public UserServiceProxy(UserService target) {
        this.target = target;
    }

    @Override
    public void save() {
        System.out.println("invoke before.");
        target.save();
        System.out.println("invoke after.");
    }
}

// 客户端
public class App {

    public static void main(String[] args) {
        // 代理对象
        UserService userService = new UserServiceProxy(new UserServiceImpl());
        // 代理执行
        userService.save();
    }
}
```
##### 优缺点

| 优点 | 缺点 |
| :--- | :--- |
| 可以做到不修改目标对象的代码的前提先对目标进行一定的扩展 | 代理类和目标类需要实现相同的接口或者继承相同的父类, 这会导致大量的代码重复 |
| - | 当有需要被代理的类较多时, 会产生同样数目的代理类 |

#### 动态代理
动态代理与静态代理最大的不同是: 代理是在运行时动态产生的, 即编译完成后没有代理类的 class 文件, 而是在运行时动态生成字节码, 并加载到 JVM 中

##### JDK 动态代理
JDK 动态代理利用了 JDK API, 动态的在内存中构建代理对象, 动态代理对象不需要实现接口, 但是要求目标对象
必须实现接口, 否则不能实现 JDK 动态代理

###### 代码示例
```
// 接口
public interface UserService {

    public void save();
}

// 目标
public class UserServiceImpl implements UserService {

    @Override
    public void save() {
        System.out.println("save.");
    }
}

// 代理工厂
public class ProxyFactory {

    // 目标对象
    private Object target;

    public ProxyFactory(Object target) {
        this.target = target;
    }

    // 为目标对象生成代理对象
    public Object getProxyInstance() {
        return Proxy.newProxyInstance(target.getClass().getClassLoader(),
                target.getClass().getInterfaces(), new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        System.out.println("invoke before.");
                        Object result = method.invoke(target, args);
                        System.out.println("invoke after.");
                        return result;
                    }
                });
    }
}

// 客户端
public class App {

    public static void main(String[] args) {
        // 代理对象
        UserService userService = (UserService) new ProxyFactory(new UserServiceImpl()).getProxyInstance();
        // 代理执行
        userService.save();
    }
}
```

###### 优缺点

| 优点 | 缺点 |
| :--- | :--- |
| 可以动态生成代理类, 减少了重复代码以及代理类数量, 更加灵活 | 代理类必须实现 InvocationHandler 接口, 通过反射调用, 执行性能有一些降低 |

##### CGLIB 代理
CGLIB (Code Generation Library) 是一个第三方代码生成类库, 运行时在内存中动态生成一个子类对象从而实现对目标对象功能的扩展

###### 代码示例
```
// 第三方包
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
    <version>3.2.12</version>
</dependency>

// 目标对象
public class UserService {

    public void save() {
        System.out.println("save.");
    }
}

// 代理工厂
public class ProxyFactory implements MethodInterceptor {

    // 目标对象
    private Object target;

    public ProxyFactory(Object target) {
        this.target = target;
    }

    // 为目标对象生成代理对象
    public Object getProxyInstance() {
        // 加强器
        Enhancer enhancer = new Enhancer();
        // 设置父类
        enhancer.setSuperclass(target.getClass());
        // 设置回调
        enhancer.setCallback(this);
        // 返回代理
        return enhancer.create();
    }

    @Override
    public Object intercept(Object object, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("invoke before.");
        Object result = proxy.invokeSuper(object, args);
        System.out.println("invoke after.");
        return result;
    }
}

// 客户端
public class App {

    public static void main(String[] args) {
        // 代理对象
        UserService userService = (UserService) new ProxyFactory(new UserService()).getProxyInstance();
        // 代理执行
        userService.save();
    }
}
```

###### 优缺点

| 优点 | 缺点 |
| :--- | :--- |
| 可以为非接口实现类生成代理类, 执行性能比反射快 | 需要引入第三方包 |

>**参考:**  
[代理模式](https://design-patterns.readthedocs.io/zh_CN/latest/structural_patterns/proxy.html)  
[Java的三种代理模式](https://segmentfault.com/a/1190000009235245)  
[Java三种代理模式：静态代理、动态代理和cglib代理](https://segmentfault.com/a/1190000011291179)  
