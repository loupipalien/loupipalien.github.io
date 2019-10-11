---
layout: post
title: "spring 整合 swagger"
date: "2019-03-26"
description: "spring 整合 swagger"
tag: [spring, swagger]
---

### 为什么要使用 swagger
在项目开发过程中后台往往要提供 rest api 给外部调用, 写完接口在去写文档, 这样的工作量太多且重复, 当接口有修改时还要再次修改文档, 后续这样很容易造成接口和文档不统一 (吐槽: 其实并不反对写接口文档, 但文档的属性是静态的, 并不适用于描述持续变化的事物)

#### 使用 swagger
首先导入 swagger 的依赖 (版本 2 的)
```
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.9.2</version>
</dependency>
```
然后创建一个 Docket 的 bean 配置注册到 spring 中即可, 无论是使用 xml 配置还是注解配置, [springfox-demo](https://github.com/springfox/springfox-demos) 中都有示例; 注解配置示例如下
```
@Configuration
@EnableSwagger2
public class SwaggerConfig {

    @Bean
    public Docket api() {
        return new Docket(DocumentationType.SWAGGER_2)
                .select()
                .apis(RequestHandlerSelectors.any())
                .build()
                .apiInfo(apiInfo());
    }

    private static ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("Example 对外开放 API 文档")
                .description("Example 对外开放 API")
                .version("V1.0")
                .termsOfServiceUrl("http://www.example.com")
                .license("LICENSE")
                .licenseUrl("http://www.example.com")
                .build();
    }

}
```
启动项目后, 访问 `http://${ip}:${port}/${context}/v2/api-docs` 即可, 如果想自定义这个路径见 [这里](https://springfox.github.io/springfox/docs/current/#customizing-the-swagger-endpoints)
#### 使用 swagger-ui
swagger 还提供了一个 swagger-ui, 将上述页面的 json 格式的接口组织成可读性更好的页面文档; 首先还是要引入 swagger-ui 的 jar 包
```
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.9.2</version>
</dependency>
```
如果你使用 Spring Boot 开发, 只要将依赖引入, 启动应用后直接访问 `http://${ip}:${port}/swagger-ui.html` 即可  
如果你使用 SpringMVC 开发, 将依赖引入后还需在 xml 或注解中配置页面资源地址, [springfox-demo](https://github.com/springfox/springfox-demos) 中都有示例; 以下示例是 xml 配置
```
<beans>
...
    <mvc:resources mapping="swagger-ui.html" location="classpath:/META-INF/resources/"/>
    <mvc:resources mapping="/webjars/**" location="classpath:/META-INF/resources/webjars/"/>
...
</beans>
```

#### 后续
在接口上使用 swagger 注解标注出接口信息, 详细使用见参考文档

>**参考:**
[一步步完成Maven+SpringMVC+SpringFox+Swagger整合示例](https://my.oschina.net/wangmengjun/blog/907679)  
[Setting Up Swagger 2 with a Spring REST API](https://www.baeldung.com/swagger-2-documentation-for-spring-rest-api)
[Springfox Reference Documentation](https://springfox.github.io/springfox/docs/current)  
[springfox-demos](https://github.com/springfox/springfox-demos)
