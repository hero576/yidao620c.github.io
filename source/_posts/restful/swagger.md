---
title: "使用Swagger生成RESTful API文档"
date: 2017-06-09 12:32:08 +0800
comments: true
toc: true
categories: restful
tags: [swagger]
---
REST API都是要对外提供服务的，那么文档是必须的。经常要给其他人员提供文档，每次都是要不断的维护word/excel的文件，挺麻烦的。
能不能做到自动生成呢？答案是可以的，swagger就是这样的一个组件帮助我们快速生成，
让开发人员只需要关注功能的开发即可，后续的工作就交给Swagger就好了。

Swagger是一个简单但功能强大的API表达工具。它具有地球上最大的API工具生态系统，数以千计的开发人员，
使用几乎所有的现代编程语言，都在支持和使用Swagger。使用Swagger生成API，我们可以得到交互式文档，
自动生成代码的SDK以及API的发现特性等。

2.X版本已经发布，Swagger变得更加强大。值得感激的是，Swagger的源码100%开源在[github](https://github.com/swagger-api)。

我演示的是为Jersey2自动生成API文档，更多的请参考[官方文档](http://swagger.io/docs/)。<!--more-->

使用Swagger不纯粹是为了生成一个漂亮的API文档，也不纯粹是为了自动生成多种语言的代码框架，
重要的是，通过遵循它的标准，可以使REST API分组清晰、定义标准。

通过Swagger生成API 文档有两种方式：

1. 通过代码注解来生成。好处：随时保持接口和文档的同步。坏处：代码入侵
2. 使用`Swagger Editor` 编写API文档的Yaml/Json定义。好处：不污染代码。坏处：不能实现文档和代码的实时同步

这里我采用第一种方式来说明，牺牲代码的纯净度来获取实时同步效果。

## pom.xml加入swagger的依赖
``` xml
<dependency>
    <groupId>io.swagger</groupId>
    <artifactId>swagger-jersey2-jaxrs</artifactId>
    <version>1.5.13</version>
</dependency>
```

## 修改资源配置类

(1) 修改自定义的`ResourceConfig`配置类，其中`com.enzhico.epay.action`是所有接口所在的包
``` java
public class ApplicationConfig extends ResourceConfig {

    public ApplicationConfig() {
        packages("com.enzhico.epay.action", "io.swagger.jaxrs.listing");

        register(ApiListingResource.class);
        register(SwaggerSerializers.class);

        BeanConfig config = new BeanConfig();
        config.setTitle("EPAY API");
        config.setVersion("1.0.0");
        config.setContact("Neng Xiong");
        config.setHost("localhost:8080");
        config.setBasePath("/api");
        config.setDescription("收费电子化项目API接口文档");
        config.setSchemes(new String[] { "http", "https" });
        config.setLicense("Apache Licence V2");
        config.setLicenseUrl("https://www.apache.org/licenses/LICENSE-2.0");
        config.setResourcePackage("com.enzhico.epay.action");
        config.setPrettyPrint(true);
        config.setScan(true);

        register(JacksonFeature.class);
        register(MultiPartFeature.class);

        register(JsonProvider.class);
        register(JacksonJsonProvider.class);

        register(AuthenticationFilter.class);
        register(CORSResponseFilter.class);
        //register(RequestContextFilter.class);  // Though it might be needed. Guess not

        property(ServerProperties.METAINF_SERVICES_LOOKUP_DISABLE, true);
    }
}
```

## 配置注解
所有的Actin类都必须添加`API`的注解，另外所有接口方法都要编写Swagger相应的注解，具体的编写方法参考Swagger Core，
种类我写一个示例：

``` java
/**
 * User Action
 *
 * @author XiongNeng
 * @version 1.0
 * @since 2017/6/12
 */
@Component
@Api(value = "users", description = "用户操作相关的API", tags = "users, mytag")
@Path("/v1")
public class UserAction {

    private static final Logger _logger = LoggerFactory.getLogger(UserAction.class);

    @Autowired
    private UserService userService;

    /**
     * 查询用户列表
     * @return
     * @throws AppException
     */
    @GET
    @Path("users")
    @Produces(MediaType.APPLICATION_JSON)
    @Authentication
    @ApiOperation(value="获取用户列表", notes="获取用户列表的详细说明")
    public List<User> getUsers() throws AppException {
        _logger.info("111111111111111111111 getUsers()");
        return userService.queryUsers();
    }
}
```

## 配置swagger-ui
有两种方式可以很直观优雅的查看这个swagger.json接口定义文件，第一种是使用swagger-ui。

先下载swagger-ui：
```
https://github.com/swagger-api/swagger-ui
```

拷贝dist目录下面的文件到webroot下面（和WEB-INF同级目录），然后修改`index.html`中的url地址：
```
url = "http://localhost:8080/api/swagger.json";
```

最后生成的文档效果如下：

![](https://xnstatic-1253397658.file.myqcloud.com/swagger01.png)

还有一种方法是使用`swagger editor`，将`swagger.json`或`swagger.yaml`导入即可。

## API开发规约

使用swagger2提供的swagger core功能，通过代码注解的方式自动生成api文档。
后端工程师在编写代码的同时即可完成接口文档的编写。

1. 后端工程师定义model，并用swagger annonation注解
2. 后端工程师定义restful接口，不需实现，并用swagger annonation注解
3. 后端工程师导出swagger接口文件分发给前端工程师，作为接口文档
4. 前端工程师根据接口文档行开发，可以将接口文档导入到swagger工具(swagger editor / swagger hub)中。
5. 前端工程师可以通过`swagger codegen`（集成在`swagger editor`中）生成server stubs来测试前端，
也可以使用swagger hub集成的virtserver（收费）来在线测试接口。
6. 如果开发过程中接口变更，后端工程师重新导出接口文档，并分发给前端工程师使用。

## swagger注解使用

* [简单例子](https://jakubstas.com/spring-jersey-swagger-create-documentation/#.WMj-khJ95E4)
* [官方接口文档](http://docs.swagger.io/swagger-core/current/apidocs/index.html)

