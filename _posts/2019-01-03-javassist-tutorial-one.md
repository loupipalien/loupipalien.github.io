---
layout: post
title: "Javassist 文档一"
date: "2019-01-03"
description: "Javassist 文档一"
tag: [java, javassist]
---

### 读写字节码
Javassist 是一个处理 Java 字节码的类库; Java 字节码存储一个叫 class 的二进制文件中, 每个 class 文件包含一个 Java 类或接口  
类 `Javassist.CtClass` 是对一个 class 文件的抽象; 一个 `CtClass` (compile-time class) 对象是一个 class 文件的处理句柄; 以下程序是一个非常简单的示例
```
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("test.Rectangle");
cc.setSuperclass(pool.get("test.Point"));
cc.writeFile();
```
程序首先获取一个 `ClassPool` 对象, 用于控制字节码的修改; `ClassPool` 是代表一个 class 文件的 `CtClass` 对象的容器; 它可以按需读取一个 class 文件用于构建 `CtClass` 对象, 并且为了响应后续访问记录已构建的对象; 为了修改一个类的定义, 用户必须先从 `ClassPool` 对象中获取一个 `CtClass` 对象引用, 使用在 `ClassPool` 的 class.get() 来达到这个目的; 在以上程序的示例中, `CtClass` 对象表示从 `ClassPool` 对象中获取的 `test.Ranctangle` 类, 并且分配给了变量 cc, 通过 `getDefault()` 返回的 `ClassPool` 对象会搜索默认的系统路径  
从实现的角度来看, `ClassPool` 是 `CtClass` 对象的哈希表, 用于使用类名用 keys.get() 在 `ClassPool` 中搜索哈希表找到关联到指定 key 的 `CtClass` 对象; 如果没有找到这样的 `CtClass` 对象, get() 方法会读取 class 文件来构造一个新的 `CtClass` 对象, 这个对象会记录在哈希表中, 然后作为 get() 的结果值返回  
从 `ClassPool` 对象中获取的 `CtClass` 对象可以被修改 ([如何修改 `CtClass` 的细节见后续章节]()); 在以上示例中, 将 `test.Ranctangle` 的父类修改为 `test.Point`, 这个修改会在 `CtClass.writeFile()` 方法调用是反射到原始类上  
`writeFile()` 会装换 `CtClass` 对象为一个 class 文件并且写到本地磁盘; Javassist 也提供了直接获取修改字节码的方法; 为了获取字节码可调用 ·`tiBytecode()`
```
byte[] b = cc.toBytecode();
```
也可以直接加载 `CtClass`
```
Class clazz = cc.toClass();
```
`toClass()` 会请求线程上下文类加载器来加载 `CtClass` 代表的 class 文件; 这会返回表示被加载类的 `java.lang.Class` 对象; 更多细节见 [以下章节]()

#### 定义一个新的 class
为了从片段中定义一个新类, `makeClass()` 必须在一个 `ClassPool` 上调用
```
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.makeClass("Point");
```
程序定义一个无成员的 `Point` 类, `Point` 类的成员方法可以使用工厂方法 `CtNewMethod` 声明, 再使用 `CtClass` 中的 `addMethod()` 添加到 `Point` 类中  
`makeClass()` 不能创建新的接口, `ClassPool.makeInterface()` 可以创建; 接口中的成员方法可以通过 `CtNewMethod.abstractMethod()` 创建; 注意接口方法必须是抽象方法  

#### 冻结 classes
如果一个 `CtClass` 类通过 `writeFile(), toClass(), toBytecode()` 方法转换为 class 文件, Javassist 会冻结这个 `CtClass` 对象; 后续堆 `CtClass` 对象的修改是不允许的; 这用于警告开发者, 当试图修改一个已经被加载过的 class 文件, 因为 JVM 不允许重加载类  
冻结的类可以解冻, 这样对类定义的修改将会被允许; 例如
```
CtClass cc = ...;
    :
cc.writeFile();
cc.defrost();
cc.setSuperclass(...);    // OK since the class is not frozen.
```
`defrost()` 在调用后, `CtClass` 就可以被再次修改  
如果 `ClassPool.doPruning` 被设置为 true, 当 Javassist 冻结 `CtClass` 时会修建包含的数据结构; 为了减少内存消耗, 修建对象中废弃不需要的属性 (attribute_info 结构); `ClassPool.doPruning` 默认值是 false  
禁止修改指定的 `CtClass` 对象, 在 `CtClass` 对象上调用 `stopPruning()` 优先级更高  
```
CtClasss cc = ...;
cc.stopPruning(true);
    :
cc.writeFile();                             // convert to a class file.
// cc is not pruned.
```
`CtClass` 对象 cc 没有被修剪, 因此可以在 `writeFile()` 调用后解冻  
>**注意:**
在 debug 时, 可能想临时停止修剪并且冻结写出一个修改的 class 文件到磁盘上, `debugWriteFile()` 是为此目的的便捷方法; 它会停止修剪, 写出 class 文件, 解冻, 并且再次开启修剪 (如果初始时开启的)

#### Class 搜索路径
通过静态方法 `ClassPool.getDefault()` 返回的默认的 `ClassPool` 会搜索底层 JVM 拥有的相同路径; 如果程序运行在一个 web 应用服务器上, 例如 JBoss 和 Tomcat, `ClassPool` 可能找不到用户的 classes, 因为 web 应用服务器使用多个类加载器和系统类加载器; 在这种情况下, 额外的 class path 必须注册在 `ClassPool` 中; 假定 pool 指向 `ClassPool` 对象
```
pool.insertClassPath(new ClassClassPath(this.getClass()));
```
这行语句注册的用于加载 class 对象的 class path 是 this 对象的 class 指向的 class path; 可以使用任意的 `Class` 对象作为参数代替 `this.getClass()`; 用于加载 这个 class 的 class path 表示 `Class` 对象已注册  
可以注册目录名作为类搜索路径; 例如, 以下代码添加 `/usr/local/javalib` 目录到搜索路径
```
ClassPool pool = ClassPool.getDefault();
pool.insertClassPath("/usr/local/javalib");
```
可以添加的搜索路径不仅可以是个目录也可以使 URL
```
ClassPool pool = ClassPool.getDefault();
ClassPath cp = new URLClassPath("www.javassist.org", 80, "/java/", "org.javassist.");
pool.insertClassPath(cp);
```
这段代码添加 "http://www.javassist.org:80/java/" 到类搜索路径中; 这个 URL 仅用于搜索属于 `org.javassist` 的类; 例如, 为了加载类 `org.javassist.test.Main` 它的 class  文件可以从此 `http://www.javassist.org:80/java/org/javassist/test/Main.class` 获取   
另外, 可以直接给定一个字节数组到 `ClassPool` 对象中, 并从数组中构造一个 `CtClass` 对象; 使用 `ByteArrayClassPath` 可以做到, 例如
```
ClassPool cp = ClassPool.getDefault();
byte[] b = a byte array;
String name = class name;
cp.insertClassPath(new ByteArrayClassPath(name, b));
CtClass cc = cp.get(name);
```
`CtClass` 对象表示由 b 指定的 class 文件的类定义; 如果 `get()` 被调用时, `ClassPool` 从给定 `ByteArrayClassPath` 中获取 class 文件, 并且 `get()` 获取的 class 名等同于 `ByteArrayClassPath` 指定的名称  
如不知道类的全限定名, 可以使用 `ClassPool.makeClass()`
```
ClassPool cp = ClassPool.getDefault();
InputStream ins = an input stream for reading a class file;
CtClass cc = cp.makeClass(ins);
```
`makeClass` 返回从给定的输入流中构造的 `CtClass`; 可以使用 `makeClass()` 可以快速将 class 文件放入 `ClassPool` 对象中; 如果搜索路径中包含大的 jar 文件可以提升性能; 因为 `ClassPool` 对象是按需读取 class 文件, 在加载每个 class 文件时可能重复搜索这个 jar 文件, `makeClass()` 可以用于优化搜索; 通过 `makeClass()` 构造的 `CtClass` 可以保持在 `ClassPool` 对象中, 并且 class  文件不会被重复读取  
用户可以扩展 class 搜索路径, 可以定义一个新的 class 实现 `ClassPath` 接口, 并且将 class 的实例使用 `insertClassPath()` 到 `ClassPool` 中; 这允许在搜索路径中包含非标资源

### ClassPool
`ClassPool` 对象是 `CtClass` 对象的容器, 一旦一个 `CtClass` 对象被创建, 会在 `ClassPool` 中永久记录; 这是因为当编译器编译指向这个用 `CtClass` 表示的 class 源码时,后续可能需要访问这个 `CtClass` 对象  
例如, 假设一个新的方法 `getter()` 被添加到表示 `Point` 类的 `CtClass` 对象中; 随后, 程序会尝试编译在 `Point` 包含 `getter()` 的方法调用源码, 并且使用被添加到另一个类 `Line` 的编译代码作为方法体; 如果表示 `Point` 的 `CtClass` 对象丢失了, 编译器将不能编译 `getter()` 的方法调用; 注意原始类定义并不包含 `getter()`; 因此, 正确的编译方法调用, `ClassPool` 必须包含程序运行时的所有 `CtClass` 实例

#### 避免内存溢出
如果 `CtClass` 的数量变得相当大, `ClassPool` 可能会造成巨大的内存消耗 (这种情况很少发生, 因为 Javassist 尝试使用各种 [方法]() 来减少内存消耗); 为了避免这个问题, 可以从 `ClassPool` 中明确移除非必须的 `CtClass` 对象; 如果调用了 `CtClass` 对象的 `detach()` 方法, 那么 `CtClass` 对象会从 `ClassPool` 中移除; 例如
```
CtClass cc = ... ;
cc.writeFile();
cc.detach();
```
在 `detach()` 方法调用后将不能调用 `CtClass` 的任何方法; 然而你可以调用 `ClassPool.get()` 方法获取一个代表相同类的新 `CtClass` 实例; 如果调用 `get()`, `ClassPool` 会再次读取 class 文件并新创建一个 `CtClass` 对象被 `get()` 方法返回   
另一个关联点是使用一个新的 `ClassPool` 替换并丢弃旧的; 如果旧的 `ClassPool` 被垃圾回收了, 那么包含在 `ClassPool` 中的 `CtClass` 也将会被回收; 为了创建一个新的 `ClassPool` 实例, 执行以下代码片段
```
ClassPool cp = new ClassPool(true);
// if needed, append an extra search path by appendClassPath()
```
创建的 `ClassPool` 和通过 `ClassPool.getDefault()` 返回 `ClassPool` 一致; 注意 `ClassPool.getDefault()` 是为了便利提供的单例工厂方法; 它创建 `ClassPool` 对象和以上显示创建的方式相同, 除了它维护了一个单例的 `ClassPool` 并且重用; `getDefault()` 返回的 `ClassPool` 对象没有指定特殊的角色, `getDefault()` 是一个便利方法  
注意 `new ClassPool(true)` 是一个便利的构造器, 可以构造一个 `ClassPool` 对象并且添加系统搜索路径; 调用此构造器等同于以下代码
```
ClassPool cp = new ClassPool();
cp.appendSystemPath();  // or append another path by appendClassPath()
```

#### 级联 ClassPools
如果程序运行在 web 应用服务器上, 创建多个 `ClassPool` 实例可能是必须的; 应该为每个类加载器创建一个 `ClassPool`; 程序应该通过 `ClassPool` 的构造器而不是通过调用 `getDefault()` 创建; 多个 `ClassPool` 对象可以像 `java.lang.ClassLoader` 那样级联; 例如
```
ClassPool parent = ClassPool.getDefault();
ClassPool child = new ClassPool(parent);
child.insertClassPath("./classes");
```
如果 `child.get()` 被调用, 孩子 `ClassPool` 首先代理到父亲 `ClassPool`, 如果父亲 `ClassPool` 查找类文件失败, 孩子 `ClassPool` 才尝试在 `./classes` 目录下查找类文件  
如果 `child.childFirstLookup` 为 true, 孩子 `ClassPool` 则在代理到父亲 `ClassPool` 之前尝试查找类文件; 例如
```
ClassPool parent = ClassPool.getDefault();
ClassPool child = new ClassPool(parent);
child.appendSystemPath();         // the same class path as the default one.
child.childFirstLookup = true;    // changes the behavior of the child.
```

#### 为定义的新类修改类名
新类的定义可以作为现有类的拷贝, 例如
```
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("Point");
cc.setName("Pair");
```
程序首先获取 `Point` 的 `CtClass` 对象; 调用 `setName()` 会给予 `CtClass` 对象一个新的名称 `Pair`; 在这之后, `CtClass` 对象中类定义所有类名出现的地方都会从 `Point` 修改为 `Pair`; 类定义的其他部分并没有改变  
注意在 `CtClass` 中的 `setName()` 改变了 `ClassPool` 对象中的记录; 从实现的角度来看, `ClassPool` 对象是一个哈希表, `Ctclass` 的 `setName()` 方法改变了哈希表中与 `CtClass` 对象关联的键; 键从原始类名改变为新类的名称  
因此, 如果 `get("Point")` 再次在 `ClassPool` 对象上调用, 它将不会返回变量 `cc` 指向的 `Ctclass` 对象; `ClassPool` 对象会再次读取 `Point.class` class 文件, 并且重新为 `Point` 类构建一个新的 `CtClass` 对象; 这是因为与键 `Poiot` 关联的 `CtClass` 不再存在; 见以下示例
```
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("Point");
CtClass cc1 = pool.get("Point");   // cc1 is identical to cc.
cc.setName("Pair");
CtClass cc2 = pool.get("Pair");    // cc2 is identical to cc.
CtClass cc3 = pool.get("Point");   // cc3 is not identical to cc.
```
`cc1` 和 `cc2` 指向的是相同的 `CtClass` 实例, 但 `cc3` 并不是; 注意在 `cc.setName("Pair")` 执行后, `cc` 和 `cc1` 都指向 `Pair` 类   
`ClassPool` 对象用于维护类和 `CtClass` 对象的一对一关系, Javassist 不允许两个不同的 `CtClass` 对象表示相同的类, 除非创建两个独立的 `ClassPool`; 对于程序一致性装换是非常重要的特性  
为了创建默认实例 `ClassPool` 的拷贝, 默认实例由 `ClassPool.getDefault()` 返回; 可以执行以下代码片段
```
ClassPool cp = new ClassPool(true);
```
如果有两个 `ClassPool` 对象, 可以从每个 `ClassPool` 中获取表示相同类的不同 `CtClass` 对象; 可以修改这些 `CtClass` 对象来生成不同版本的类

#### 为了定义一个新类重命名一个冻结类
一旦 `CtClass` 对象通过 `writeFile()` 或 `toBytecode()` 方法转换为 class 文件, Javassist 将拒绝进一步修改 `CtClass` 对象; 因为表示 `Point` 类的 `CtClass` 对象被转换为类文件, 就不能通过拷贝 `Point` 来定义 `Pair` 类, 因为在 `Point` 上执行 `setName()` 会被拒绝; 以下代码片段是错误的
```
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("Point");
cc.writeFile();
cc.setName("Pair");    // wrong since writeFile() has been called.
```
为了避免这种限制, 可以调用 `ClassPool` 的 `getAndRename()` 方法, 例如
```
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("Point");
cc.writeFile();
CtClass cc2 = pool.getAndRename("Point", "Pair");
```
调用 `getAndRename()` 方法时, `ClassPool` 首先读取 `Point.class` 来创建一个代表 `Point` 类的 `CtClass` 对象, 在记录到 `ClassPool` 的哈希表前将 `CtClass` 对象从 `Point` 转换到 `Pair`; 因此 `getAndRename()` 可以在表示 `Point` 类的 `CtClass` 对象的 `writeFile()` 或 `toBytecode()` 方法调用后执行

### 类加载器
如果类的修改是提前知道的, 修改类的最简便的方法如下
- 通过调用 `ClassPool.get()` 获取 `CtClass` 对象
- 修改它
- 调用 `CtClass` 对象的 `writeFile()` 或 `toBytecode()` 方法以便再次获取修改的类文件

如果类的修改在加载时还未确定, 用户必须使 Javassist 与类加载器协作; Javassist 可以和类加载器一起使用时字节码才可以在加载时修改, 用户可以自定义类加载器也可以使用 Javassist 提供的类加载器

#### CtClass 的 toClass 方法
`CtClass` 提供了 `toClass()` 的便利方法, 会请求线程上下文类加载器加载 `CtClass` 表示的类; 调用这个方法, 调用者需要有合适的权限, 否则会抛出 `SecurityException` 异常; 以下代码展示如何使用 `toClass()`
```
public class Hello {
    public void say() {
        System.out.println("Hello");
    }
}

public class Test {
    public static void main(String[] args) throws Exception {
        ClassPool cp = ClassPool.getDefault();
        CtClass cc = cp.get("Hello");
        CtMethod m = cc.getDeclaredMethod("say");
        m.insertBefore("{ System.out.println(\"Hello.say():\"); }");
        Class c = cc.toClass();
        Hello h = (Hello)c.newInstance();
        h.say();
    }
}
```
`Test.main()` 在 `Hello` 的 `say()` 方法体中插入了 `println()` 的调用; 然后构造了修改后的 `Hello` 类的实例, 并调用了实例的 `say()` 方法; 注意以上程序依赖的事实是 `Hello` 类在 `toClass()` 调用之前未被加载过; 如果不是, 即 JVM 在 `toClass()` 请求加载修改的 `Hello` 类之前家在旅客原始类; 这样加载修改的 `Hello` 类将会失败 (抛出 `LinkageError`) ; 例如, 如果 `Test` 的 `main()` 方法如下所示
```
public static void main(String[] args) throws Exception {
    Hello orig = new Hello();
    ClassPool cp = ClassPool.getDefault();
    CtClass cc = cp.get("Hello");
        :
}
```
原始 `Hello` 类在 `main()` 方法的第一行加载, 再调用 `toClass()` 方法会抛出异常, 因为类加载器不能同时加载两个不同版本的 `Hello` 类  
如果程序运行在例如 JBoss 和 Tomcat 的应用服务器上, `toClass()` 使用线程上下文加载器可能不正确; 在这种情况下, 会出现 `ClassCastException`; 为了避免这种情况, 必须为 `toClass()` 明确指定正确的类加载器; 例如, 如果 `bean` 是会话 bean 对象, 则如下代码可以工作
```
CtClass cc = ...;
Class c = cc.toClass(bean.getClass().getClassLoader());
```
为 `toClass()` 给定程序已加载的类加载器  
`toClass()` 是提供的便利方法, 如果需要更复杂的功能, 需要自定义类加载器

#### Java 中的类加载
在 Java 中多级类加载器可以同时存在, 每个类加载器有自己的命名空间; 不同的类加载器可以加载相同类名的不同类文件; 加载的两个类会被认为是不同的; 这个特性允许在单个 JVM 中运行多个应用程序, 甚至程序中包含同名的不同类
>**注意:**
JVM 不允许动态的重加载类, 一旦一个类加载器加载了一个类, 它就不能再运行时重加载这个类的修改版本;因此, 不能再 JVM 加载类后修改类的定义; 然而, JPDA  (Java Platform Debugger Architecture) 提供了限制性的类重加载, 见 3.6 节

如果同一个类被两个不同的类加载器加载, JVM 会认为是有着相同类名和定义的不同类; 这两个类会被认为是不同的; 因为两个类时不相同的, 一个类的实例不能赋值给另一个类的类型变量; 两个类之间的 cast 操作会失败并抛出 `ClassCastException`  
例如, 以下代码片段会抛出异常
```
MyClassLoader myLoader = new MyClassLoader();
Class clazz = myLoader.loadClass("Box");
Object obj = clazz.newInstance();
Box b = (Box)obj;    // this always throws ClassCastException.
```
`Box` 类被两个加载器加载; 假定类加载器 CL 加载了代码片段中的类, 因为代码片段中包括了 `MyClassLoader, Class, Object, Box`, CL 加载了这些类 (除非它代理到了其他的类加载器); 因为变量 b 的类型 `Box` 是 CL 加载, 另一方面 `myLoader` 也加载了 `Box` 类, 对象 `obj` 是 `myLoader` 加载的 `Box` 的实例; 因此最后的语句会抛出 `ClassCastException`, 因为 `obj` 的类与变量 `b` 的类型是不同版本的 `Box` 类  
多级类加载器形成一个树形结构, 除了 启动类加载器之外每个类加载器都有一个父类加载器, 通常会加载子类加载器的类; 因为类的加载请求可被代理到这个类加载器的继承结构, 一个类的类加载器可能并不是请求的类加载器; 因此, 被请求用于加载类 C 的类加载器可能不是真正加载类 C 的类加载器; 为了区分, 称前一个加载器为 C 的启动器, 后一个加载器为 C 的真实加载器  
另外, 如果一个类加载器 CL 请求加载类 C (C 的启动器) 代理到了父类加载器 PL, 那么类加载器 CL 不会在请求加载在类 C 定义中的任何类引用, CL 不再是这些类的启动器, 父类加载 PL 称为它们的启动器并且请求加载这些类; 类 C 定义中引用的类也被 C 的真实加载器加载; 为了理解这些行为, 考虑以下代码
```
public class Point {    // loaded by PL
    private int x, y;
    public int getX() { return x; }
        :
}

public class Box {      // the initiator is L but the real loader is PL
    private Point upperLeft, size;
    public int getBaseX() { return upperLeft.x; }
        :
}

public class Window {    // loaded by a class loader L
    private Box box;
    public int getBaseX() { return box.getBaseX(); }
}
```
假定类 `Window` 被类加载器 L 加载, `Window` 类的启动器和真实记载器都是 L; 因为 `Window` 类的定义引用了 `Box`, JVM 将请求 L 去加载 `Box`, 这里假定 L 代理这个任务到父类加载器 PL, 那么 `Box` 的启动器是 L 而真实加载器是 PL; 在这种情况下, `Point` 的启动器不是 L 而是 PL, 与 `Box` 的真实加载器相同; 因此 L 将不会请求加载 `Point`; 接着, 考虑以下略微修改后的示例
```
public class Point {
    private int x, y;
    public int getX() { return x; }
        :
}

public class Box {      // the initiator is L but the real loader is PL
    private Point upperLeft, size;
    public Point getSize() { return size; }
        :
}

public class Window {    // loaded by a class loader L
    private Box box;
    public boolean widthIs(int w) {
        Point p = box.getSize();
        return w == p.getX();
    }
}
```
现在, `Window` 的定义也引用了 `Point`, 在这种情况下, 当类加载器 L 请求加载 `Point` 时也代理到 PL; 必须避免两个类加载重复加载同一个类, 其中一个类加载器必须代理到另一个  
如果当加载 `Point` 时 L 没有代理到 PL, `widthIs()` 将会抛出 ClassCastException; 因为 `Box` 的真实加载器是 PL, `Box` 中引用了 `Point` 也被 PL 加载; 因此, `getSize()` 的结果值是 PL 加载的 `Point` 实例, `widthIs()` 中的变量 `p` 的类型 `Point` 则是 L 加载; JVM 视它们为不同的类型, 因此因为类型不匹配抛出异常  
这种行为是不方便的但是必要的; 如果以下语句不抛出异常
```
Point p = box.getSize();
```
那么 `Window` 的使用者可以打破 `Point` 对象的封装; 例如, PL 加载的 `Point` 的字段 `x` 是私有化的, 如果 L 加载的 `Point` 的定义如下, 那么 `Window` 类可以直接访问 `x` 的值
```
public class Point {
    public int x, y;    // not private
    public int getX() { return x; }
        :
}
```
更多关于 Java 中类加载的细节, 见 [Dynamic Class Loading in the Java Virtual Machine](https://www.researchgate.net/publication/2803707_Dynamic_Class_Loading_in_the_Java_Virtual_Machine)

#### 使用 javassist.Loader
Javassist 提供了一个类加载器 `javassist.Loader`, 这个类加载器使用 `javassist.ClassPool` 来读取类文件  
例如, `javassist.Loader` 可以用于加载被 Javassist 修改的指定类
```
import javassist.*;
import test.Rectangle;

public class Main {
  public static void main(String[] args) throws Throwable {
     ClassPool pool = ClassPool.getDefault();
     Loader cl = new Loader(pool);

     CtClass ct = pool.get("test.Rectangle");
     ct.setSuperclass(pool.get("test.Point"));

     Class c = cl.loadClass("test.Rectangle");
     Object rect = c.newInstance();
         :
  }
}
```
这个程序修改了类 `test.Rectangle`, `test.Rectangle` 的父类被设置为 `test.Point`; 然后加载这个修改的类, 并创建一个 `test.Rectangle` 类的实例  
如果想在加载时按需修改类, 可以为 `javassist.Loader` 添加一个事件监听器; 当类加载器加载一个类时, 添加的时间监听器会被通知; 时间监听器必须实现依赖接口
```
public interface Translator {
    public void start(ClassPool pool)
        throws NotFoundException, CannotCompileException;
    public void onLoad(ClassPool pool, String classname)
        throws NotFoundException, CannotCompileException;
}
```
当事件监听器通过 `javassist.Loader` 的 `addTranslator` 添加到 `javassist.Loader` 时 `start()` 方法会被调用; `onload()` 方法会在 `javassist.Loader` 加载一个类前调用, `onload()` 中可以修改加载的类的定义  
例如, 以下事件监听器会在类加载前将其修改为公共类
```
public class MyTranslator implements Translator {
    void start(ClassPool pool)
        throws NotFoundException, CannotCompileException {}
    void onLoad(ClassPool pool, String classname)
        throws NotFoundException, CannotCompileException
    {
        CtClass cc = pool.get(classname);
        cc.setModifiers(Modifier.PUBLIC);
    }
}
```
注意, `onload()` 方法中不能调用 `toBytecode()` 或者 `writeFile()` 方法, 因为 `javassist.Loader` 调用这些方法时会获取类文件  
为了运行带 `MyTranslator` 对象的应用类 `MyApp`, main 类如下
```
import javassist.*;

public class Main2 {
  public static void main(String[] args) throws Throwable {
     Translator t = new MyTranslator();
     ClassPool pool = ClassPool.getDefault();
     Loader cl = new Loader();
     cl.addTranslator(pool, t);
     cl.run("MyApp", args);
  }
}
```
运行这个程序
```
% java Main2 arg1 arg2...
```
类 `MyApp` 和其他应用类被 `MyTranslator` 转换  
注意, 例如 `MyApp` 的应用类不能够访问加载器类, 例如 `Main2, MyTranslator， ClassPool`, 因为他们在不同的加载器中加载; 应用类通过 `javassist.Loader` 加载, 而加载器类例如 `Main2` 是通过默认加载器加载  
`javassist.Loader` 搜索类的顺序不同于 `java.lang.ClassLoader`; `ClassLoader` 首先将加载操作代理给父类加载器, 只在父类加载不到时才尝试加; 相反, `javassist.Loader` 在代理给父类加载器前先尝试加载类, 代理仅在以下情形
- 调用 `ClassPool` 的 `get()` 方法没有找到类
- 类被指定使用 `delefateLoadingOf()` 方法通过父类加载加载

这样的搜索顺序袁旭加载 Javassist 修改的类; 然而, 当因为某些原因找不到类时会代理给父类加载器; 一旦类被父类加载器加载, 则在类中引用的其他类也会被父类加载器加载, 并且它们不会被修改; 类 C 中所有引用的类都会别 C 的真实加载器加载; 如果应用加载修改类失败, 则需要确认这个类中使用的所有类都是被 `javassist.Loader` 加载

#### 自定义类加载器
使用 Javassist 定义的简单类加载器如下
```
import javassist.*;

public class SampleLoader extends ClassLoader {
    /* Call MyApp.main().
     */
    public static void main(String[] args) throws Throwable {
        SampleLoader s = new SampleLoader();
        Class c = s.loadClass("MyApp");
        c.getDeclaredMethod("main", new Class[] { String[].class })
         .invoke(null, new Object[] { args });
    }

    private ClassPool pool;

    public SampleLoader() throws NotFoundException {
        pool = new ClassPool();
        pool.insertClassPath("./class"); // MyApp.class must be there.
    }

    /* Finds a specified class.
     * The bytecode for that class can be modified.
     */
    protected Class findClass(String name) throws ClassNotFoundException {
        try {
            CtClass cc = pool.get(name);
            // modify the CtClass object here
            byte[] b = cc.toBytecode();
            return defineClass(name, b, 0, b.length);
        } catch (NotFoundException e) {
            throw new ClassNotFoundException();
        } catch (IOException e) {
            throw new ClassNotFoundException();
        } catch (CannotCompileException e) {
            throw new ClassNotFoundException();
        }
    }
}
```
类 `MyApp` 是一个应用程序; 为了执行这个程序, 首先将类文件放在 `./class` 目录下, 此目录不包括在类的搜索路径下; 否则, `MyApp.class` 将会别默认的系统类加载器加载, 这是 `SampleLoader` 的父类加载器; 目录名 `./class` 在 `insertClassPath()` 中被指定; 如果你想, 你可以选择不同的目录来代替 `./class`; 然后启动
```
% java SampleLoader
```
类加载器会加载类 `MyApp(./class/MyApp.class)` 并且使用命令行参数调用 `MyApp.main()`  
这是使用 Javassist 定义类加载器的最简单的方式; 如果需要些更复杂的类加载器, 则需要了解 Java 类加载器机制的更多细节; 例如, 以上程序将 `MyApp` 类放置在与类 `SimpleLoader` 不同的命名空间, 因为这两个雷被不同的类加载器加载; 所以, `MyApp` 类不允许直接访问类  `SimpleLoader`

##### 修改系统类
系统类如 `java.lang.String` 不能被类加载器加载, 只能被系统类加载器加载; 因此, `SampleLoader` 或者 `javassist.Loader` 都不能在加载时修改系统类  
如果你的应用需要这样做, 那么系统类必须静态的被修改; 例如, 以下代码为 `java.lang.String` 添加一个 `hiddenValue` 的新域
```
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("java.lang.String");
CtField f = new CtField(CtClass.intType, "hiddenValue", cc);
f.setModifiers(Modifier.PUBLIC);
cc.addField(f);
cc.writeFile(".");
```
这个程序产生 `"./java/lang/String.class"` 文件  
然后运行使用这个修改 `String` 类 `MyApp`
```
% java -Xbootclasspath/p:. MyApp arg1 arg2...
```
假定 `MyApp` 德行定义如下
```
public class MyApp {
    public static void main(String[] args) throws Exception {
        System.out.println(String.class.getField("hiddenValue").getName());
    }
}
```
如果修改的 `String` 类被正确加载, `MyApp` 会打印出 `hiddenValue`
>**注意:**
使用此技术用于覆盖在 `rt.jar` 中的系统类的应用不应该被部署, 因为会违反 Java 2 运行环境二进制代码许可

#### 在运行时重新加载类
如果 JVM 启动时打开了 JPDA (Java Platform Debugger Architecture), 那么一个类可以动态的加载; 在 JVM 加载一个类后, 旧版本的类定义可以被卸载, 并且新的可以被重新加载; 这样, 类定义可以在运行时动态的修改; 然而, 新类定义必须与旧版兼容; JVM 不允许两个版本之间有模式改变, 它们必须由相同的方法和字段  
Javassist 为运行时重加载提供了便利类, 更多信息见 `javassist.tools.HotSwapper` API

>**参考:**
[Javassist Tutorial One](http://www.javassist.org/tutorial/tutorial.html)
