---
layout: post
title: "Disconf 示例"
date: "2019-06-13"
description: "Disconf 示例"
tag: [java]
---

### Disconf 示例
在项目中引入 disconf, 完成以下几步即可

#### 在项目依赖中加入 `disconf-client` 依赖
```
<dependency>
    <groupId>com.baidu.disconf</groupId>
    <artifactId>disconf-client</artifactId>
    <version>2.6.36</version>
</dependency>
```

#### 在 disconf-web 中新建 APP 以及配置文件
创建的 APP 名和配置文件所在环境会在 `disconf.properties` 文件中使用  
![image](https://ws1.sinaimg.cn/large/d8f31fa4ly1g4kl5bnp13j20yf0h13zx.jpg)

#### 在 classpath 路径下添加 `disconf.properties` 文件
```
# 是否使用远程配置文件: true (默认) 会从远程获取配置, false 则直接获取本地配置
disconf.enable.remote.conf=true
# 配置服务器的 HOST, 用逗号分隔 127.0.0.1:8000, 127.0.0.1:8000
disconf.conf_server_host=192.168.127.100:80
# 版本, 请采用 X_X_X_X 格式
disconf.version=1_0_0_0
# APP 请采用 产品线_服务名 格式
disconf.app=spring-boot_demo-disconf
# 环境
disconf.env=rd
# debug
disconf.debug=true
# 忽略哪些分布式配置，用逗号分隔
disconf.ignore=
# 获取远程配置重试次数, 默认是 3 次
disconf.conf_server_url_retry_times=1
# 获取远程配置重试时休眠时间, 默认是 5 秒
disconf.conf_server_url_retry_sleep_seconds=1
# 用户定义的下载文件夹, 远程文件下载后会放在这里
disconf.user_define_download_dir=./disconf/download
# 下载的文件会被迁移到classpath根路径下，强烈建议将此选项置为 true (默认是 true)
disconf.enable_local_download_dir_in_class_path=true
```
更多配置了解见 [这里](https://disconf.readthedocs.io/zh_CN/latest/config/src/client-config.html#disconf-client)

#### 使用 xml 或注解配置
disconf 支持 xml 和注解配置两种方式; xml 方式除了支持自动 reload 方式还支持非自动 reload, 非自动 reload 即从 disconf-web 中下载配置文件但不热更新到应用中  
无论是使用 xml 还是注解, 都需要配置以下两个 bean (不想使用 xml 配置可以用 java 配置)
```
<!-- 使用 disconf 必须添加以下配置 -->
<bean id="disconfMgrBean" class="com.baidu.disconf.client.DisconfMgrBean" destroy-method="destroy">
    <property name="scanPackage" value="com.sf.demo.disconf"/>
</bean>
<bean id="disconfMgrBeanSecond" class="com.baidu.disconf.client.DisconfMgrBeanSecond"
      init-method="init" destroy-method="destroy">
</bean>
```
##### 使用 xml 配置
```
<!-- <bean id="reloadingPropertyPlaceholderConfigurer" class="com.baidu.disconf.client.addons.properties.ReloadingPropertyPlaceholderConfigurer"> -->
<bean id="propertyPlaceholderConfigurer" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="ignoreResourceNotFound" value="true"/>
    <property name="ignoreUnresolvablePlaceholders" value="true"/>
    <property name="propertiesArray">
        <list>
            <ref bean="reloadablePropertiesFactoryBean"/>
        </list>
    </property>
</bean>

<!-- 使用托管方式的 disconf 配置 (无代码侵入, 配置更改会自动 reload) -->
<bean id="reloadablePropertiesFactoryBean" class="com.baidu.disconf.client.addons.properties.ReloadablePropertiesFactoryBean">
    <property name="locations">
        <list>
            <value>classpath:redis.properties</value>
        </list>
    </property>
</bean>

<!-- 这里可以在类中使用 @Value 注解替换, 但 @Value 只在 Spring 启动时注入一次, 所以使用 xml 方式 + @Value 注解不支持自动 reload -->
<bean id="redisConfig" class="com.sf.demo.disconf.JedisConfig">
    <property name="host" value="${redis.host}"/>
    <property name="port" value="${redis.port}"/>
</bean>
```
`ReloadingPropertyPlaceholderConfigurer` 为 disconf 的类, 支持配置更新后 APP 自动 reload

##### 使用注解
```
@Service
@DisconfFile(filename = "redis.properties")
public class JedisConfig {

    private String host;
    private Integer port;

    @DisconfFileItem(name = "redis.host", associateField = "host")
    public String getHost() {
        return host;
    }

    public void setHost(String host) {
        this.host = host;
    }

    @DisconfFileItem(name = "redis.port", associateField = "port")
    public Integer getPort() {
        return port;
    }

    public void setPort(Integer port) {
        this.port = port;
    }

}

```
以上是一个使用 disconf 注解的类, 注解 `@DisconfFile` 和 `@DisconfFileItem` 分别关注配置文件和配置条目， 当 disconf-web 中对应的值变化时会重新下载配置文件并自动 reload (如果使用 lombok　的注解生成　getter/setter 方法如何使用 `@DisconfFileItem` 注解?)

>**参考:**  
[disconf-client turorial](https://disconf.readthedocs.io/zh_CN/latest/tutorial-client/index.html)  
[分布式配置中心 Disconf实践- 使用篇](http://www.dczou.com/viemall/758.html)
[disconf实践（二）基于XML的分布式配置文件管理，不会自动reload](https://www.cnblogs.com/warehouse/p/6883658.html)  
[disconf实践（三）基于XML的分布式配置文件管理，自动reload](https://www.cnblogs.com/warehouse/p/6885444.html)  
[disconf实践（四）基于注解的分布式配置文件管理，自动reload](https://www.cnblogs.com/warehouse/p/6885483.html)
