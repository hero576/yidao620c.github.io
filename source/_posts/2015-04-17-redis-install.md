---
layout: post
title: "redis入门与安装"
date: 2015-04-17 16:44:16 +0800
categories: 技术随笔
tags: redis
---

## 一 Redis介绍

Redis是一个开源的使用ANSI C语言编写、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API。从2010年3月15日起，Redis的开发工作由VMware主持。

Redis能运行在大多数POSIX(Linux, *BSD, OS X 和Solaris等)系统上，官方没有支持Windows的版本。目前最新的版本是2.2.11，这个版本主要是修复了一个2.2.7版本中遍历方式优化带来的一个bug。

和普通的Key-Value结构不同，Redis的Key支持灵活的数据结构，除了strings，还有hashes、lists、 sets 和sorted sets等结构。正是这些灵活的数据结构，丰富了Redis的应用场景，能满足更多业务上的灵活存储需求。

Redis的数据都保存在内存中，而且底层实现上是自己写了epoll event loop部分，而没有采用开源的libevent等通用框架，所以读写效率很高。
为了实现数据的持久化，Redis支持定期刷新(可通过配置实现)或写日志的方式来保存数据到磁盘。

**1\. 数据类型**

作为Key-value型数据库，Redis也提供了键(Key)和键值(Value)的映射关系。但是，除了常规的数值或字符串，Redis的键值还可以是以下形式之一：

- Lists (列表)
- Sets (集合)
- Sorted sets (有序集合)
- Hashes (哈希表)<!--more-->

键值的数据类型决定了该键值支持的操作。Redis支持诸如列表、集合或有序集合的交集、并集、查集等高级原子操作;同时，如果键值的类型是普通数字，Redis则提供自增等原子操作。

**2\. 持久化**

通常，Redis将数据存储于内存中，或被配置为使用虚拟内存。通过两种方式可以实现数据持久化：使用截图的方式，
将内存中的数据不断写入磁盘;或使用类似MySQL的日志方式，记录每次更新的日志。前者性能较高，但是可能会引起一定程度的数据丢失;后者相反。

**3\. 主从同步**

Redis支持将数据同步到多台从库上，这种特性对提高读取性能非常有益。

**4\. 性能**

相比需要依赖磁盘记录每个更新的数据库，基于内存的特性无疑给Redis带来了非常优秀的性能。读写操作之间有显著的性能差异。

**5\. 提供API的语言**

很多...

**6\. 适用场景**

毫无疑问，Redis开创了一种新的数据存储思路，使用Redis，我们不用在面对功能单调的数据库时，把精力放在如何把大象放进冰箱这样的问题上，
而是利用Redis灵活多变的数据结构和数据操作，为不同的大象构建不同的冰箱。希望你喜欢这个比喻。

下面是Redis适用的一些场景:

(1)、取最新N个数据的操作

比如典型的取你网站的最新文章，通过下面方式，我们可以将最新的5000条评论的ID放在Redis的List集合中，并将超出集合部分从数据库获取。
使用LPUSH latest.comments命令，向list集合中插入数据。
插入完成后再用LTRIM latest.comments 0 5000命令使其永远只保存最近5000个ID，然后我们在客户端获取某一页评论时可以用下面的逻辑：

```
FUNCTION get_latest_comments(start,num_items):
    id_list = redis.lrange("latest.comments",start,start+num_items-1)
    IF id_list.length &lt; num_items
        id_list = SQL_DB("SELECT ... ORDER BY time LIMIT ...")
    END
    RETURN id_list
END
```

如果你还有不同的筛选维度，比如某个分类的最新N条，那么你可以再建一个按此分类的List，只存ID的话，Redis是非常高效的。

(2)、排行榜应用，取TOP N操作

这个需求与上面需求的不同之处在于，前面操作以时间为权重，这个是以某个条件为权重，比如按顶的次数排序，这时候就需要我们的sorted set出马了，
将你要排序的值设置成sorted set的score，将具体的数据设置成相应的value，每次只需要执行一条ZADD命令即可。

(3)、需要精准设定过期时间的应用

比如你可以把上面说到的sorted set的score值设置成过期时间的时间戳，那么就可以简单地通过过期时间排序，定时清除过期数据了，不仅是清除Redis中的过期数据，
你完全可以把Redis里这个过期时间当成是对数据库中数据的索引，用Redis来找出哪些数据需要过期删除，然后再精准地从数据库中删除相应的记录。

(4)、计数器应用

Redis的命令都是原子性的，你可以轻松地利用INCR，DECR命令来构建计数器系统。

(5)、Uniq操作，获取某段时间所有数据排重值

这个使用Redis的set数据结构最合适了，只需要不断地将数据往set中扔就行了，set意为集合，所以会自动排重。

(6)、实时系统，反垃圾系统

通过上面说到的set功能，你可以知道一个终端用户是否进行了某个操作，可以找到其操作的集合并进行分析统计对比等。没有做不到，只有想不到。

(7)、Pub/Sub构建实时消息系统

Redis的Pub/Sub系统可以构建实时的消息系统，比如很多用Pub/Sub构建的实时聊天系统的例子。

(8)、构建队列系统

使用list可以构建队列系统，使用sorted set甚至可以构建有优先级的队列系统。

(9)、缓存

这个不必说了，性能优于Memcached，数据结构更多样化。

## 二、安装及使用 ##

**步骤一: 下载Redis**
``` bash
wget http://redis.googlecode.com/files/redis-2.6.14.tar.gz
```

**步骤二: 编译源程序并且测试一下**
``` bash
mkdir env
tar -zxvf redis-2.6.14.tar.gz -C env/
cd redis-2.6.14/
make
yum install tcl
make test
```

最后测试结果为：
	\o/ All tests passed without errors!
	Cleanup: may take some time… OK
那么，恭喜你，成功安装了。
安装完成后，会在src目录下生成几个可执行文件：

    redis-server：Redis服务器的daemon启动程序
    redis-cli：Redis命令行操作工具。当然，你也可以用telnet根据其纯文本协议来操作
    redis-benchmark：Redis性能测试工具，测试Redis在你的系统及你的配置下的读写性能

**步骤三： 生产环境下的Redis的安装与启动**

1，新建目录/usr/local/redis/，将上面make后的文件夹复制到这个下面，同时将redis的bin目录加入的PATH中
``` bash
vi /etc/profile
```
添加export PATH=$PATH:/usr/local/resin/src，然后执行
``` bash
source /etc/profile
```
2，新建/etc/redis/，用来存放redis的配置文件

3，新建/var/redis/端口号/，用来存放每个redis实例的持久化文件

4，修改配置文件，首先将配置文件模板复制到/etc/redis 目录中，以端口号命名，比如6379.conf，然后至少修改以下四个配置项：
	daemonize yes  以redis守护进程模式运行
	pidfile /var/run/redis_端口号.pid  设置redis的PID文件位置
	port  端口号  设置redis实例监听的端口号
	dir  /var/redis/端口号  设置持久化文件存放的位置
5，配置初始化脚本

首先将初始化脚本$REDIS_HOME/utils/目录中的redis_init_script复制到/etc/init.d/目录中，修改名字为redis_端口号，其中端口号表示要让redis监听的端口号，
客户端通过该端口号连接redis，然后修改脚本的REDISPORT变量值为同样的端口号。同时修改EXEC和CLIEXEC的执行路径变量
``` bash
cp utils/redis_init_script /etc/init.d/redis_6379
```
打开redis_6379，修改如下：
    REDISPORT=6379
    EXEC=/usr/local/redis/src/redis-server
    CLIEXEC=/usr/local/redis/src/redis-cli
6，执行/etc/init.d/redis_端口号 start  启动

7，执行/etc/init.d/redis_端口号 stop  停止，或者是用  $ redis-cli SHUTDOWN，
使用kill redis进程PID也可以正常结束redis，但不要用kill -9，谢谢。

8，配置redis服务随系统启动而启动
``` bash
vi /etc/init.d/redis_6379
```
头部三行：
``` bash
#!/bin/sh
#chkconfig: 2345 80 90
#description:redis_6379
```
然后使用chkconfig命令：
``` bash
chkconfig --add redis_6379
chkconfig redis_6379 on
```

**步骤四: 客户端连接验证**

新打开一个Session输入：src/redis-cli，如果出现下面提示，那么您就可以开始Redis之旅了
``` bash
src/redis-cli
redis 127.0.0.1:6379 >
```
**步骤五: 查看Redis日志**

查看服务器端session，即可对Redis的运行状况进行查看或分析了
以上的几个步骤就OK了!!这样一个简单的Redis数据库就可以畅通无阻地运行起来了。

**步骤六 停止Redis实例**

客户端来停止服务，如可以用shutdown来停止Redis实例, 具体如下:
``` bash
src/redis-cli shutdown
```

## 三、配置Redis ##
使用配置文件启动：
``` bash
src/redis-server redis.conf
```
主要配置项：

Redis支持很多的参数，但都有默认值。

	●daemonize:
	默认情况下，redis不是在后台运行的，如果需要在后台运行，把该项的值更改为yes。
	●pidfile
	当Redis在后台运行的时候，Redis默认会把pid文件放在/var/run/redis.pid，
	你可以配置到其他地址。当运行多个redis服务时，需要指定不同的pid文件和端口。
	●bind
	指定Redis只接收来自于该IP地址的请求，如果不进行设置，那么将处理所有请求，在生产环境中最好设置该项。
	●port
	监听端口，默认为6379。
	●timeout
	设置客户端连接时的超时时间，单位为秒。当客户端在这段时间内没有发出任何指令，那么关闭该连接。
	●loglevel
	log等级分为4级，debug, verbose, notice, 和warning。生产环境下一般开启notice。
	●logfile
	配置log文件地址，默认使用标准输出，即打印在命令行终端的窗口上。
	●databases
	设置数据库的个数，可以使用SELECT 命令来切换数据库。默认使用的数据库是0。
	●save
	设置Redis进行数据库镜像的频率。
		if(在60秒之内有10000个keys发生变化时){
		    进行镜像备份
		}else if(在300秒之内有10个keys发生了变化){
		    进行镜像备份
		}else if(在900秒之内有1个keys发生了变化){
		    进行镜像备份
		}
	●rdbcompression
	在进行镜像备份时，是否进行压缩。
	●dbfilename
	镜像备份文件的文件名。
	●dir
	数据库镜像备份的文件放置的路径。这里的路径跟文件名要分开配置是因为Redis在进行备份时，
	先会将当前数据库的状态写入到一个临时文件中，等备份完成时，
	再把该该临时文件替换为上面所指定的文件，而这里的临时文件和上面所配置的备份文件都会放在这个指定的路径当中。
	●slaveof
	设置该数据库为其他数据库的从数据库。
	●masterauth
	当主数据库连接需要密码验证时，在这里指定。
	●requirepass
	设置客户端连接后进行任何其他指定前需要使用的密码。警告：因为redis速度相当快，所以在一台比较好的服务器下，
	一个外部的用户可以在一秒钟进行150K次的密码尝试，这意味着你需要指定非常非常强大的密码来防止暴力破解。
	●maxclients
	限制同时连接的客户数量。当连接数超过这个值时，redis将不再接收其他连接请求，客户端尝试连接时将收到error信息。
	●maxmemory
	设置redis能够使用的最大内存。当内存满了的时候，如果还接收到set命令，redis将先尝试剔除设置过expire信息的key，
	而不管该key的过期时间还没有到达。在删除时，将按照过期时间进行删除，
	最早将要被过期的key将最先被删除。如果带有expire信息的key都删光了，
	那么将返回错误。这样，redis将不再接收写请求，只接收get请求。
	maxmemory的设置比较适合于把redis当作于类似memcached的缓存来使用。
	●appendonly
	默认情况下，redis会在后台异步的把数据库镜像备份到磁盘，但是该备份是非常耗时的，
	而且备份也不能很频繁，如果发生诸如拉闸限电、拔插头等状况，
	那么将造成比较大范围的数据丢失。所以redis提供了另外一种更加高效的数据库备份及灾难恢复方式。开启append only模式之后，
	redis会把所接收到的每一次写操作请求都追加到appendonly.aof文件中，当redis重新启动时，会从该文件恢复出之前的状态。
	但是这样会造成appendonly.aof文件过大，所以redis还支持了BGREWRITEAOF指令，对appendonly.aof进行重新整理。
	所以我认为推荐生产环境下的做法为关闭镜像，开启appendonly.aof，同时可以选择在访问较少的时间每天对appendonly.aof进行重写一次。
	●appendfsync
	设置对appendonly.aof文件进行同步的频率。always表示每次有写操作都进行同步，everysec表示对写操作进行累积，
	每秒同步一次。这个需要根据实际业务场景进行配置。
	●activerehashing
	开启之后，redis将在每100毫秒时使用1毫秒的CPU时间来对redis的hash表进行重新hash，可以降低内存的使用。当你的使用场景中，
	有非常严格的实时性需要，不能够接受Redis时不时的对请求有2毫秒的延迟的话，把这项配置为no。
	如果没有这么严格的实时性要求，可以设置为yes，以便能够尽可能快的释放内存。

## 四、操作Redis ##
1、插入数据
``` bash
redis 127.0.0.1:6379> set name wwl #设置一个key-value对
```
2、查询数据
``` bash
redis 127.0.0.1:6379> get name #取出key所对应的value
```
3、删除键值
``` bash
redis 127.0.0.1:6379> del name #删除这个key以及对应的value
```
4、验证键是否存在
``` bash
redis 127.0.0.1:6379> exists name #其中0，代表此key不存在;1代表存在
```

----------------------------

##FAQ

1. redis的java客户端是Jedis，地址：<https://github.com/xetorthio/jedis>
1. 用redis做像memcache的缓存如何设计？

想利用redis作为缓存，将查询结果缓存起来，像memcache的效果。
但是jedis(java客户端)的参数只接收String或者byte[]参数，那么Object如何设计存储呢？

我先找的做法是将结果拼接成字符串存储，取出来再将字符串分割，但是这样简单的内容可以，大的对象就太麻烦了。
如果将Object序列化、反序列化的性能如何保证呢？Memcache的客户端是怎么做的来存取Object的呢？

**最佳答案：**

memcache存取对象是序列化和反序列化。redis可以在客户端自行实现。

如果非要存储对象，这部分工作必然要做，区别只在在哪里做，有无经过验证的第三方代码，自己开发必然是一部分工作量，这个东西开发的质量不高，会导致CPU过高的。
我在项目中使用memcache不会缓存对象，而是存储json字符串。因为频繁的序列化和反序列化会占用cpu，所以我们这样使用可以降低memcache服务对CPU的要求。
一般部署memcache的主机都是内存比较空闲的主机，而把解析json这部分工作移到应用内部，应用所在主机的CPU配置必然是高的。这样可以有效的利用主机资源。

解析json字符串的工具，选择fastjson，地址：<https://github.com/alibaba/fastjson>

**还有一个选择：**

I want to persist my objects in Redis. How can I do it?

You should definitely check JOhm! And of course, you can always serialize it and store it.
