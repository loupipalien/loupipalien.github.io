---
layout: post
title: "使用内置 Jetty Container 的 Jersey 应用"
date: "2018-12-01"
description: "使用内置 Jetty Container 的 Jersey 应用"
tag: [java, jersey, jetty]
---

### 什么是 [Jersey](https://jersey.github.io/)
开发 RESTful Web 服务并无缝在各种表现层中展现你的数据与抽象客户端-服务端通信的底层细节, 在没有一个好工具的情况下不是一个简单任务; 为了简化使用 Java 开发 RESTful Web 服务和它们的客户端, 设计了标准的且轻量级的 [JAX-RS API](http://jax-rs-spec.java.net/); Jersey RESTful Web 服务框架是一个开源的, 生产级别的框架, 此框架是为了提供对 JAX-RS APIs 支持的, 使用 Java 开发的 RESTful Web 服务, 并作为 JAX-RS (JSR 311 & JSR 339) 的一个参考实现  
Jersey 框架比 JAX-RS 参考实现更强大, Jersey 提供了拓展 JAX-RS 工具的 [API](https://jersey.github.io/apidocs/latest/jersey/index.html), 提供了额外的特性和工具来更加简化 RESTful 服务和客户端的开发; Jersey 也暴露了一些可扩展的 SPIs, 供开发者扩展 Jersey 以满足自己的需要

#### Jersey 应用示例
```
@Path("message")
@Consumes({MediaType.APPLICATION_JSON, MediaType.TEXT_XML})
@Produces({MediaType.APPLICATION_JSON, MediaType.TEXT_XML})
public class Message {

    /**
     * 支持注入 org.eclipse.jetty.server.Request
     * 注意点: 在类上注解 @Singleton, 会使得所有请求都使用同一个 Request, 第一个请求的 Request 可以正确获取, 后续的请求的 Request 字段都会为 null
     * reference: https://stackoverflow.com/questions/29434312/is-it-possible-in-jersey-to-have-access-to-an-injected-httpservletrequest-inste
     */
    @Context
    private Request request;

    @GET
    @Path("get")
    public String getMessage() {
        return request.getRemoteAddr();
    }
}
```

#### 使用 Jetty Http Container
##### 依赖的 jar 包
```
<dependencies>
    <dependency>
        <groupId>org.glassfish.jersey.containers</groupId>
        <artifactId>jersey-container-jetty-http</artifactId>
        <version>2.27</version>
    </dependency>
    <dependency>
        <groupId>org.glassfish.jersey.inject</groupId>
        <artifactId>jersey-hk2</artifactId>
        <version>2.27</version>
    </dependency>
</dependencies>
```
###### 启动 Jetty Http Container
```
/**
 * reference: https://jersey.github.io/documentation/latest/deployment.html#deployment.http
 */
public class App {
    public static void main( String[] args ) {
        // 基础路径
        URI baseUri = UriBuilder.fromUri("http://localhost/").port(9998).build();
        // 配置提供 rest 接口的类
        ResourceConfig config = new ResourceConfig(Message.class);
        // 创建 Jetty Http Server
        Server server = JettyHttpContainerFactory.createServer(baseUri, config);
        try {
            server.start();
            server.join();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            server.destroy();
        }
    }
}
```
#### 使用 Jetty Servlet Container
##### 依赖的 jar 包
```
<dependencies>
    <dependency>
        <groupId>org.glassfish.jersey.containers</groupId>
        <artifactId>jersey-container-jetty-servlet</artifactId>
        <version>2.27</version>
    </dependency>
    <dependency>
        <groupId>org.glassfish.jersey.inject</groupId>
        <artifactId>jersey-hk2</artifactId>
        <version>2.27</version>
    </dependency>
</dependencies>
```
##### 启动 Jetty Http Container
```
/**
 * reference: http://zetcode.com/articles/jerseyembeddedjetty/
 */
public class App {
    public static void main( String[] args ) {
        // 新建 jetty server
        Server server = new Server(9999);
        // 设置 context path
        ServletContextHandler ctx = new ServletContextHandler(ServletContextHandler.NO_SESSIONS);
        ctx.setContextPath("/");
        server.setHandler(ctx);
        // 将那些路径加入 servlet container
        ServletHolder holder = ctx.addServlet(ServletContainer.class, "/*");
        holder.setInitOrder(1);
        // 设置在哪里找处理类
        holder.setInitParameter("jersey.config.server.provider.packages", "com.ltchen.demo.jersey.container.jetty.servlet");

        try {
            server.start();
            server.join();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            server.destroy();
        }
    }
}
```

> 参考:
[Jersey application with embedded Jetty](http://zetcode.com/articles/jerseyembeddedjetty/)  
[Java SE Deployment Environments](https://jersey.github.io/documentation/latest/deployment.html#deployment.http)  
[java轻量RESTful api服务搭建(jersey+jetty)](https://www.jianshu.com/p/7c4b11096553)
