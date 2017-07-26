# 数据查询语法

InfluxQL是一种类似SQL的查询语言，用于与InfluxDB中的数据进行交互。 以下部分详细介绍了InfluxQL的`SELECT`语句有关查询语法。

### 示例数据
本文使用国家海洋和大气管理局（NOAA）海洋作业和服务中心的公开数据。请参阅[示例数据](https://docs.influxdata.com/influxdb/v1.3/query_language/data_download/)页面下载数据，并按照以下部分中的示例查询进行跟踪。开始之后，请随时了解`h2o_feet`这个measurement中的数据样本：

```
name: h2o_feet
-———————————–
time            level description      location      water_level   
2015-08-18T00:00:00Z    between 6 and 9 feet       coyote_creek    8.12
2015-08-18T00:00:00Z    below 3 feet        santa_monica       2.064
2015-08-18T00:06:00Z    between 6 and 9 feet      coyote_creek      8.005
2015-08-18T00:06:00Z    below 3 feet        santa_monica       2.116
2015-08-18T00:12:00Z    between 6 and 9 feet       coyote_creek    7.887
2015-08-18T00:12:00Z    below 3 feet        santa_monica       2.028
```

`h2o_feet`这个measurement中的数据以六分钟的时间间隔进行。measurement具有一个tag key(`location`)，它具有两个tag value：`coyote_creek`和`santa_monica`。measurement还有两个field：`level description`用字符串类型和`water_level`浮点型。所有这些数据都在`NOAA_water_database`数据库中。

>声明：`level description`字段不是原始NOAA数据的一部分——我们将其存储在那里，以便拥有一个带有特殊字符和字符串field value的field key。

## 基本的SELECT语句
`SELECT`语句从特定的measurement中查询数据。 如果厌倦阅读，查看这个InfluxQL短片(*注意：可能看不到，请到原文处查看[https://docs.influxdata.com/influxdb/v1.3/query_language/data_exploration/#syntax](https://docs.influxdata.com/influxdb/v1.3/query_language/data_exploration/#syntax)*)：

<iframe src="https://player.vimeo.com/video/192712451?title=0&byline=0&portrait=0" width="60%" height="250px" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

### 语法
```
SELECT <field_key>[,<field_key>,<tag_key>] FROM <measurement_name>[,<measurement_name>]
```

#### 语法描述
`SELECT`语句需要一个`SELECT`和`FROM`子句。

##### `SELECT`子句

`SELECT`支持指定数据的几种格式：

`SLECT *`

返回所有的field和tag。

`SELECT "<field_key>"`

返回特定的field。

`SELECT "<field_key>","<field_key>"`

返回多个field。

`SELECT "<field_key>","<tag_key>"`

返回特定的field和tag，`SELECT`在包括一个tag时，必须只是指定一个field。

`SELECT "<field_key>"::field,"<tag_key>"::tag`

返回特定的field和tag，`::[field | tag]`语法指定标识符的类型。 使用此语法来区分具有相同名称的field key和tag key。

##### `FROM`子句
`FROM`子句支持几种用于指定measurement的格式：

`FROM <measurement_name>`

从单个measurement返回数据。如果使用CLI需要先用`USE`指定数据库，并且使用的`DEFAULT`存储策略。如果您使用HTTP API,需要用`db`参数来指定数据库，也是使用`DEFAULT`存储策略。

`FROM <measurement_name>,<measurement_name>`

从多个measurement中返回数据。

`FROM <database_name>.<retention_policy_name>.<measurement_name>`

从一个完全指定的measurement中返回数据，这个完全指定是指指定了数据库和存储策略。

`FROM <database_name>..<measurement_name>`

从一个用户指定的数据库中返回存储策略为`DEFAULT`的数据。

##### 引号
如果标识符包含除[A-z，0-9，_]之外的字符，如果它们以数字开头，或者如果它们是InfluxQL关键字，那么它们必须用双引号。虽然并不总是需要，我们建议您双引号标识符。

>注意：查询的语法与行协议是不同的。

#### 例子
##### 例一：从单个measurement查询所有的field和tag

```
> SELECT * FROM "h2o_feet"

name: h2o_feet
--------------
time                   level description      location       water_level
2015-08-18T00:00:00Z   below 3 feet           santa_monica   2.064
2015-08-18T00:00:00Z   between 6 and 9 feet   coyote_creek   8.12
[...]
2015-09-18T21:36:00Z   between 3 and 6 feet   santa_monica   5.066
2015-09-18T21:42:00Z   between 3 and 6 feet   santa_monica   4.938
```

该查询从`h2o_feet`measurement中选择所有field和tag。

如果您使用CLI，请确保在运行查询之前输入`USE NOAA_water_database`。CLI查询`USE`的数据库并且存储策略是`DEFAULT`的数据。如果使用HTTP API，请确保将`db`查询参数设置为`NOAA_water_database`。如果没有设置rp参数，则HTTP API会自动选择数据库的`DEFAULT`存储策略。
 
 ##### 例二：从单个measurement中查询特定tag和field
 
 ```
 > SELECT "level description","location","water_level" FROM "h2o_feet"

name: h2o_feet
--------------
time                   level description      location       water_level
2015-08-18T00:00:00Z   below 3 feet           santa_monica   2.064
2015-08-18T00:00:00Z   between 6 and 9 feet   coyote_creek   8.12
[...]
2015-09-18T21:36:00Z   between 3 and 6 feet   santa_monica   5.066
2015-09-18T21:42:00Z   between 3 and 6 feet   santa_monica   4.938
 ```
 
该查询field `level descriptio`，tag `location`和field `water_leval`。 请注意，`SELECT`子句在包含tag时必须至少指定一个field。

##### 例三：从单个measurement中选择特定的tag和field，并提供其标识符类型

```
> SELECT "level description"::field,"location"::tag,"water_level"::field FROM "h2o_feet"

name: h2o_feet
--------------
time                   level description      location       water_level
2015-08-18T00:00:00Z   below 3 feet           santa_monica   2.064
2015-08-18T00:00:00Z   between 6 and 9 feet   coyote_creek   8.12
[...]
2015-09-18T21:36:00Z   between 3 and 6 feet   santa_monica   5.066
2015-09-18T21:42:00Z   between 3 and 6 feet   santa_monica   4.938
```

查询从measurement `h2o_feet`中选择field `level description`，tag `location`和field `water_leval`。`:: [field | tag]`语法指定标识符是field还是tag。使用`:: [field | tag]`以区分相同的field key和tag key。大多数用例并不需要该语法。

##### 例四：从单个measurement查询所有field

```
> SELECT *::field FROM "h2o_feet"

name: h2o_feet
--------------
time                   level description      water_level
2015-08-18T00:00:00Z   below 3 feet           2.064
2015-08-18T00:00:00Z   between 6 and 9 feet   8.12
[...]
2015-09-18T21:36:00Z   between 3 and 6 feet   5.066
2015-09-18T21:42:00Z   between 3 and 6 feet   4.938
```

该查询从measurement `h2o_feet`中选择所有field。`SELECT`子句支持将`*`语法与`::`语法相结合。

##### 例五：从measurement中选择一个特定的field并执行基本计算

```
> SELECT ("water_level" * 2) + 4 from "h2o_feet"

name: h2o_feet
--------------
time                   water_level
2015-08-18T00:00:00Z   20.24
2015-08-18T00:00:00Z   8.128
[...]
2015-09-18T21:36:00Z   14.132
2015-09-18T21:42:00Z   13.876
```

该查询将`water_level`字段值乘以2，并加上4。请注意，InfluxDB遵循标准操作顺序。

##### 例六：从多个measurement中查询数据

```
> SELECT * FROM "h2o_feet","h2o_pH"

name: h2o_feet
--------------
time                   level description      location       pH   water_level
2015-08-18T00:00:00Z   below 3 feet           santa_monica        2.064
2015-08-18T00:00:00Z   between 6 and 9 feet   coyote_creek        8.12
[...]
2015-09-18T21:36:00Z   between 3 and 6 feet   santa_monica        5.066
2015-09-18T21:42:00Z   between 3 and 6 feet   santa_monica        4.938

name: h2o_pH
------------
time                   level description   location       pH   water_level
2015-08-18T00:00:00Z                       santa_monica   6
2015-08-18T00:00:00Z                       coyote_creek   7
[...]
2015-09-18T21:36:00Z                       santa_monica   8
2015-09-18T21:42:00Z                       santa_monica   7
```

该查询从两个measurement `h2o_feet`和`h2o_pH`中查询所有的field和tag，多个measurement之间用逗号`,`分割。

##### 例七：从完全限定的measurement中选择所有数据

```
> SELECT * FROM "NOAA_water_database"."autogen"."h2o_feet"

name: h2o_feet
--------------
time                   level description      location       water_level
2015-08-18T00:00:00Z   below 3 feet           santa_monica   2.064
2015-08-18T00:00:00Z   between 6 and 9 feet   coyote_creek   8.12
[...]
2015-09-18T21:36:00Z   between 3 and 6 feet   santa_monica   5.066
2015-09-18T21:42:00Z   between 3 and 6 feet   santa_monica   4.938
```

该查询选择数据库`NOAA_water_database`中的数据，`autogen`为存储策略，`h2o_feet`为measurement。

在CLI中，可以直接这样来代替`USE`指定数据库，以及指定`DEFAULT`之外的存储策略。 在HTTP API中，如果需要完全限定使用`db`和`rp`参数来指定。

##### 例八：从特定数据库中查询measurement的所有数据

```
> SELECT * FROM "NOAA_water_database".."h2o_feet"

name: h2o_feet
--------------
time                   level description      location       water_level
2015-08-18T00:00:00Z   below 3 feet           santa_monica   2.064
2015-08-18T00:00:00Z   between 6 and 9 feet   coyote_creek   8.12
[...]
2015-09-18T21:36:00Z   between 3 and 6 feet   santa_monica   5.066
2015-09-18T21:42:00Z   between 3 and 6 feet   santa_monica   4.938
```

该查询选择数据库`NOAA_water_database`中的数据，`DEFAULT`为存储策略和`h2o_feet`为measurement。 ..表示指定数据库的`DEFAULT`存储策略。

#### SELECT语句中常见的问题
##### 问题一：在SELECT语句中查询tag key
一个查询在`SELECT`子句中至少需要一个field key来返回数据。如果`SELECT`子句仅包含单个tag key或多个tag key，则查询返回一个空的结果。这是系统如何存储数据的结果。

例如：

下面的查询不会返回结果，因为在`SELECT`子句中只指定了一个tag key(`location`)：

```
> SELECT "location" FROM "h2o_feet"
>
```
要想有任何有关tag key为`location`的数据，`SELECT`子句中必须至少有一个field(`water_level`)：

```
> SELECT "water_level","location" FROM "h2o_feet" LIMIT 3
name: h2o_feet
time                   water_level  location
----                   -----------  --------
2015-08-18T00:00:00Z   8.12         coyote_creek
2015-08-18T00:00:00Z   2.064        santa_monica
[...]
2015-09-18T21:36:00Z   5.066        santa_monica
2015-09-18T21:42:00Z   4.938        santa_monica
```

### WHERE子句
`WHERE`子句用作field，tag和timestamp的过滤。
如果厌倦阅读，查看这个InfluxQL短片(*注意：可能看不到，请到原文处查看[https://docs.influxdata.com/influxdb/v1.3/query_language/data_exploration/#the-where-clause](https://docs.influxdata.com/influxdb/v1.3/query_language/data_exploration/#the-where-clause)*)：

<iframe src="https://player.vimeo.com/video/195058724?title=0&byline=0&portrait=0" width="60%" height="250px" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

#### 语法

```
SELECT_clause FROM_clause WHERE <conditional_expression> [(AND|OR) <conditional_expression> [...]]
```

#### 语法描述
`WHERE`子句在field，tag和timestamp上支持`conditional_expressions`.

##### fields

```
field_key <operator> ['string' | boolean | float | integer]
```

`WHERE`子句支持field value是字符串，布尔型，浮点数和整数这些类型。

在`WHERE`子句中单引号来表示字符串字段值。具有无引号字符串字段值或双引号字符串字段值的查询将不会返回任何数据，并且在大多数情况下也不会返回错误。

支持的操作符： 

`=` 等于  
`<>` 不等于  
`!=` 不等于  
`>` 大于  
`>=` 大于等于  
`<` 小于  
`<=` 小于等于

##### tags

```
tag_key <operator> ['tag_value']
```

`WHERE`子句中的用单引号来把tag value引起来。具有未用单引号的tag或双引号的tag查询将不会返回任何数据，并且在大多数情况下不会返回错误。

支持的操作符： 

`=` 等于  
`<>` 不等于  
`!=` 不等于  

##### timestamps
对于大多数`SELECT`语句，默认时间范围为UTC的`1677-09-21 00：12：43.145224194`到`2262-04-11T23：47：16.854775806Z`。 对于只有`GROUP BY time()`子句的`SELECT`语句，默认时间范围在UTC的`1677-09-21 00：12：43.145224194`和`now()`之间。

#### 例子
##### 例一：查询有特定field的key value的数据

```
> SELECT * FROM "h2o_feet" WHERE "water_level" > 8

name: h2o_feet
--------------
time                   level description      location       water_level
2015-08-18T00:00:00Z   between 6 and 9 feet   coyote_creek   8.12
2015-08-18T00:06:00Z   between 6 and 9 feet   coyote_creek   8.005
[...]
2015-09-18T00:12:00Z   between 6 and 9 feet   coyote_creek   8.189
2015-09-18T00:18:00Z   between 6 and 9 feet   coyote_creek   8.084
```

这个查询将会返回measurement为`h2o_feet`，字段`water_level`的值大于8的数据。

##### 例二：查询有特定field的key value为字符串的数据

```
> SELECT * FROM "h2o_feet" WHERE "level description" = 'below 3 feet'

name: h2o_feet
--------------
time                   level description   location       water_level
2015-08-18T00:00:00Z   below 3 feet        santa_monica   2.064
2015-08-18T00:06:00Z   below 3 feet        santa_monica   2.116
[...]
2015-09-18T14:06:00Z   below 3 feet        santa_monica   2.999
2015-09-18T14:36:00Z   below 3 feet        santa_monica   2.907
```

该查询从`h2o_feet`返回数据，其中`level description`等于`below 3 feet`。InfluxQL在`WHERE`子句中需要单引号来将字符串field value引起来。

##### 例三：查询有特定field的key value并且带计算的数据

```
> SELECT * FROM "h2o_feet" WHERE "water_level" + 2 > 11.9

name: h2o_feet
--------------
time                   level description           location       water_level
2015-08-29T07:06:00Z   at or greater than 9 feet   coyote_creek   9.902
2015-08-29T07:12:00Z   at or greater than 9 feet   coyote_creek   9.938
2015-08-29T07:18:00Z   at or greater than 9 feet   coyote_creek   9.957
2015-08-29T07:24:00Z   at or greater than 9 feet   coyote_creek   9.964
2015-08-29T07:30:00Z   at or greater than 9 feet   coyote_creek   9.954
2015-08-29T07:36:00Z   at or greater than 9 feet   coyote_creek   9.941
2015-08-29T07:42:00Z   at or greater than 9 feet   coyote_creek   9.925
2015-08-29T07:48:00Z   at or greater than 9 feet   coyote_creek   9.902
2015-09-02T23:30:00Z   at or greater than 9 feet   coyote_creek   9.902
```

该查询从`h2o_feet`返回数据，其字段值为`water_level`加上2大于11.9。

##### 例四：查询有特定tag的key value的数据

```
> SELECT "water_level" FROM "h2o_feet" WHERE "location" = 'santa_monica'

name: h2o_feet
--------------
time                   water_level
2015-08-18T00:00:00Z   2.064
2015-08-18T00:06:00Z   2.116
[...]
2015-09-18T21:36:00Z   5.066
2015-09-18T21:42:00Z   4.938
```

该查询从`h2o_feet`返回数据，其中tag `location`为`santa_monica`。InfluxQL需要`WHERE`子句中tag的过滤带单引号。

##### 例五：查询有特定tag的key value以及特定field的key value的数据

```
> SELECT "water_level" FROM "h2o_feet" WHERE "location" <> 'santa_monica' AND (water_level < -0.59 OR water_level > 9.95)

name: h2o_feet
--------------
time                   water_level
2015-08-29T07:18:00Z   9.957
2015-08-29T07:24:00Z   9.964
2015-08-29T07:30:00Z   9.954
2015-08-29T14:30:00Z   -0.61
2015-08-29T14:36:00Z   -0.591
2015-08-30T15:18:00Z   -0.594
```

该查询从`h2o_feet`中返回数据，其中tag `location`设置为`santa_monica`，并且field `water_level`的值小于-0.59或大于9.95。 `WHERE`子句支持运算符`AND`和`O`R，并支持用括号分隔逻辑。

##### 例六：根据时间戳来过滤数据

```
> SELECT * FROM "h2o_feet" WHERE time > now() - 7d
```

该查询返回来自`h2o_feet`，该measurement在过去七天内的数据。

#### WHERE子句常见的问题

##### 问题一：WHERE子句返回结果为空

在大多数情况下，这个问题是tag value或field value缺少单引号的结果。具有无引号或双引号tag value或field value的查询将不会返回任何数据，并且在大多数情况下不会返回错误。

下面的代码块中的前两个查询尝试指定tag value为`santa_monica`，没有任何引号和双引号。那些查询不会返回结果。 第三个查询单引号`santa_monica`（这是支持的语法），并返回预期的结果。

```
> SELECT "water_level" FROM "h2o_feet" WHERE "location" = santa_monica

> SELECT "water_level" FROM "h2o_feet" WHERE "location" = "santa_monica"

> SELECT "water_level" FROM "h2o_feet" WHERE "location" = 'santa_monica'

name: h2o_feet
--------------
time                   water_level
2015-08-18T00:00:00Z   2.064
[...]
2015-09-18T21:42:00Z   4.938
```

下面的代码块中的前两个查询尝试指定field字符串为`at or greater than 9 feet`，没有任何引号和双引号。第一个查询返回错误，因为field字符串包含空格。 第二个查询不返回结果。 第三个查询单引号`at or greater than 9 feet`（这是支持的语法），并返回预期结果。

```
> SELECT "level description" FROM "h2o_feet" WHERE "level description" = at or greater than 9 feet

ERR: error parsing query: found than, expected ; at line 1, char 86

> SELECT "level description" FROM "h2o_feet" WHERE "level description" = "at or greater than 9 feet"

> SELECT "level description" FROM "h2o_feet" WHERE "level description" = 'at or greater than 9 feet'

name: h2o_feet
--------------
time                   level description
2015-08-26T04:00:00Z   at or greater than 9 feet
[...]
2015-09-15T22:42:00Z   at or greater than 9 feet
```

## GROUP BY子句
`GROUP BY`子句后面可以跟用户指定的tags或者是一个时间间隔。

### GROUP BY tags
`GROUP BY <tag>`后面跟用户指定的tags。如果厌倦阅读，查看这个InfluxQL短片(*注意：可能看不到，请到原文处查看[https://docs.influxdata.com/influxdb/v1.3/query_language/data_exploration/#group-by-tags](https://docs.influxdata.com/influxdb/v1.3/query_language/data_exploration/#group-by-tags)*)：

<iframe src="https://player.vimeo.com/video/200898048?title=0&byline=0&portrait=0" width="60%" height="250px" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>


#### 语法

```
SELECT_clause FROM_clause [WHERE_clause] GROUP BY [* | <tag_key>[,<tag_key]]
```

#### 语法描述

`GROUP BY *`  
对结果中的所有tag作group by。

`GROUP BY <tag_key>`  
对结果按指定的tag作group by。

`GROUP BY <tag_key>,<tag_key>`  
对结果数据按多个tag作group by，其中tag key的顺序没所谓。

#### 例子
##### 例一：对单个tag作group by

```
> SELECT MEAN("water_level") FROM "h2o_feet" GROUP BY "location"

name: h2o_feet
tags: location=coyote_creek
time			               mean
----			               ----
1970-01-01T00:00:00Z	 5.359342451341401


name: h2o_feet
tags: location=santa_monica
time			               mean
----			               ----
1970-01-01T00:00:00Z	 3.530863470081006
```
上面的查询中用到了InfluxQL中的函数来计算measurement `h2o_feet`的每`location`的`water_level`的平均值。InfluxDB返回了两个series：分别是`location`的两个值。

>说明：在InfluxDB中，epoch 0(`1970-01-01T00:00:00Z`)通常用作等效的空时间戳。如果要求查询不返回时间戳，例如无限时间范围的聚合函数，InfluxDB将返回epoch 0作为时间戳。

##### 例二：对多个tag作group by

```
> SELECT MEAN("index") FROM "h2o_quality" GROUP BY location,randtag

name: h2o_quality
tags: location=coyote_creek, randtag=1
time                  mean
----                  ----
1970-01-01T00:00:00Z  50.69033760186263

name: h2o_quality
tags: location=coyote_creek, randtag=2
time                   mean
----                   ----
1970-01-01T00:00:00Z   49.661867544220485

name: h2o_quality
tags: location=coyote_creek, randtag=3
time                   mean
----                   ----
1970-01-01T00:00:00Z   49.360939907550076

name: h2o_quality
tags: location=santa_monica, randtag=1
time                   mean
----                   ----
1970-01-01T00:00:00Z   49.132712456344585

name: h2o_quality
tags: location=santa_monica, randtag=2
time                   mean
----                   ----
1970-01-01T00:00:00Z   50.2937984496124

name: h2o_quality
tags: location=santa_monica, randtag=3
time                   mean
----                   ----
1970-01-01T00:00:00Z   49.99919903884662
```
上面的查询中用到了InfluxQL中的函数来计算measurement `h2o_quality`的每个`location`和`randtag`的`Index`的平均值。在`GROUP BY`子句中用逗号来分割多个tag。

##### 例三：对所有tag作group by
```
> SELECT MEAN("index") FROM "h2o_quality" GROUP BY *

name: h2o_quality
tags: location=coyote_creek, randtag=1
time			               mean
----			               ----
1970-01-01T00:00:00Z	 50.55405446521169


name: h2o_quality
tags: location=coyote_creek, randtag=2
time			               mean
----			               ----
1970-01-01T00:00:00Z	 50.49958856271162


name: h2o_quality
tags: location=coyote_creek, randtag=3
time			               mean
----			               ----
1970-01-01T00:00:00Z	 49.5164137518956


name: h2o_quality
tags: location=santa_monica, randtag=1
time			               mean
----			               ----
1970-01-01T00:00:00Z	 50.43829082296367


name: h2o_quality
tags: location=santa_monica, randtag=2
time			               mean
----			               ----
1970-01-01T00:00:00Z	 52.0688508894012


name: h2o_quality
tags: location=santa_monica, randtag=3
time			               mean
----			               ----
1970-01-01T00:00:00Z	 49.29386362086556
```
上面的查询中用到了InfluxQL中的函数来计算measurement `h2o_quality`的每个tag的`Index`的平均值。

请注意，查询结果与例二中的查询结果相同，其中我们明确指定了tag key `location`和`randtag`。 这是因为measurement `h2o_quality`中只有这两个tag key。

### GROUP BY时间间隔
`GROUP BY time()`返回结果按指定的时间间隔group by。

#### 基本的GROUP BY time()语法
##### 语法
```
SELECT <function>(<field_key>) FROM_clause WHERE <time_range> GROUP BY time(<time_interval>),[tag_key] [fill(<fill_option>)]
```

##### 基本语法描述
基本`GROUP BY time()`查询需要`SELECT`子句中的InfluxQL函数和`WHERE`子句中的时间范围。请注意，`GROUP BY`子句必须在`WHERE`子句之后。

`time(time_interval)`  
`GROUP BY time()`语句中的`time_interval`是一个时间duration。决定了InfluxDB按什么时间间隔group by。例如：`time_interval`为`5m`则在`WHERE`子句中指定的时间范围内将查询结果分到五分钟时间组里。

`fill(<fill_option>)`   
`fill（<fill_option>）`是可选的。它会更改不含数据的时间间隔的返回值。

覆盖范围：基本`GROUP BY time()`查询依赖于`time_interval`和InfluxDB的预设时间边界来确定每个时间间隔中包含的原始数据以及查询返回的时间戳。

#### 基本语法示例










