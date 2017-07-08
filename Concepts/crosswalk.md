# 与SQL比较

## database里面是什么？
该章向SQL用户介绍了InfluxDB哪里像SQL数据库以及它哪里不像。将突出讲了两者之间的一些主要区别，并提供了不同的数据库术语和查询语言。

### 一般来说
InfluxDB是一个时间序列数据库，SQL数据库可以提供时序的功能，但严格说时序不是其目的。简而言之，InfluxDB用于存储大量的时间序列数据，并对这些数据进行快速的实时分析。

### 时间是一切
在InfluxDB中，timestamp标识了在任何给定数据series中的单个点。这就像一个SQL数据库表，其中主键是由系统预先设置的，并且始终是时间。

InfluxDB还会认识到您的schema可能随时间而改变。在InfluxDB中，您不需要在前面定义schema。数据点可以有一个measurement的field的一个，也可以有这个measurement的所有field，或其间的任何数字。 您可以在写数据的时候为该measurement添加一个新的field。

## 术语
下表是一个叫`foodships`的SQL数据库的例子，并有没有索引的`#_foodships`列和有索引的`park_id`,`planet`和`time`列。

| park_id | planet  | time                | #_foodships  |
|---------|---------|---------------------|--------------|
|       1 | Earth   | 1429185600000000000 |            0 |
|       1 | Earth   | 1429185601000000000 |            3 |
|       1 | Earth   | 1429185602000000000 |           15 |
|       1 | Earth   | 1429185603000000000 |           15 |
|       2 | Saturn  | 1429185600000000000 |            5 |
|       2 | Saturn  | 1429185601000000000 |            9 |
|       2 | Saturn  | 1429185602000000000 |           10 |
|       2 | Saturn  | 1429185603000000000 |           14 |
|       3 | Jupiter | 1429185600000000000 |           20 |
|       3 | Jupiter | 1429185601000000000 |           21 |
|       3 | Jupiter | 1429185602000000000 |           21 |
|       3 | Jupiter | 1429185603000000000 |           20 |
|       4 | Saturn  | 1429185600000000000 |            5 |
|       4 | Saturn  | 1429185601000000000 |            5 |
|       4 | Saturn  | 1429185602000000000 |            6 |
|       4 | Saturn  | 1429185603000000000 |            5 |

这些数据在InfluxDB看起来就像这样：

```
name: foodships
tags: park_id=1, planet=Earth
time			               #_foodships
----			               ------------
2015-04-16T12:00:00Z	 0
2015-04-16T12:00:01Z	 3
2015-04-16T12:00:02Z	 15
2015-04-16T12:00:03Z	 15

name: foodships
tags: park_id=2, planet=Saturn
time			               #_foodships
----			               ------------
2015-04-16T12:00:00Z	 5
2015-04-16T12:00:01Z	 9
2015-04-16T12:00:02Z	 10
2015-04-16T12:00:03Z	 14

name: foodships
tags: park_id=3, planet=Jupiter
time			               #_foodships
----			               ------------
2015-04-16T12:00:00Z	 20
2015-04-16T12:00:01Z	 21
2015-04-16T12:00:02Z	 21
2015-04-16T12:00:03Z	 20

name: foodships
tags: park_id=4, planet=Saturn
time			               #_foodships
----			               ------------
2015-04-16T12:00:00Z	 5
2015-04-16T12:00:01Z	 5
2015-04-16T12:00:02Z	 6
2015-04-16T12:00:03Z	 5
```

参考上面的数据，一般可以这么说：

* InfluxDB的measurement(`foodships`)和SQL数据库里的table类似；
* InfluxDB的tag(`park_id`和`planet`)类似于SQL数据库里索引的列；
* InfluxDB中的field(`#_foodships`)类似于SQL数据库里没有索引的列；
* InfluxDB里面的数据点(例如`2015-04-16T12:00:00Z	 5`)类似于SQL数据库的行；

基于这些数据库术语的比较，InfluxDB的continuous query和retention policy与SQL数据库中的存储过程类似。 它们被指定一次，然后定期自动执行。

当然，SQL数据库和InfluxDB之间存在一些重大差异。SQL中的`JOIN`不适用于InfluxDB中的measurement。而且，正如我们上面提到的那样，一个measurement就像一个SQL的table，其中主索引总是被预设为时间。InfluxDB的时间戳记必须在UNIX epoch（GMT）或格式化为日期时间RFC3339格式的字符串才有效。

查看更多关于InfluxDB的术语的详细解释，请参考[专业术语](glossary.md)。

## InfluxQL和SQL
在InfluxDB中InfluxQL是一种类SQL的语言。对于来自其他SQL或类SQL环境的用户来说，它已经被精心设计，而且还提供特定于存储和分析时间序列数据的功能。

InfluxQL的`select`语句来自于SQL中的`select`形式：

```
SELECT <stuff> FROM <measurement_name> WHERE <some_conditions>
```

`where`是可选的，在InfluxDB里为了查询到上面数据，需要输入：

```
SELECT * FROM "foodships"
```

如果你仅仅想看planet为`Saturn`的数据：

```
SELECT * FROM "foodships" WHERE "planet" = 'Saturn'
```

如果你想看到planet为`Saturn`，并且在UTC时间为2015年4月16号12:00:01之后的数据：

```
SELECT * FROM "foodships" WHERE "planet" = 'Saturn' AND time > '2015-04-16 12:00:01'
```
如上例所示，InfluxQL允许您在`WHERE`子句中指定查询的时间范围。您可以使用包含单引号的日期时间字符串，格式为YYYY-MM-DD HH：MM：SS.mmm（mmm为毫秒，为可选项，您还可以指定微秒或纳秒。您还可以使用相对时间与`now()`来指代服务器的当前时间戳：

```
SELECT * FROM "foodships" WHERE time > now() - 1h
```

该查询输出measurement为`foodships`中的数据，其中时间戳比服务器当前时间减1小时。与now()做计算来决定时间范围的可选单位有：

字母|意思
----|----
u或µ|微秒
ms|毫秒
s|秒
m|分钟
h|小时
d|天
w|星期

InfluxQL还支持正则表达式，表达式中的运算符，`SHOW`语句和`GROUP BY`语句。有关这些主题的深入讨论，请参阅我们的[数据探索]()页面。 InfluxQL功能还包括`COUNT`，`MIN`，`MAX`，`MEDIAN`，`DERIVATIVE`等。 有关完整列表，请查看[函数]()页面。

## 为什么InfluxDB不是CRUD的一个解释
InfluxDB是针对时间序列数据进行了优化的数据库。这些数据通常来自分布式传感器组，来自大型网站的点击数据或金融交易列表等。

这个数据有一个共同之处在于它只看一个点没什么用。一个读者说，在星期二UTC时间为12:38:35时根据他的电脑CPU利用率为12％，这个很难得出什么结论。只有跟其他的series结合并可视化时，它变得更加有用。随着时间的推移开始显现的趋势，是我们从这些数据里真正想要看到的。另外，时间序列数据通常是一次写入，很少更新。

结果是，由于优先考虑create和read数据的性能而不是update和delete，InfluxDB不是一个完整的CRUD数据库，更像是一个CR-ud。