---
title: "SpringBoot系列 - 缓存"
date: 2017-07-28 20:12:33 +0800
comments: true
toc: true
categories: spring
tags: [springboot]
---

内存的速度远远大于硬盘的速度，当我们需要重复获取相同的数据的时候，一次又一次的请求数据库或远程服务，
导致大量时间都消耗在数据库查询或远程方法调用上面，性能下降，这时候就需要使用到缓存技术了。

本文介绍SpringBoot 如何使用redis做缓存，如何对redis缓存进行定制化配置(如key的有效期)以及初始化redis做缓存。
使用具体的代码介绍了@Cacheable，@CacheEvict，@CachePut，@CacheConfig等注解及其属性的用法。<!--more-->

## Spring缓存支持

Spring定义了`org.springframework.cache.CacheManager` 和 `org.springframework.cache.Cache` 接口来统一不同缓存技术。
其中CacheManager是Spring提供的各种缓存技术抽象接口，内部使用Cache接口进行缓存的增删改查操作，我们一般不会直接和Cache打交道。

针对不同的缓存技术，Spring有不同的CacheManager实现类，定义如下表：

CacheManager                   | 描述
-------------------------------|--------------------------------------------------------------------
SimpleCacheManager             | 使用简单的Collection存储缓存数据，用来做测试用
ConcurrentMapCacheManager      | 使用ConcurrentMap存储缓存数据
EhCacheCacheManager            | 使用EhCache作为缓存技术
GuavaCacheManager              | 使用Google Guava的GuavaCache作为缓存技术
JCacheCacheManager             | 使用JCache(JSR-107)标准的实现作为缓存技术，比如Apache Commons JCS
RedisCacheManager              | 使用Redis作为缓存技术

在我们使用任意一个实现的CacheManager的时候，需要注册实现Bean：

``` java
/**
 * EhCache的配置
 */
@Bean
public EhCacheCacheManager cacheManager(CacheManager cacheManager) {
    return new EhCacheCacheManager(cacheManager);
}
```

当然，各种缓存技术都有很多其他配置，但是配置cacheManager是必不可少的。

## 声明式缓存注解

Spring提供4个注解来声明缓存规则，如下表所示：

注解           | 说明
---------------|-------------------------
@Cacheable     | 方法执行前先看缓存中是否有数据，如果有直接返回。如果没有就调用方法，并将方法返回值放入缓存
@CachePut      | 无论怎样都会执行方法，并将方法返回值放入缓存
@CacheEvict    | 将数据从缓存中删除
@Caching       | 可通过此注解组合多个注解策略在一个方法上面

@Cacheable 、@CachePut 、@CacheEvict都有value属性，指定要使用的缓存名称，而key属性指定缓存中存储的键。

## 集成Redis缓存

接下来将讲解如何集成redis来实现缓存。

### 安装redis

安装和配置redis服务器网上很多教程，这里就不多讲了。在linux服务器上面安装一个redis，启动后端口号为默认的6379。

### 添加maven依赖

``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### 配置application.yml

* 指定缓存的类型
* 配置redis的服务器信息

``` yml
spring:
  profiles: dev
  cache:
    type: REDIS
  redis:
    host: 123.207.66.156
    port: 6379
    timeout: 0
    database: 0
    pool:
      max-active: 100
      max-wait: -1
      max-idle: 8
      min-idle: 0
```

### 缓存配置类

重新配置`RedisCacheManager`，使用新的自定义配置值：

``` java
@Configuration
@EnableCaching
public class RedisCacheConfig {
    /**
     * 重新配置RedisCacheManager
     */
    @Autowired
    public void configRedisCacheManger(RedisCacheManager rd) {
        rd.setDefaultExpiration(100L);
    }
}
```

### keyGenerator

一般来讲我们使用key属性就可以满足大部分要求，但是如果你还想更好的自定义key，可以实现keyGenerator。

这个属性为定义key生成的类，和key属性不能同时存在。

在`RedisCacheConfig`配置类中添加我自定义的KeyGenerator：

``` java
/**
 * 自定义缓存key的生成类实现
 */
@Bean(name = "myKeyGenerator")
public KeyGenerator myKeyGenerator() {
    return new KeyGenerator() {
        @Override
        public Object generate(Object o, Method method, Object... params) {
            logger.info("自定义缓存，使用第一参数作为缓存key，params = " + Arrays.toString(params));
            // 仅仅用于测试，实际不可能这么写
            return params[0];
        }
    };
}
```

经过以上配置后，redis缓存管理对象已经生成。下面简单介绍如何使用。

### 使用

在service中定义增删改的几个常见方法，通过注解实现缓存：

``` java
@Service
@CacheConfig(cacheNames="users")
public class UserService {
    private Logger logger = LoggerFactory.getLogger(this.getClass());
    @Resource
    private UserMapper userMapper;

    /**
     * cacheNames 设置缓存的值
     * key：指定缓存的key，这是指参数id值。 key可以使用spEl表达式
     * @param id
     * @return
     */
    @Cacheable(cacheNames="user1", key="#id")
    public User getById(int id) {
        logger.info("获取用户start...");
        return userMapper.selectById(id);
    }

    /***
     * 如果设置sync=true，
     * 如果缓存中没有数据，多个线程同时访问这个方法，则只有一个方法会执行到方法，其它方法需要等待
     * 如果缓存中已经有数据，则多个线程可以同时从缓存中获取数据
     * @param id
     * @return
     */
    @Cacheable(cacheNames="user1", key="#id", sync = true)
    public User getById2(int id) {
        logger.info("获取用户start...");
        return userMapper.selectById(id);
    }
    
    /**
     * 以上我们使用默认的keyGenerator，对应spring的SimpleKeyGenerator
     * 如果你的使用很复杂，我们也可以自定义myKeyGenerator的生成key
     * <p>
     * key和keyGenerator是互斥，如果同时制定会出异常
     * The key and keyGenerator parameters are mutually exclusive and an operation specifying both will result in an exception.
     *
     * @param id
     * @return
     */
    @Cacheable(cacheNames = "user1", keyGenerator = "myKeyGenerator")
    public User queryUserById(int id) {
        logger.info("queryUserById,id={}", id);
        return userMapper.selectById(id);
    }

    /**
     * 每次执行都会执行方法，同时使用新的返回值的替换缓存中的值
     * @param user
     */
    @CachePut(cacheNames="user1", key="#user.id")
    public void createUser(User user) {
        logger.info("创建用户start...");
        userMapper.insert(user);
    }

    /**
     * 每次执行都会执行方法，同时使用新的返回值的替换缓存中的值
     * @param user
     */
    @CachePut(cacheNames="user1", key="#user.id")
    public void updateUser(User user) {
        logger.info("更新用户start...");
        userMapper.updateById(user);
    }

    /**
     * 对符合key条件的记录从缓存中user1移除
     */
    @CacheEvict(cacheNames="user1", key="#id")
    public void deleteById(int id) {
        logger.info("删除用户start...");
        userMapper.deleteById(id);
    }

    /**
     * allEntries = true: 清空user1里的所有缓存
     */
    @CacheEvict(cacheNames="user1", allEntries=true)
    public void clearUser1All(){
        logger.info("clearAll");
    }
}
```

注意可以在类上面通过`@CacheConfig`配置全局缓存名称，方法上面如果也配置了就会覆盖。

然后写个测试类：

``` java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
public class UserServiceTest {
    @Autowired
    private UserService userService;
    @Test
    public void testCache() {
        int id = new Random().nextInt(100);
        User user = new User(id, "admin", "admin");
        userService.createUser(user);
        User user1 = userService.getById(id); // 第1次访问
        assertEquals(user1.getPassword(), "admin");
        User user2 = userService.getById(id); // 第2次访问
        assertEquals(user2.getPassword(), "admin");
        User user3 = userService.queryUserById(id); // 第3次访问，使用自定义的KeyGenerator
        assertEquals(user3.getPassword(), "admin");
        user.setPassword("123456");
        userService.updateUser(user);
        User user4 = userService.getById(id); // 第4次访问
        assertEquals(user4.getPassword(), "123456");
        userService.deleteById(id);
        assertNull(userService.getById(id));
    }
}
```

下面是测试的打印日志一部分：

```
Started UserServiceTest in 12.919 seconds (JVM runni
创建用户start...
==>  Preparing: INSERT INTO t_user ( id, username, `
==> Parameters: 14(Integer), admin(String), admin(St
<==    Updates: 1
获取用户start...
==>  Preparing: SELECT id AS id,username,`password` 
==> Parameters: 14(Integer)
<==      Total: 1
自定义缓存，使用第一参数作为缓存key，params = [14]
更新用户start...
==>  Preparing: UPDATE t_user SET username=?, `passw
==> Parameters: admin(String), 123456(String), 14(In
<==    Updates: 1
获取用户start...
==>  Preparing: SELECT id AS id,username,`password` 
==> Parameters: 14(Integer)
<==      Total: 1
删除用户start...
==>  Preparing: DELETE FROM t_user WHERE id=? 
==> Parameters: 14(Integer)
<==    Updates: 1
获取用户start...
==>  Preparing: SELECT id AS id,username,`password` 
==> Parameters: 14(Integer)
<==      Total: 0
```

可以看到，第2次、第3次获取的时候并没有执行方法，说明缓存生效了。后面更新会同时更新缓存，取出来的也是更新后的数据。

## 切换缓存技术

得益于SpringBoot的自动配置机制，切换缓存技术除了替换相关maven依赖包和配置Bean外，使用方式和实例中一样，
不需要修改业务代码。如果你要切换到其他缓存技术非常简单。

**EhCache**

当我们需要使用EhCache作为缓存技术的时候，只需要在pom.xml中添加EhCache的依赖：

``` xml
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcahe</artifactId>
</dependency>
```

EhCache的配置文件ehcache.xml只需要放到类路径下面，SpringBoot会自动扫描，例如：

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd"
         updateCheck="false" monitoring="autodetect"
         dynamicConfig="true">

    <diskStore path="java.io.tmpdir/ehcache"/>

    <defaultCache
            maxElementsInMemory="50000"
            eternal="false"
            timeToIdleSeconds="3600"
            timeToLiveSeconds="3600"
            overflowToDisk="true"
            diskPersistent="false"
            diskExpiryThreadIntervalSeconds="120"
    />

    <cache name="authorizationCache"
           maxEntriesLocalHeap="2000"
           eternal="false"
           timeToIdleSeconds="3600"
           timeToLiveSeconds="3600"
           overflowToDisk="false"
           statistics="true">
    </cache>
</ehcache>
```

SpringBoot会为我们自动配置`EhCacheCacheManager`这个Bean，不过你也可以自己定义。

**Guava**

当我们需要Guava作为缓存技术的时候，只需要在pom.xml中增加Guava的依赖即可：

``` xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>18.0</version>
</dependency>
```

SpringBoot会为我们自动配置`GuavaCacheManager`这个Bean。

**Redis**

最后还提一点，本篇采用Redis作为缓存技术，添加了依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

SpringBoot会为我们自动配置`RedisCacheManager`这个Bean，同时还会配置`RedisTemplate`这个Bean。
后面这个Bean就是下一篇要讲解的操作Redis数据库用，这个就比单纯注解缓存强大和灵活的多了。

## GitHub源码

[springboot-cache](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-cache)

