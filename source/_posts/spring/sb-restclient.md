---
title: "SpringBoot系列 - 使用RestTemplate"
date: 2017-07-06 18:29:52 +0800
comments: true
toc: true
categories: spring
tags: [springboot]
---

spring框架提供的RestTemplate类可用于在应用中调用rest服务，它简化了与http服务的通信方式，统一了RESTful的标准，封装了http链接，
我们只需要传入url及返回值类型即可。相较于之前常用的HttpClient，RestTemplate是一种更优雅的调用RESTful服务的方式。

RestTemplate默认依赖JDK提供http连接的能力（HttpURLConnection），如果有需要的话也可以通过setRequestFactory方法替换为例如
Apache HttpComponents、Netty或OkHttp等其它HTTP library。

本篇介绍如何使用RestTemplate，以及在SpringBoot里面的配置和注入。<!--more-->

## 实现逻辑

RestTemplate包含以下几个部分：

* HttpMessageConverter 对象转换器
* ClientHttpRequestFactory 默认是JDK的HttpURLConnection
* ResponseErrorHandler 异常处理
* ClientHttpRequestInterceptor 请求拦截器

用一张图可以很直观的理解：

![](https://xnstatic-1253397658.file.myqcloud.com/rest01.png)

直接使用方式很简单：

``` java
public class RestTemplateTest {
	public static void main(String[] args) {
		RestTemplate restT = new RestTemplate();
		//通过Jackson JSON processing library直接将返回值绑定到对象
		Quote quote = restT.getForObject("http://gturnquist-quoters.cfapps.io/api/random", Quote.class);
		String quoteString = restT.getForObject("http://gturnquist-quoters.cfapps.io/api/random", String.class);
		System.out.println(quoteString);
	}
}
```

## 发送GET请求

``` java
// 1-getForObject()
User user1 = this.restTemplate.getForObject(uri, User.class);

// 2-getForEntity()
ResponseEntity<User> responseEntity1 = this.restTemplate.getForEntity(uri, User.class);
HttpStatus statusCode = responseEntity1.getStatusCode();
HttpHeaders header = responseEntity1.getHeaders();
User user2 = responseEntity1.getBody();
  
// 3-exchange()
RequestEntity requestEntity = RequestEntity.get(new URI(uri)).build();
ResponseEntity<User> responseEntity2 = this.restTemplate.exchange(requestEntity, User.class);
User user3 = responseEntity2.getBody();
```

## 发送POST请求

``` java
// 1-postForObject()
User user1 = this.restTemplate.postForObject(uri, user, User.class);

// 2-postForEntity()
ResponseEntity<User> responseEntity1 = this.restTemplate.postForEntity(uri, user, User.class);

// 3-exchange()
RequestEntity<User> requestEntity = RequestEntity.post(new URI(uri)).body(user);
ResponseEntity<User> responseEntity2 = this.restTemplate.exchange(requestEntity, User.class);
```

## 设置HTTP Header

``` java
// 1-Content-Type
RequestEntity<User> requestEntity = RequestEntity
        .post(new URI(uri))
        .contentType(MediaType.APPLICATION_JSON)
        .body(user);

// 2-Accept
RequestEntity<User> requestEntity = RequestEntity
        .post(new URI(uri))
        .accept(MediaType.APPLICATION_JSON)
        .body(user);

// 3-Other
RequestEntity<User> requestEntity = RequestEntity
        .post(new URI(uri))
        .header("Authorization", "Basic " + base64Credentials)
        .body(user);
```

## 捕获异常

捕获`HttpServerErrorException`

``` java
try {
    responseEntity = restTemplate.exchange(requestEntity, String.class);
} catch (HttpServerErrorException e) {
    // log error
}
```

## 自定义异常处理器

``` java
public class CustomErrorHandler extends DefaultResponseErrorHandler {
    @Override
    public void handleError(ClientHttpResponse response) throws IOException {
        // todo
    }
}
```

然后设置下异常处理器：

``` java
@Configuration
public class RestClientConfig {
    @Bean
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.setErrorHandler(new CustomErrorHandler());
        return restTemplate;
    }
}
```

## 配置类

创建`RestClientConfig`类，设置连接池大小、超时时间、重试机制等。配置如下：

``` java
@Configuration
public class RestClientConfig {
    @Bean
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.setRequestFactory(clientHttpRequestFactory());
        restTemplate.setErrorHandler(new DefaultResponseErrorHandler());
        return restTemplate;
    }
    @Bean
    public HttpComponentsClientHttpRequestFactory clientHttpRequestFactory() {
        try {
            HttpClientBuilder httpClientBuilder = HttpClientBuilder.create();
            SSLContext sslContext = new SSLContextBuilder().loadTrustMaterial(null, new TrustStrategy() {
                public boolean isTrusted(X509Certificate[] arg0, String arg1) throws CertificateException {
                    return true;
                }
            }).build();
            httpClientBuilder.setSSLContext(sslContext);
            HostnameVerifier hostnameVerifier = NoopHostnameVerifier.INSTANCE;
            SSLConnectionSocketFactory sslConnectionSocketFactory = new SSLConnectionSocketFactory(sslContext, hostnameVerifier);
            Registry<ConnectionSocketFactory> socketFactoryRegistry = RegistryBuilder.<ConnectionSocketFactory>create()
                    .register("http", PlainConnectionSocketFactory.getSocketFactory())
                    .register("https", sslConnectionSocketFactory).build();// 注册http和https请求
            // 开始设置连接池
            PoolingHttpClientConnectionManager poolingHttpClientConnectionManager = new PoolingHttpClientConnectionManager(socketFactoryRegistry);
            poolingHttpClientConnectionManager.setMaxTotal(500); // 最大连接数500
            poolingHttpClientConnectionManager.setDefaultMaxPerRoute(100); // 同路由并发数100
            httpClientBuilder.setConnectionManager(poolingHttpClientConnectionManager);
            httpClientBuilder.setRetryHandler(new DefaultHttpRequestRetryHandler(3, true)); // 重试次数
            HttpClient httpClient = httpClientBuilder.build();
            HttpComponentsClientHttpRequestFactory clientHttpRequestFactory = new HttpComponentsClientHttpRequestFactory(httpClient); // httpClient连接配置
            clientHttpRequestFactory.setConnectTimeout(20000);              // 连接超时
            clientHttpRequestFactory.setReadTimeout(30000);                 // 数据读取超时时间
            clientHttpRequestFactory.setConnectionRequestTimeout(20000);    // 连接不够用的等待时间
            return clientHttpRequestFactory;
        } catch (KeyManagementException | NoSuchAlgorithmException | KeyStoreException e) {
            log.error("初始化HTTP连接池出错", e);
        }
        return null;
    }
}
```

注意，如果没有apache的HttpClient类，需要在pom文件中添加：

``` xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.3</version>
</dependency>
```

## 发送文件

``` java
MultiValueMap<String, Object> multiPartBody = new LinkedMultiValueMap<>();
multiPartBody.add("file", new ClassPathResource("/tmp/user.txt"));
RequestEntity<MultiValueMap<String, Object>> requestEntity = RequestEntity
        .post(uri)
        .contentType(MediaType.MULTIPART_FORM_DATA)
        .body(multiPartBody);
```

## 下载文件

``` java
// 小文件
RequestEntity requestEntity = RequestEntity.get(uri).build();
ResponseEntity<byte[]> responseEntity = restTemplate.exchange(requestEntity, byte[].class);
byte[] downloadContent = responseEntity.getBody();
  
// 大文件  
ResponseExtractor<ResponseEntity<File>> responseExtractor = new ResponseExtractor<ResponseEntity<File>>() {
    @Override
    public ResponseEntity<File> extractData(ClientHttpResponse response) throws IOException {
        File rcvFile = File.createTempFile("rcvFile", "zip");
        FileCopyUtils.copy(response.getBody(), new FileOutputStream(rcvFile));
        return ResponseEntity.status(response.getStatusCode()).headers(response.getHeaders()).body(rcvFile);
    }
};
File getFile = this.restTemplate.execute(targetUri, HttpMethod.GET, null, responseExtractor);
```

## Service注入

``` java
@Service
public class DeviceService {
    private static final Logger logger = LoggerFactory.getLogger(DeviceService.class);

    @Resource
    private RestTemplate restTemplate;
}
```

## 实际使用例子

``` java
// 开始推送消息
logger.info("解绑成功后推送消息给对应的POS机");
LoginParam loginParam = new LoginParam();
loginParam.setUsername(managerInfo.getUsername());
loginParam.setPassword(managerInfo.getPassword());
HttpBaseResponse r = restTemplate.postForObject(
        p.getPosapiUrlPrefix() + "/notifyLogin", loginParam, HttpBaseResponse.class);
if (r.isSuccess()) {
    logger.info("推送消息登录认证成功");
    String token = (String) r.getData();
    UnbindParam unbindParam = new UnbindParam();
    unbindParam.setImei(pos.getImei());
    unbindParam.setLocation(location);
    // 设置HTTP Header信息
    URI uri;
    try {
        uri = new URI(p.getPosapiUrlPrefix() + "/notify/unbind");
    } catch (URISyntaxException e) {
        logger.error("URI构建失败", e);
        return 1;
    }
    RequestEntity<UnbindParam> requestEntity = RequestEntity
            .post(uri)
            .contentType(MediaType.APPLICATION_JSON)
            .accept(MediaType.APPLICATION_JSON)
            .header("Authorization", token)
            .body(unbindParam);
    ResponseEntity<HttpBaseResponse> responseEntity = restTemplate.exchange(requestEntity, HttpBaseResponse.class);
    HttpBaseResponse r2 = responseEntity.getBody();
    if (r2.isSuccess()) {
        logger.info("推送消息解绑网点成功");
    } else {
        logger.error("推送消息解绑网点失败，errmsg = " + r2.getMsg());
    }
} else {
    logger.error("推送消息登录认证失败");
}
```

## GitHub源码

[springboot-resttemplate](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-resttemplate)

