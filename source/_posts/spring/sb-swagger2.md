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

如果你感觉默认的样式不好看，还可以将上面的`springfox-swagger-ui`依赖换成我修改过的：

``` xml
<dependency>
    <groupId>com.xncoding</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.7.0</version>
</dependency>
```

github地址：<https://github.com/yidao620c/springfox-swagger-ui>

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
                .produces(Sets.newHashSet("application/json"))
                .consumes(Sets.newHashSet("application/json"))
                .protocols(Sets.newHashSet("http", "https"))
                .apiInfo(apiInfo())
                .forCodeGeneration(true)
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

说明：

* @Api：用在类上，说明该类的作用
* @ApiOperation：用在方法上，说明方法的作用
* @ApiImplicitParams：用在方法上包含一组参数说明
* @ApiImplicitParam：用在@ApiImplicitParams注解中，指定一个请求参数的各个方面
    - paramType：参数放在哪个地方
        * header-->请求参数的获取：@RequestHeader
        * query-->请求参数的获取：@RequestParam
        * path（用于restful接口）-->请求参数的获取：@PathVariable
        * body（不常用）
        * form（不常用）
    - name：参数名
    - dataType：参数类型
    - required：参数是否必须传
    - value：参数的意思
    - defaultValue：参数的默认值
* @ApiResponses：用于表示一组响应
* @ApiResponse：用在@ApiResponses中，一般用于表达一个错误的响应信息
    - code：数字，例如400
    - message：信息，例如"请求参数没填好"
    - response：抛出异常的类
* @ApiModel：描述一个Model的信息（这种一般用在post创建的时候，使用@RequestBody这样的场景，请求参数无法使用@ApiImplicitParam注解进行描述的时候）
* @ApiModelProperty：描述一个model的属性

以上这些就是最常用的几个注解了。

具体其他的注解，查看：

<https://github.com/swagger-api/swagger-core/wiki/Annotations#apimodel>

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

## 生成PDF文档

参考我的另一篇文章 [使用Swagger生成RESTful API文档](https://www.xncoding.com/2017/06/09/restful/swagger.html)

这里我通过SpringBoot + Swagger2方式来生成AsciiDoc，然后剩下的步骤和上面博客一样。

### maven依赖

``` xml
<dependency>
    <groupId>org.pegdown</groupId>
    <artifactId>pegdown</artifactId>
    <version>1.6.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.github.swagger2markup</groupId>
    <artifactId>swagger2markup</artifactId>
    <version>1.3.1</version>
    <scope>test</scope>
</dependency>
```

另外修改下surefire插件，增加2个系统属性，也就是swagger.json和adoc文件生成的位置：

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>2.20</version>
    <configuration>
        <systemPropertyVariables>
            <swaggerOutputDir>${project.basedir}/src/main/resources/swagger</swaggerOutputDir>
            <asciiDocOutputDir>${project.basedir}/src/main/resources/swagger/swagger</asciiDocOutputDir>
        </systemPropertyVariables>
        <skip>true</skip>
    </configuration>
</plugin>
```

### 编写单元测试方法

原理是通过SpringBoot的MockMvc启动后访问`/v2/api-docs`，这个是Swagger的接口数据，然后保存为`swagger.json`，

``` java
@AutoConfigureMockMvc
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class, webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class Swagger2MarkupTest {
    @Autowired
    private MockMvc mockMvc;

    private static final Logger LOG = LoggerFactory.getLogger(Swagger2MarkupTest.class);

    @Test
    public void createSpringFoxSwaggerJson() throws Exception {
//        String outputDir = System.getProperty("swaggerOutputDir"); // mvn test
        MvcResult mvcResult = this.mockMvc.perform(get("/v2/api-docs")
                .accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andReturn();
        MockHttpServletResponse response = mvcResult.getResponse();
        String swaggerJson = response.getContentAsString();
//        Files.createDirectories(Paths.get(outputDir));
//        try (BufferedWriter writer = Files.newBufferedWriter(Paths.get(outputDir, "swagger.json"), StandardCharsets.UTF_8)){
//            writer.write(swaggerJson);
//        }
        LOG.info("--------------------swaggerJson create --------------------");
        convertAsciidoc(swaggerJson);
        LOG.info("--------------------swagon.json to asciiDoc finished --------------------");
    }

    /**
     * 将swagger.yaml或swagger.json转换成漂亮的 AsciiDoc
     * 访问：http://localhost:9095/v2/api-docs
     * 将页面结果保存为src/main/resources/swagger.json
     */
    private void convertAsciidoc(String swaggerStr) {
//        Path localSwaggerFile = Paths.get(System.getProperty("swaggerOutputDir"), "swagger.json");
        Path outputFile = Paths.get(System.getProperty("asciiDocOutputDir"));
        Swagger2MarkupConfig config = new Swagger2MarkupConfigBuilder()
                .withMarkupLanguage(MarkupLanguage.ASCIIDOC)
                .withOutputLanguage(Language.ZH)
                .withPathsGroupedBy(GroupBy.TAGS)
                .withGeneratedExamples()
                .withoutInlineSchema()
                .build();
        Swagger2MarkupConverter converter = Swagger2MarkupConverter.from(swaggerStr)
                .withConfig(config)
                .build();
        converter.toFile(outputFile);
    }
}
```

执行之后会在`resources/swagger/`下面生成`swagger.adoc`

后面就参考上面的博客，将adoc文件转换成好看的PDF。

![](https://xnstatic-1253397658.file.myqcloud.com/swagger07.png)

**参考文章**

* [Spring Boot中使用Swagger2构建API文档](https://github.com/Yuicon/blog/issues/1)
* [Spring Boot 04.构建RESTful API](https://www.jianshu.com/p/79e6f2dca03b)

