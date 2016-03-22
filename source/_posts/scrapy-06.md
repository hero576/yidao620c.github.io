---
title: "Scrapy笔记（6）- Item Pipeline"
date: 2016-03-18 01:00:12 +0800
comments: true
toc: true
categories: python
tags: [scrapy]
---

## 开篇
当一个item被蜘蛛爬取到之后会被发送给Item Pipeline，然后多个组件按照顺序处理这个item。
每个Item Pipeline组件其实就是一个实现了一个简单方法的Python类。他们接受一个item并在上面执行逻辑，还能决定这个item到底是否还要继续往下传输，如果不要了就直接丢弃。

使用Item Pipeline的常用场景：

* 清理HTML数据
* 验证被抓取的数据(检查item是否包含某些字段)
* 重复性检查(然后丢弃)
* 将抓取的数据存储到数据库中

## 编写自己的Pipeline
定义一个Python类，然后实现方法`process_item(self, item, spider)`即可，返回一个字典或Item，或者抛出`DropItem`异常丢弃这个Item。

或者还可以实现下面几个方法：<!--more-->

* `open_spider(self, spider)` 蜘蛛打开的时执行
* `close_spider(self, spider)` 蜘蛛关闭时执行
* `from_crawler(cls, crawler)` 可访问核心组件比如配置和信号，并注册钩子函数到Scrapy中

## Item Pipeline示例

### 价格验证
我们通过一个价格验证例子来看看怎样使用
``` python
from scrapy.exceptions import DropItem

class PricePipeline(object):

    vat_factor = 1.15

    def process_item(self, item, spider):
        if item['price']:
            if item['price_excludes_vat']:
                item['price'] = item['price'] * self.vat_factor
            return item
        else:
            raise DropItem("Missing price in %s" % item)
```

### 将item写入json文件
下面的这个Pipeline将所有的item写入到一个单独的json文件，一行一个item
``` python
import json

class JsonWriterPipeline(object):

    def __init__(self):
        self.file = open('items.jl', 'wb')

    def process_item(self, item, spider):
        line = json.dumps(dict(item)) + "\n"
        self.file.write(line)
        return item
```

### 将item存储到MongoDB中
这个例子使用[pymongo](http://api.mongodb.org/python/current/)来演示怎样讲item保存到MongoDB中。
MongoDB的地址和数据库名在配置中指定，这个例子主要是向你展示怎样使用`from_crawler()`方法，以及如何清理资源。
``` python
import pymongo

class MongoPipeline(object):

    collection_name = 'scrapy_items'

    def __init__(self, mongo_uri, mongo_db):
        self.mongo_uri = mongo_uri
        self.mongo_db = mongo_db

    @classmethod
    def from_crawler(cls, crawler):
        return cls(
            mongo_uri=crawler.settings.get('MONGO_URI'),
            mongo_db=crawler.settings.get('MONGO_DATABASE', 'items')
        )

    def open_spider(self, spider):
        self.client = pymongo.MongoClient(self.mongo_uri)
        self.db = self.client[self.mongo_db]

    def close_spider(self, spider):
        self.client.close()

    def process_item(self, item, spider):
        self.db[self.collection_name].insert(dict(item))
        return item
```

### 重复过滤器
假设我们的item里面的id字典是唯一的，但是我们的蜘蛛返回了多个相同id的item
``` python
from scrapy.exceptions import DropItem

class DuplicatesPipeline(object):

    def __init__(self):
        self.ids_seen = set()

    def process_item(self, item, spider):
        if item['id'] in self.ids_seen:
            raise DropItem("Duplicate item found: %s" % item)
        else:
            self.ids_seen.add(item['id'])
            return item
```

## 激活一个Item Pipeline组件
你必须在配置文件中将你需要激活的Pipline组件添加到`ITEM_PIPELINES`中
``` python
ITEM_PIPELINES = {
    'myproject.pipelines.PricePipeline': 300,
    'myproject.pipelines.JsonWriterPipeline': 800,
}
```
后面的数字表示它的执行顺序，从低到高执行，范围0-1000

