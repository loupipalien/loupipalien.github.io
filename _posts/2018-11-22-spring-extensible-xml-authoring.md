---
layout: post
title: "Spring 编写扩展 xml"
date: "2018-11-22"
description: "Spring 编写扩展 xml"
tag: [spring]
---

### 介绍
从 2.0 开始, 为了在基本的 spring xml 格式中定义和配置 bean, spring 提供了一种基于模式的扩展机制; 遵循以下简单的几个步骤就可以创建一个新的 xml 配置扩展
- 编写一个 xml 模式描述你的自定义元素
- 编码一个自定义的 `NamespaceHandler` 实现
- 编码一个或多个 `BeanDefinitionParser` 实现
- 注册以上组件到 spring 中

例如, 可以创建一个 xml 扩展允许使用一种简单的方式配置 `SimpleDateFormat` 类型的对象; 当实现完就可以使用新的定义方式
```
<!-- 未自定义 xml 前定义一个日期的 bean -->
<bean id="dateFormat1" class="java.text.SimpleDateFormat">
    <constructor-arg value="yyyy-MM-dd HH:mm:ss"/>
    <property name="lenient" value="true"/>
</bean>

<!-- 自定义 xml 后定义一个日期的 bean -->
<myns:dateFormat id="dateFormat2" pattern="yyyy-MM-dd HH:mm:ss" lenient="true"/>
```

#### 编写一个模式
spring 的 IoC 容器启动时使用编写的 xml 模式描述这个扩展, 以下是用于配置 `SimpleDateFormat` 对象的模式
```
<?xml version="1.0" encoding="UTF-8"?>
<xsd:schema xmlns="http://www.ltchen.com/schema/myns"
            xmlns:xsd="http://www.w3.org/2001/XMLSchema"
            xmlns:beans="http://www.springframework.org/schema/beans"
            targetNamespace="http://www.ltchen.com/schema/myns"
            elementFormDefault="qualified">

    <xsd:import namespace="http://www.springframework.org/schema/beans"/>

    <xsd:element name="dateFormat">
        <xsd:complexType>
            <xsd:complexContent>
                <xsd:extension base="beans:identifiedType">
                    <xsd:attribute name="pattern" type="xsd:string" use="required"/>
                    <xsd:attribute name="lenient" type="xsd:boolean"/>
                </xsd:extension>
            </xsd:complexContent>
        </xsd:complexType>
    </xsd:element>
</xsd:schema>
```
xsd 文件如何定义可以参考 [W3C XML Schema ](http://www.w3school.com.cn/schema/schema_elements_ref.asp) 和 spring 文档 [ Extensible XML authoring](https://docs.spring.io/spring/docs/4.3.x/spring-framework-reference/html/xml-custom.html)

#### 编码一个 NamespaceHandler
除了以上定义的模式, 还需要一个 `NamespaceHandler` 解析 spring 在解析配置文件时的遇到的指定空间的所有元素, 这个 `NamespaceHandler` 需要关注 `myns:dateFormat` 元素的解析; `NamespaceHandler` 接口相当简单, 仅支持三个方法
- `init()`: 允许初始化这个 `NamespaceHandler`, spring 会在使用这个 handler 之前调用这个方法
- `BeanDefinition parse(Element, ParserContext)`: 当 spring 遇到一个顶级元素 (不是内嵌在 bean 定义或者其他空间) 时被调用, 这个方法注册 bean 定义本身并且返回一个 bean 定义
- `BeanDefinitionHolder decorate(Node, BeanDefinitionHolder, ParserContext)`: 当 spring 在遇到一个不同空间的属性或者内嵌元素时被调用

尽管可以为整个命名空间编码自定义的的 `NamespaceHandler` (提供解析整个命名空间中的元素), 而常见的情形是在 spring 配置文件中的顶级 xml 元素往往是单个 bean 定义 (例如在本次例子中, 单个 `<myns:dateformat/>` 元素是单个 bean 定义), spring 提供了一系列的便捷类支持这些场景
```
public class MyNamespaceHandler extends NamespaceHandlerSupport {

    @Override
    public void init() {
        registerBeanDefinitionParser("dateFormat", new SimpleDateFormatBeanDefinitionParser());
    }
}
```
`NamespaceHandlerSupport` 类內建代理的概念, 它支持注册一系列的 `BeanDefinitionParser` 实例, 用来代理解析在命名空间中需要解析的元素; 这样代理到 `BeanDefinitionParser` 中做这些解析 xml 解析的工作, 分离的概念允许一个 `NamespaceHandler` 处理在这个命名空间中编写的所有元素的解析; 这意味着每个 `BeanDefinitionParser` 仅包含解析单个自定义元素的逻辑

#### BeanDefinitionParser
`BeanDefinitionParser` 负责解析这个模式中定义的单个顶级 xml 元素, 在解析器中将会访问这些 xml 元素来解析自定义的 xml 内容
```
public class SimpleDateFormatBeanDefinitionParser extends AbstractSingleBeanDefinitionParser {

    @Override
    protected Class<?> getBeanClass(Element element) {
        return SimpleDateFormat.class;
    }

    @Override
    protected void doParse(Element element, BeanDefinitionBuilder builder) {
        //  this will never be null, since the schema explicitly requires that a value be supplied
        String pattern = element.getAttribute("pattern");
        builder.addConstructorArgValue(pattern);

        // this is an optional property
        String lenient  = element.getAttribute("lenient");
        if (StringUtils.hasText(lenient)) {
            builder.addPropertyValue("lenient", Boolean.valueOf(lenient));
        }
    }
}
```
以上实际要做的是通过 `AbstractSingleBeanDefinitionParser` 处理这个单个 `BeanDefinition` 的创建

#### 注册 handler 和 schema
完成了编码, 剩余要完成的就是如何让 spring xml 的解析组件知道自定义组件; 这个通过将自定义的 `NamespaceHandler` 和 自定义的 xsd 的文件注册到两个特定的属性文件中, 这些文件要放置在应用的 `META-INF` 的文件目录中, 和 Jar 文件中的二进制文件一起; spring xml 解析组件将通过消费这些特殊的属性文件自动获取新的扩展

##### 'META-INF/spring-handlers'
`spring-handlers` 包含 xml 模式的 url 和命名空间处理器类的
一个映射
```
http\://www.ltchen.com/schema/myns=com.ltchen.demo.spring.custom.xml.MyNamespaceHandler
```
url 需要与自定义的命名空间扩展联系起来, 需要精确匹配自定义 xsd 模式文件中的 `targetNamespace` 属性的值

##### 'META-INF/spring-schemas'
`spring-schemas` 包含 xml 模式位置 (即在模式声明 xml 文件中使用 `xsi:schemaLocation` 中引用的部分) 和资源在 classpath 中的路径的映射
```
http\://www.ltchen.com/schema/myns/myns.xsd=META-INF/myns.xsd
```
如果在这个文件中指定了映射, spring 将会在 classpath 中搜寻这个模式

####  在 spring xml 配置中使用自定义的扩展
使用实现的自定义的扩展和使用 spring 自身提供的 '自定义' 扩展并没有什么不同
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:myns="http://www.ltchen.com/schema/myns"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.ltchen.com/schema/myns http://www.ltchen.com/schema/myns/myns.xsd">

    <!-- 未自定义 xml 前定义一个日期的 bean -->
    <bean id="dateFormat1" class="java.text.SimpleDateFormat">
        <constructor-arg value="yyyy-MM-dd HH:mm:ss"/>
        <property name="lenient" value="true"/>
    </bean>

    <!-- 自定义 xml 后定义一个日期的 bean -->
    <myns:dateFormat id="dateFormat2" pattern="yyyy-MM-dd HH:mm:ss" lenient="true"/>
</beans>
```

>**参考:**
[Extensible XML authoring](https://docs.spring.io/spring/docs/4.3.x/spring-framework-reference/html/xml-custom.html)  
[Spring可扩展的XML Schema机制](https://www.jianshu.com/p/8639e5e9fba6)  
[XML Schema 参考手册](http://www.w3school.com.cn/schema/schema_elements_ref.asp)
