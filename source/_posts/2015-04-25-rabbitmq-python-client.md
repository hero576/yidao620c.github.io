---
layout: post
title: "使用python开发RabbitMQ应用"
date: 2015-04-25 11:21:34 +0800
comments: true
toc: true
categories: linux
tags: rabbitmq
---

### 安装环境

参考了RabbitMQ网站上提供的英文版本入门指南: <http://www.rabbitmq.com/getstarted.html>

测试环境：CentOS 6.2

### 测试环境准备

安装python（一般系统都自带了python）

安装RabbitMQ server可以参考前面的文章。

安装pika

使用pip安装的时候可能会报错：
```
importerror no module named pkg_resources
```

请用下面命令解决这个问题：
```
$ curl https://bitbucket.org/pypa/setuptools/raw/bootstrap/ez_setup.py | python
```

然后还可能出现：<!--more-->
```
pkg_resources.distributionnotfound pip==1.4.1
```

这时候先把pip卸载掉，执行：
```
sudo yum remove python-pip
```
然后去下载最新的get-pip.py文件，执行`python get-pip.py`安装

在`/etc/profile`里面将`/usr/local/python27/bin`加入PATH最前面

把rabbitmq server启动一下和准备好测试目录rabbitmq_app：
```
$ /usr/local/rabbitmq/sbin/rabbitmq-server -detached
$ cd ~
$ mkdir -p test /rabbitmq_app
$ cd test /rabbitmq_app
$ mkdir tut1 tut2 tut3 tut4 tut5 tut6
```

### 实例一：来个hello world程序
```
$ cd tut1
$ vim send.py (代码如下)
$ vim receive.py (代码如下)
```

首先是消息发送程序: send.py
``` python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import sys
import pika
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
channel.queue_declare(queue = 'hello')
if len (sys.argv) < 2 :
     print 'message is empty!'
     sys.exit(0)
message = sys.argv[1]
channel.basic_publish(exchange = '', routing_key='hello', body = message)
print "[x] sent: '" + message + "'\n"
connection.close()
```
跑一下send.py发送一个消息
```
$ python send.py ‘Hello World!’
$ python send.py ‘你好刀哥’
$ /usr/local/rabbitmq/sbin/rabbitmqctl list_queues
```
结果如下面：
```
Listing queues …
hello 2
… done .
```

如果你也看到hello队列里面有一个消息的话，就证明可以发消息了。

然后写一个接收消息脚本：receive.py
``` python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import pika
connection = pika.BlockingConnection(pika.ConnectionParameters( 'localhost' ))
channel = connection.channel()
channel.queue_declare(queue = 'hello' )
print '[*] Waiting for messages. To exit press CTRL+C'

def callback(ch, method, properties, body):
     print body

channel.basic_consume(callback, queue = 'hello' , no_ack = True )
channel.start_consuming()
```
其中第12行的 no_ack=True 表示消费完了这个消息以后不主动把完成状态通知rabbitmq。

然后开另外一个shell，执行一下receive.py
```
$ python receive.py
```
结果：
```
[*] Waiting for messages. To exit press CTRL+C
Hello World!
你好刀哥
```

### 实例二：工作队列（work queue / task queue）

一般应用于把比较耗时的任务从主线任务分离出来。比如一个http页面请求，
里面需要发送带大附件的邮件、或者是要处理一张头像图片等。这类型工作队列的处理端一般有多个worker进程，
分担队列里面的任务。这就有点负载均衡的策略在里面了。
尽量做到每个进程的工作量比较平均，而且是完成了一个任务才接 第二个任务。看看我们的实现吧。
```
$ cd tut2
$ vim manager.py (代码如下)
$ vim worker.py (代码如下)
```
首先是消息发送程序: manager.py
``` python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import pika
import sys
parameters = pika.ConnectionParameters(host = 'localhost' )
connection = pika.BlockingConnection(parameters)
channel = connection.channel()
channel.queue_declare(queue = 'task_queue' , durable = True )
message = ' ' .join(sys.argv[ 1 :]) or "Hello World!"
channel.basic_publish(exchange = '',
                       routing_key = 'task_queue' ,
                       body = message,
                       properties = pika.BasicProperties(
                          delivery_mode = 2 , # make message persistent
                       ))
print " [x] Sent %r" % (message,)
connection.close()
```
其中第8行的 durable=True 声明了队列需要持久化，第14行的 delivery_mode = 2 声明了队列的消息需要持久化。

然后写一个接收消息脚本：worker.py
``` python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import pika
import time
connection = pika.BlockingConnection(pika.ConnectionParameters(
         host = 'localhost' ))
channel = connection.channel()
channel.queue_declare(queue = 'task_queue' , durable = True )
print ' [*] Waiting for messages. To exit press CTRL+C'

def callback(ch, method, properties, body):
     print " [x] Received %r" % (body,)
     time.sleep( body.count( '.' ) )
     print " [x] Done"
     ch.basic_ack(delivery_tag = method.delivery_tag)

channel.basic_qos(prefetch_count = 1 )
channel.basic_consume(callback,
                       queue = 'task_queue' )
channel.start_consuming()
```
其中第15行的 basic_ack 是执行完任务通知rabbitmq，
第17行的basic_qos是告诉rabbitmq只有当worker完成了任务以后才分派1条新的消息，实现公平分派。

测试方法，开3个bash，2个跑worker，1个跑manager：
```
$ python manager.py task1.
$ python manager.py task2..
$ python manager.py task3…
$ python manager.py task4….
```
点号数量决定worker工作的时间( 其实是睡觉时间，呵呵 time.sleep(body.count(‘.’)) )。
而在worker那边，可以看到每个worker都处理了两个任务。
这种分配机制就是所谓的循环调度（Round-robin dispatching）

### 实例三：发布和订阅

发布订阅模式，简单来说就像是广播，一个消息发布出来以后，所有订阅者都能听到，
至于接收到这个信息以后大家做什么就看具体个人了。

啊！怎么忽然冒出个X，是什么玩意！这个X就是所谓的exchange，简单来说就是消息的管家，
由他决定接收到的信息是放特定的队列，还是所有队列，还是直接丢弃。

其实在前两个实例里面，已经用到了exchange （channel.basic_publish(exchange=”,…），
这个exchange的名字为空，外号无名（人若无名，便可专心练剑~）。他会把你的消息都转达给routing_key指明的队列。
当我们声明了exchange以后，我们需要为queue和exchange建立联系，这时候，就要用到绑定（binding）了。
```
$ cd tut3
$ vim emitlog.py (代码如下)
$ vim recelog.py (代码如下)
```
日志生产者
``` python emitlog.py
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
         host = 'localhost' ))
channel = connection.channel()

channel.exchange_declare(exchange = 'logs' ,
                          type = 'fanout' )

message = ' ' .join(sys.argv[ 1 :]) or "info: Hello World!"
channel.basic_publish(exchange = 'logs' ,
                       routing_key = '',
                       body = message)
print " [x] Sent %r" % (message,)
connection.close()
```
然后是日志消费者
``` python recelog.py
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
         host = 'localhost' ))
channel = connection.channel()
channel.exchange_declare(exchange = 'logs' ,
                          type = 'fanout' )
result = channel.queue_declare(exclusive = True )
queue_name = result.method.queue
channel.queue_bind(exchange = 'logs' ,
                    queue = queue_name)
print ' [*] Waiting for logs. To exit press CTRL+C'

def callback(ch, method, properties, body):
     print " [x] %r" % (body,)

channel.basic_consume(callback,
                       queue = queue_name,
                       no_ack = True )
channel.start_consuming()
```
测试：

和前一个实例差不多。开3个bash，2个跑recelog，1个跑emitlog。
查看recelog是否都收到emitlog发送的消息。代码里面用 了一个fanout(意思是成扇形展开)类型的exchange，
只要和exchange绑定的queue都能收到一份消息的 copy，routing_key会被忽略掉。

### 路由模式 （选择接收信息）
```
$ cd tut4
$ vim emitlog.py (代码如下)
$ vim recelog.py (代码如下)
```
生产者
``` python emitlog.py
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
         host = 'localhost' ))
channel = connection.channel()
channel.exchange_declare(exchange = 'direct_logs' ,
                          type = 'direct' )
severity = sys.argv[ 1 ] if len (sys.argv) > 1 else 'info'
message = ' ' .join(sys.argv[ 2 :]) or 'Hello World!'
channel.basic_publish(exchange = 'direct_logs' ,
                       routing_key = severity,
                       body = message)
print " [x] Sent %r:%r" % (severity, message)
connection.close()
```
这里声明exchange时类型定义为direct（直接匹配），就是说只有当一个信息的routing_key和队列的binding_key一 致时，
信息才会被放入到这个队列。消息发布给exchange时必须带上routing_key。其实在消息生产端，队列这个概念是透明的。

消费者
``` python recelog.py
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
         host = 'localhost' ))
channel = connection.channel()

channel.exchange_declare(exchange = 'direct_logs' ,
                          type = 'direct' )
result = channel.queue_declare(exclusive = True )
queue_name = result.method.queue
severities = sys.argv[ 1 :]
if not severities:
     print >> sys.stderr, "Usage: %s [info] [warning] [error]" % \
                          (sys.argv[ 0 ],)
     sys.exit( 1 )
for severity in severities:
     channel.queue_bind(exchange = 'direct_logs' ,
                        queue = queue_name,
                        routing_key = severity)
print ' [*] Waiting for logs. To exit press CTRL+C'
def callback(ch, method, properties, body):
     print " [x] %r:%r" % (method.routing_key, body,)
channel.basic_consume(callback,
                       queue = queue_name,
                       no_ack = True )
channel.start_consuming()
```

这里首先定义exchange，和消息发送端是一样的。然后定义队列，队列是自动命名，
并且只要进程终止，队列就会终止。然后把队列和 exchange绑定，绑定时的routing_key是用户输入的，
如果输入多个key，就做多次的绑定。注意这里的队列还是一个。如果你需要建立两个 队列，就得跑两次这个python脚本。

### topic和rpc

官方tutorial还有两个高级一点的实例，topic和rpc，这里就不作说明了，留着大家学学英文吧 :)

RabbitMQ提供了很多消息队列客户端代码，比如python，java，c等等，大家可以根据产品或项目的实际情况选择。关键是原理必须搞懂。

### 其他资源

中文入门篇：<http://adamlu.net/dev/2011/09/rabbitmq-get-started/>
