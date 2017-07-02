# 采样和数据保留

InfluxDB每秒可以处理数十万的数据点。如果要长时间地存储大量的数据，对于存储会是很大的压力。一个很自然的方式就是对数据进行采样，对于高精度的裸数据存储较短的时间，而对于低精度的的数据可以保存得久一些甚至永久保存。

InfluxDB提供了两个特性——连续查询(Continuous Queries简称CQ)和保留策略(Retention Policies简称RP)，分别用来处理数据采样和管理老数据的。这一章将会展示CQs和RPs的例子，看下在InfluxDB中怎么使用这两个特性。

## 定义
**Continuous Query (CQ)**是在数据库内部自动周期性跑着的一个InfluxQL的查询，CQs需要在`SELECT`语句中使用一个函数，并且一定包括一个`GROUP BY time()`语句。

**Retention Policy (RP)**是InfluxDB数据架构的一部分，它描述了InfluxDB保存数据的时间。InfluxDB会比较服务器本地的时间戳和你数据的时间戳，并删除比你在RPs里面用`DURATION`设置的更老的数据。单个数据库中可以有多个RPs但是每个数据的RPs是唯一的。

这一章不会详细地介绍创建和管理CQs和RPs的语法，如果你对这两个概念还是很陌生的话，建议查看[CQ文档]()和[RP文档]()。

## 数据采样
本节使用虚构的实时数据，以10秒的间隔，来追踪餐厅通过电话和网站订购食品的订单数量。我们会把这些数据存在`food_data`数据库里，其measurement为`orders`，fields分别为`phone`和`website`。

就像这样：

```
name: orders
------------
time			               phone	 website
2016-05-10T23:18:00Z	 10 	   30
2016-05-10T23:18:10Z	 12 	   39
2016-05-10T23:18:20Z	 11 	   56
```

## 目标
假定在长时间的运行中，我们只关心每三十分钟通过手机和网站订购的平均数量，我们希望用RPs和CQs实现下面的需求：

* 自动将十秒间隔数据聚合到30分钟的间隔数据
* 自动删除两个小时以上的原始10秒间隔数据
* 自动删除超过52周的30分钟间隔数据

## 数据库准备
在写入数据到数据库`food_data`之前，我们先做如下的准备工作，在写入之前设置CQs是因为CQ只对最近的数据有效; 即数据的时间戳不会比`now()`减去CQ的`FOR`子句的时间早，或是如果没有`FOR`子句的话比`now()`减去`GROUP BY time()`间隔早。

### 1. 创建数据库

```
> CREATE DATABASE "food_data"
```

### 2. 创建一个两个小时的默认RP
如果我们写数据的时候没有指定RP的话，InfluxDB会使用默认的RP，我们设置默认的RP是两个小时。使用`CREATE RETENTION POLICY`语句来创建一个默认RP:

```
> CREATE RETENTION POLICY "two_hours" ON "food_data" DURATION 2h REPLICATION 1 DEFAULT
```

这个RP的名字叫`two_hours`作用于`food_data`数据库上，`two_hours`保存数据的周期是两个小时，并作为`food_data`的默认RP。

*复制片参数(REPLICATION 1)是必须的，但是对于单个节点的InfluxDB实例，复制片只能设为1*

>说明：在步骤1里面创建数据库时，InfluxDB会自动生成一个叫做`autogen`的RP，并作为数据库的默认RP，`autogen`这个RP会永远保留数据。在输入上面的命令之后，`two_hours`会取代`autogen`作为`food_data`的默认RP。

### 3. 创建一个保留52周数据的RP
接下来我们创建另一个RP保留数据52周，但不是数据库的默认RP。最终30分钟间隔的数据会保存在这个RP里面。

使用`CREATE RETENTION POLICY`语句来创建一个非默认的RP：

```
> CREATE RETENTION POLICY "a_year" ON "food_data" DURATION 52w REPLICATION 1
```

这个语句对数据库`food_data`创建了一个叫做`a_year`的RP，`a_year`保存数据的周期是52周。去掉`DEFAULT`参数可以保证`a_year`不是数据库`food_data`的默认RP。这样在读写的时候如果没有指定，仍然是使用`two_hours`这个默认RP。

### 4. 创建CQ
现在我们已经创建了RPs，现在我们要创建一个CQ，去将10秒间隔的数据采样到30分钟的间隔，并把它们安装不同存储策略把它们存在不同的measurement里。

使用`CREATE CONTINUOUS QUERY`来生成一个CQ：

```
> CREATE CONTINUOUS QUERY "cq_30m" ON "food_data" BEGIN
  SELECT mean("website") AS "mean_website",mean("phone") AS "mean_phone"
  INTO "a_year"."downsampled_orders"
  FROM "orders"
  GROUP BY time(30m)
END
```

上面创建了一个叫做`cq_30m`的CQ作用于`food_data`数据库上。`cq_30m`告诉InfluxDB每30分钟计算一次measurement为`orders`并使用默认RP`tow_hours`的字段`website`和`phone`的平均值，然后把结果写入到RP为`a_year`，两个字段分别是`mean_website`和`mean_phone`的measurement名为`downsampled_orders`的数据中。InfluxDB会每隔30分钟跑对之前30分钟的数据跑一次这个查询。

>说明：注意到我们在`INTO`语句中使用了`"<retention_policy>"."<measurement>"`这样的语句，当要写入到非默认的RP时，就需要这样的写法。

## 结果
使用新的CQ和两个新的RPs，`food_data`已经开始接收数据了。之后我们向数据库里写数据，并且持续一段时间之后，我们可以看到两个measurement分别是`orders`和`downsampled_orders`。

```
> SELECT * FROM "orders" LIMIT 5
name: orders
---------
time			                phone  website
2016-05-13T23:00:00Z	  10     30
2016-05-13T23:00:10Z	  12     39
2016-05-13T23:00:20Z	  11     56
2016-05-13T23:00:30Z	  8      34
2016-05-13T23:00:40Z	  17     32

> SELECT * FROM "a_year"."downsampled_orders" LIMIT 5
name: downsampled_orders
---------------------
time			                mean_phone  mean_website
2016-05-13T15:00:00Z	  12          23
2016-05-13T15:30:00Z	  13          32
2016-05-13T16:00:00Z	  19          21
2016-05-13T16:30:00Z	  3           26
2016-05-13T17:00:00Z	  4           23
```

在`orders`里面是10秒钟间隔的裸数据，保存时间为2小时。在`downsampled_orders`里面是30分钟的聚合数据，保存时间为52周。

注意到`downsampled_orders`返回的第一个时间戳比`orders`返回的第一个时间戳要早，这是因为InfluxDB已经删除了`orders`中时间比本地早两个小时的数据。InfluxDB会在52周之后开始删除`downsampled_orders`中的数据。

>说明：注意这里我们在第二个语句中使用了` "<retention_policy>"."<measurement>"`来查询`downsampled_orders`，因为只有不是使用默认的RP我们就需要指定RP。
>
默认InfluxDB是每隔三十分钟check一次RP，在两次check之间，`orders`中可能有超过两个小时的数据，这个check的间隔可以在InfluxDB的配置文件中更改。

使用RPs和CQs的组合，我们已经成功地创建的数据库并保存高精度的裸数据较短的时间，而保存高精度的数据更长时间。现在我们对这些特性的工作有了大概的了解，我们推荐到[CQs]()和[RPs]()去看更详细的文档。