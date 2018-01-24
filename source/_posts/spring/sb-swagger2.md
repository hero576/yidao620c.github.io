---
title: "SpringBoot系列 - 集成Swagger2"
date: 2017-07-07 15:26:19 +0800
comments: true
toc: true
categories: spring
tags: [springboot]
---

Swagger是一个简单但功能强大的API表达工具。它具有地球上最大的API工具生态系统，数以千计的开发人员，
使用几乎所有的现代编程语言，都在支持和使用Swagger。使用Swagger生成API，我们可以得到交互式文档，
自动生成代码的SDK以及API的发现特性等。

Swagger2可以利用注解快速、自动地生成接口文档页面，方便调用方查阅！

这一篇讲解如何在Spring Boot中集成Swagger2.<!--more-->

先来张效果图：

![](https://xnstatic-1253397658.file.myqcloud.com/sb-swagger01.png)

可以看到Swagger-Ui是以controller分类,点击一个controller可以看到其中的具体接口,再点击接口就可以看到接口的信息了,如图:

![](https://xnstatic-1253397658.file.myqcloud.com/sb-swagger02.png)

我们可以看到该接口的请求方式,返回数据信息和需要传递的参数.而且以上数据是自动生成的,即使代码有一些修改,
Swagger文档也会自动同步修改.非常的方便。

## 构建RESTful API

在使用Swagger2前我们需要有一个RESTful API的项目，Spring Boot创建RESTful API项目非常的方便和快速。

Spring Boot构建RESTful API极为简单，实际就是Spring MVC。

比如我的机具管理API如下，提供了3个接口：

``` java
@RestController
@RequestMapping(value = "/api/v1")
public class PublicApi {
    @Resource
    private ApiService apiService;
    private static final Logger _logger = LoggerFactory.getLogger(PublicApi.class);

    /**
     * 入网查询接口
     *
     * @return 是否入网
     */
    @RequestMapping(value = "/join", method = RequestMethod.GET)
    public BaseResponse join(@RequestParam("imei") String imei) {
        _logger.info("入网查询接口 start....");
        BaseResponse result = new BaseResponse();
        int posCount = apiService.selectCount(imei);
        result.setSuccess(true);
        result.setMsg("已经入网");
        return result;
    }

    /**
     * 请求入网接口
     *
     * @return 处理结果
     */
    @RequestMapping(value = "/join", method = RequestMethod.POST)
    public BaseResponse doJoin(@RequestBody PosParam posParam) {
        _logger.info("请求入网接口 start....");
        BaseResponse result = new BaseResponse();
        result.setSuccess(true);
        result.setMsg("入网成功");
        return result;
    }

    /**
     * 定时报告接口
     *
     * @return 报告结果
     */
    @RequestMapping(value = "/report", method = RequestMethod.POST)
    public BaseResponse report(@RequestBody ReportParam param) {
        _logger.info("定时报告接口 start....");
        BaseResponse result = new BaseResponse();
        int r = apiService.report(param);
        result.setSuccess(true);
        result.setMsg("报告成功");
        return result;
    }
}
```

## 添加Swagger2依赖

maven依赖：

``` xml
<!-- swagger2 集成-->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.7.0</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.7.0</version>
</dependency>
```

如果你赶紧默认的样式不好看，还可以将上面的`springfox-swagger-ui`依赖换成下面的：

``` xml
<dependency>
    <groupId>com.github.xiaoymin</groupId>
    <artifactId>swagger-bootstrap-ui</artifactId>
    <version>1.7.2</version>
</dependency>
```

## 创建Swagger2的Java配置类

通过`@Configuration`注解，表明它是一个配置类，`@EnableSwagger2` 注解开启swagger2。
`apiInfo()` 方法配置一些基本的信息。`createRestApi()` 方法指定扫描的包会生成文档，
默认是显示所有接口,可以用`@ApiIgnore`注解标识该接口不显示。

``` java
/**
 * Swagger2的Java配置类
 *
 * @author XiongNeng
 * @version 1.0
 * @since 2018/1/7
 */
@Configuration
@EnableSwagger2
public class Swagger2Config {

    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                // 指定controller存放的目录路径
                .apis(RequestHandlerSelectors.basePackage("com.enzhico.pos.api"))
                .paths(PathSelectors.ant("/api/v1/*"))
                .build();
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                // 文档标题
                .title("恩志科技API服务")
                // 文档描述
                .description("恩志科技对外API接口文档")
                // .termsOfServiceUrl("https://github.com/yidao620c")
                .version("v1")
                // .license("MIT 协议")
                // .licenseUrl("http://www.opensource.org/licenses/MIT")
                // .contact(new Contact("熊能","https://github.com/yidao620c","yidao620@gmail.com"))
                .build();
    }
}
```

## 添加Swagger2注解

通过在接口上面添加注解方式可配置丰富接口的信息，先看一个例子：

``` java
/**
 * 对外机具API类
 */
@Api(value = "对外机具API类", tags = "机具入网监控", description = "提供POS机的入网和监控的统一管理")
@ApiResponses(value = {
        @ApiResponse(code = 200, message = "Successful — 请求已完成"),
        @ApiResponse(code = 400, message = "请求中有语法问题，或不能满足请求"),
        @ApiResponse(code = 401, message = "未授权客户机访问数据"),
        @ApiResponse(code = 404, message = "服务器找不到给定的资源；文档不存在"),
        @ApiResponse(code = 500, message = "服务器不能完成请求")}
)
@RestController
@RequestMapping(value = "/api/v1")
public class PublicApi {

    @Resource
    private ApiService apiService;

    private static final Logger _logger = LoggerFactory.getLogger(PublicApi.class);

    /**
     * 入网查询接口
     *
     * @return 是否入网
     */
    @ApiOperation(value = "入网查询接口", notes = "根据IMEI码来查询该POS机是否入网", produces = "application/json")
    @ApiImplicitParam(name = "imei", value = "IMEI码", required = true, dataType = "String", paramType = "query", defaultValue = "1234555SHA")
    @RequestMapping(value = "/join", method = RequestMethod.GET)
    public BaseResponse join(@RequestParam("imei") String imei) {
        _logger.info("入网查询接口 start....");
        BaseResponse result = new BaseResponse();
        int posCount = apiService.selectCount(imei);
        if (posCount > 0) {
            result.setSuccess(true);
            result.setMsg("已经入网");
        } else {
            result.setSuccess(false);
            result.setMsg("没有入网");
        }
        return result;
    }
}
```

Swagger2提供了一些注解来丰富接口的信息,常用的有:

@ApiOperation：用在方法上，说明方法的作用

* value: 表示接口名称
* notes: 表示接口详细描述

@ApiImplicitParams：用在方法上包含一组参数说明

@ApiImplicitParam：用在@apiimplicitparams注解中，指定一个请求参数的各个方面

* paramType：参数位置
* header 对应注解：@requestheader
* query 对应注解：@requestparam
* path 对应注解: @pathvariable
* body 对应注解: @requestbody
* name：参数名
* dataType：参数类型
* required：参数是否必须传
* value：参数的描述
* defaultValue：参数的默认值

@ApiResponses：用于表示一组响应

@ApiResponse：用在@apiresponses中，一般用于表达一个错误的响应信息

* code：状态码
* message：返回自定义信息
* response：抛出异常的类

更多请参考[Swagger注解文档](http://docs.swagger.io/swagger-core/apidocs/com/wordnik/swagger/annotations/package-summary.html)

## 与Shiro集成配置

注意如果Spring Boot使用过Shiro或Spring Security框架，需要将相应的URL访问权限放开，以Shiro为例，添加匿名访问过滤器：

``` java
filterChainDefinitionMap.put("/api/v1/**", "anon");//API接口

// swagger接口文档
filterChainDefinitionMap.put("/v2/api-docs", "anon");
filterChainDefinitionMap.put("/webjars/**", "anon");
filterChainDefinitionMap.put("/swagger-resources/**", "anon");
filterChainDefinitionMap.put("/swagger-ui.html", "anon");
filterChainDefinitionMap.put("/doc.html", "anon");
```

## 访问SwaggerUI

访问`http://localhost:8080/swagger-ui.html`页面查看API文档

如果使用的是`swagger-bootstrap-ui`，请访问`http://localhost:8080/doc.html`

**参考文章**

* [Spring Boot中使用Swagger2构建API文档](https://github.com/Yuicon/blog/issues/1)
* [Spring Boot 04.构建RESTful API](https://www.jianshu.com/p/79e6f2dca03b)

