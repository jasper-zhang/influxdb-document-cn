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

##### 基本语法示例
下面的例子用到的示例数据如下：

```
> SELECT "water_level","location" FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z'

name: h2o_feet
--------------
time                   water_level   location
2015-08-18T00:00:00Z   8.12          coyote_creek
2015-08-18T00:00:00Z   2.064         santa_monica
2015-08-18T00:06:00Z   8.005         coyote_creek
2015-08-18T00:06:00Z   2.116         santa_monica
2015-08-18T00:12:00Z   7.887         coyote_creek
2015-08-18T00:12:00Z   2.028         santa_monica
2015-08-18T00:18:00Z   7.762         coyote_creek
2015-08-18T00:18:00Z   2.126         santa_monica
2015-08-18T00:24:00Z   7.635         coyote_creek
2015-08-18T00:24:00Z   2.041         santa_monica
2015-08-18T00:30:00Z   7.5           coyote_creek
2015-08-18T00:30:00Z   2.051         santa_monica
```

##### 例一：时间间隔为12分钟的group by
```
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m)

name: h2o_feet
--------------
time                   count
2015-08-18T00:00:00Z   2
2015-08-18T00:12:00Z   2
2015-08-18T00:24:00Z   2
```

该查询使用InfluxQL函数来计算`location=coyote_creek`的`water_level`数，并将其分组结果分为12分钟间隔。每个时间戳的结果代表一个12分钟的间隔。 第一个时间戳记的计数涵盖大于`2015-08-18T00：00：00Z`的原始数据，但小于且不包括`2015-08-18T00：12：00Z`。第二时间戳的计数涵盖大于`2015-08-18T00：12：00Z`的原始数据，但小于且不包括`2015-08-18T00：24：00Z`。

##### 例二：时间间隔为12分钟并且还对tag key作group by
```
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m),"location"

name: h2o_feet
tags: location=coyote_creek
time                   count
----                   -----
2015-08-18T00:00:00Z   2
2015-08-18T00:12:00Z   2
2015-08-18T00:24:00Z   2

name: h2o_feet
tags: location=santa_monica
time                   count
----                   -----
2015-08-18T00:00:00Z   2
2015-08-18T00:12:00Z   2
2015-08-18T00:24:00Z   2
```

该查询使用InfluxQL函数来计算`water_leval`的数量。它将结果按`location`分组并分隔12分钟。请注意，时间间隔和tag key在`GROUP BY`子句中以逗号分隔。查询返回两个measurement的结果：针对tag `location`的每个值。每个时间戳的结果代表一个12分钟的间隔。第一个时间戳记的计数涵盖大于`2015-08-18T00：00：00Z`的原始数据，但小于且不包括`2015-08-18T00：12：00Z`。第二时间戳的计数涵盖大于`2015-08-18T00：12：00Z`原始数据，但小于且不包括`2015-08-18T00：24：00Z`。

##### 基本语法的共同问题
##### 在查询结果中出现意想不到的时间戳和值
使用基本语法，InfluxDB依赖于`GROUP BY time()`间隔和系统预设时间边界来确定每个时间间隔中包含的原始数据以及查询返回的时间戳。 在某些情况下，这可能会导致意想不到的结果。

原始值：

```
> SELECT "water_level" FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:18:00Z'
name: h2o_feet
--------------
time                   water_level
2015-08-18T00:00:00Z   8.12
2015-08-18T00:06:00Z   8.005
2015-08-18T00:12:00Z   7.887
2015-08-18T00:18:00Z   7.762
```

查询和结果：

以下查询涵盖12分钟的时间范围，并将结果分组为12分钟的时间间隔，但返回两个结果：

```
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:06:00Z' AND time < '2015-08-18T00:18:00Z' GROUP BY time(12m)

name: h2o_feet
time                   count
----                   -----
2015-08-18T00:00:00Z   1        <----- 请注意，此时间戳记的发生在查询时间范围最小值之前
2015-08-18T00:12:00Z   1
```

解释：

InfluxDB使用独立于`WHERE`子句中任何时间条件的`GROUP BY`间隔的预设的四舍五入时间边界。当计算结果时，所有返回的数据必须在查询的显式时间范围内发生，但`GROUP BY`间隔将基于预设的时间边界。

下表显示了结果中预设时间边界，相关`GROUP BY time()`间隔，包含的点以及每个`GROUP BY time()`间隔的返回时间戳。


时间间隔序号|预设的时间边界|`GROUP BY time()`间隔|包含的数据点|返回的时间戳
---|---|---|---|---
1|`time >= 2015-08-18T00:00:00Z AND time < 2015-08-18T00:12:00Z`|`time >= 2015-08-18T00:06:00Z AND time < 2015-08-18T00:12:00Z`|`8.005`|`2015-08-18T00:00:00Z`
2|`time >= 2015-08-12T00:12:00Z AND time < 2015-08-18T00:24:00Z`|`time >= 2015-08-12T00:12:00Z AND time < 2015-08-18T00:18:00Z`|`7.887`|`2015-08-18T00:12:00Z`

第一个预设的12分钟时间边界从0`0:00`开始，在`00:12`之前结束。只有一个数据点（`8.005`）落在查询的第一个`GROUP BY time()`间隔内，并且在第一个时间边界。请注意，虽然返回的时间戳在查询的时间范围开始之前发生，但查询结果排除了查询时间范围之前发生的数据。

第二个预设的12分钟时间边界从`00:12`开始，在`00:24`之前结束。 只有一个原点（`7.887`）都在查询的第二个`GROUP BY time()`间隔内，在该第二个时间边界内。

高级`GROUP BY time()`语法允许用户移动InfluxDB预设时间边界的开始时间。高级语法部分中的例三将继续显示此处的查询; 它将预设时间边界向前移动六分钟，以便InfluxDB返回：

```
name: h2o_feet
time                   count
----                   -----
2015-08-18T00:06:00Z   2
```

#### 高级`GROUP BY time()`语法
##### 语法

```
SELECT <function>(<field_key>) FROM_clause WHERE <time_range> GROUP BY time(<time_interval>,<offset_interval>),[tag_key] [fill(<fill_option>)]
``` 

##### 高级语法描述
高级`GROUP BY time()`查询需要`SELECT`子句中的InfluxQL函数和`WHERE`子句中的时间范围。 请注意，`GROUP BY`子句必须在`WHERE`子句之后。

`time(time_interval,offset_interval)`

`offset_interval`是一个持续时间。它向前或向后移动InfluxDB的预设时间界限。`offset_interval`可以为正或负。

`fill(<fill_option>)`

`fill(<fill_option>)`是可选的。它会更改不含数据的时间间隔的返回值。

范围

高级`GROUP BY time()`查询依赖于`time_interval`，`offset_interval`和InfluxDB的预设时间边界，以确定每个时间间隔中包含的原始数据以及查询返回的时间戳。

##### 高级语法的例子
下面例子都使用这份示例数据：

```
> SELECT "water_level" FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:54:00Z'

name: h2o_feet
--------------
time                   water_level
2015-08-18T00:00:00Z   8.12
2015-08-18T00:06:00Z   8.005
2015-08-18T00:12:00Z   7.887
2015-08-18T00:18:00Z   7.762
2015-08-18T00:24:00Z   7.635
2015-08-18T00:30:00Z   7.5
2015-08-18T00:36:00Z   7.372
2015-08-18T00:42:00Z   7.234
2015-08-18T00:48:00Z   7.11
2015-08-18T00:54:00Z   6.982
```

##### 例一：查询结果间隔按18分钟group by，并将预设时间边界向前移动
```
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:06:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(18m,6m)

name: h2o_feet
time                   mean
----                   ----
2015-08-18T00:06:00Z   7.884666666666667
2015-08-18T00:24:00Z   7.502333333333333
2015-08-18T00:42:00Z   7.108666666666667
```

该查询使用InfluxQL函数来计算平均`water_level`，将结果分组为18分钟的时间间隔，并将预设时间边界偏移六分钟。

没有`offset_interval`的查询的时间边界和返回的时间戳符合InfluxDB的预设时间边界。我们先来看看没有`offset_interval`的结果：

```
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:06:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(18m)

name: h2o_feet
time                   mean
----                   ----
2015-08-18T00:00:00Z   7.946
2015-08-18T00:18:00Z   7.6323333333333325
2015-08-18T00:36:00Z   7.238666666666667
2015-08-18T00:54:00Z   6.982
```

没有`offset_interval`的查询的时间边界和返回的时间戳符合InfluxDB的预设时间界限：

时间间隔序号|预设的时间边界|`GROUP BY time()`间隔|包含的数据点|返回的时间戳
---|---|---|---|---
1|`time >= 2015-08-18T00:00:00Z AND time < 2015-08-18T00:18:00Z`|`time >= 2015-08-18T00:06:00Z AND time < 2015-08-18T00:18:00Z`|`8.005,7.887`|`2015-08-18T00:00:00Z`
2|`time >= 2015-08-18T00:18:00Z AND time < 2015-08-18T00:36:00Z`|`同坐`|`7.762,7.635,7.5`|`2015-08-18T00:18:00Z`
3|`time >= 2015-08-18T00:36:00Z AND time < 2015-08-18T00:54:00Z`|`同左`|`7.372,7.234,7.11`|`2015-08-18T00:36:00Z`
4|`time >= 2015-08-18T00:54:00Z AND time < 2015-08-18T01:12:00Z`|`time = 2015-08-18T00:54:00Z`|`6.982`|`2015-08-18T00:54:00Z`

第一个预设的18分钟时间边界从`00:00`开始，在`00:18`之前结束。 两个点（`8.005和7.887`）都落在第一个`GROUP BY time()`间隔内，并且在第一个时间边界。请注意，虽然返回的时间戳在查询的时间范围开始之前发生，但查询结果排除了查询时间范围之前发生的数据。

第二个预设的18分钟时间边界从`00:18`开始，在`00:36`之前结束。 三个点（`7.762和7.635和7.5`）都落在第二个`GROUP BY time()`间隔内，在第二个时间边界。 在这种情况下，边界时间范围和间隔时间范围是相同的。 

第四个预设的18分钟时间边界从`00:54`开始，在`1:12:00`之前结束。 一个点（`6.982`）落在第四个`GROUP BY time()`间隔内，在第四个时间边界。 

具有`offset_interval`的查询的时间边界和返回的时间戳符合偏移时间边界：

时间间隔序号|预设的时间边界|`GROUP BY time()`间隔|包含的数据点|返回的时间戳
---|---|---|---|---
1|`time >= 2015-08-18T00:06:00Z AND time < 2015-08-18T00:24:00Z`|`同左`|`8.005,7.887,7.762`|`2015-08-18T00:06:00Z`
2|`time >= 2015-08-18T00:24:00Z AND time < 2015-08-18T00:42:00Z`|`同坐`|`7.635,7.5,7.372`|`2015-08-18T00:24:00Z`
3|`time >= 2015-08-18T00:42:00Z AND time < 2015-08-18T01:00:00Z`|`同左`|`7.234,7.11,6.982`|`2015-08-18T00:42:00Z`
4|`time >= 2015-08-18T01:00:00Z AND time < 2015-08-18T01:18:00Z`|`无`|`无`|`无`

六分钟偏移间隔向前移动预设边界的时间范围，使得边界时间范围和相关`GROUP BY time()`间隔时间范围始终相同。使用偏移量，每个间隔对三个点执行计算，返回的时间戳与边界时间范围的开始和`GROUP BY time()`间隔时间范围的开始匹配。

请注意，`offset_interval`强制第四个时间边界超出查询的时间范围，因此查询不会返回该最后一个间隔的结果。

##### 例二：查询结果按12分钟间隔group by，并将预设时间界限向后移动
```
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:06:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(18m,-12m)

name: h2o_feet
time                   mean
----                   ----
2015-08-18T00:06:00Z   7.884666666666667
2015-08-18T00:24:00Z   7.502333333333333
2015-08-18T00:42:00Z   7.108666666666667
```

该查询使用InfluxQL函数来计算平均`water_level`，将结果分组为18分钟的时间间隔，并将预设时间边界偏移-12分钟。

>注意：例二中的查询返回与例一中的查询相同的结果，但例二中的查询使用负的`offset_interval`而不是正的`offset_interval`。 两个查询之间没有性能差异; 在确定正负`offset_intervel`之间时，请任意选择最直观的选项。

没有`offset_interval`的查询的时间边界和返回的时间戳符合InfluxDB的预设时间边界。 我们首先检查没有偏移量的结果：

```
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:06:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(18m)

name: h2o_feet
time                    mean
----                    ----
2015-08-18T00:00:00Z    7.946
2015-08-18T00:18:00Z    7.6323333333333325
2015-08-18T00:36:00Z    7.238666666666667
2015-08-18T00:54:00Z    6.982
```

没有`offset_interval`的查询的时间边界和返回的时间戳符合InfluxDB的预设时间界限：

时间间隔序号|预设的时间边界|`GROUP BY time()`间隔|包含的数据点|返回的时间戳
---|---|---|---|---
1|`time >= 2015-08-18T00:00:00Z AND time < 2015-08-18T00:18:00Z`|`time >= 2015-08-18T00:06:00Z AND time < 2015-08-18T00:18:00Z`|`8.005,7.887`|`2015-08-18T00:00:00Z`
2|`time >= 2015-08-18T00:18:00Z AND time < 2015-08-18T00:36:00Z`|`同坐`|`7.762,7.635,7.5`|`2015-08-18T00:18:00Z`
3|`time >= 2015-08-18T00:36:00Z AND time < 2015-08-18T00:54:00Z`|`同左`|`7.372,7.234,7.11`|`2015-08-18T00:36:00Z`
4|`time >= 2015-08-18T00:54:00Z AND time < 2015-08-18T01:12:00Z`|`time = 2015-08-18T00:54:00Z`|`6.982`|`2015-08-18T00:54:00Z`

第一个预设的18分钟时间边界从`00:00`开始，在`00:18`之前结束。 两个点（`8.005和7.887`）都落在第一个`GROUP BY time()`间隔内，并且在第一个时间边界。请注意，虽然返回的时间戳在查询的时间范围开始之前发生，但查询结果排除了查询时间范围之前发生的数据。

第二个预设的18分钟时间边界从`00:18`开始，在`00:36`之前结束。 三个点（`7.762和7.635和7.5`）都落在第二个`GROUP BY time()`间隔内，在第二个时间边界。 在这种情况下，边界时间范围和间隔时间范围是相同的。 

第四个预设的18分钟时间边界从`00:54`开始，在`1:12:00`之前结束。 一个点（`6.982`）落在第四个`GROUP BY time()`间隔内，在第四个时间边界。 

具有`offset_interval`的查询的时间边界和返回的时间戳符合偏移时间边界：

时间间隔序号|预设的时间边界|`GROUP BY time()`间隔|包含的数据点|返回的时间戳
---|---|---|---|---
1|`time >= 2015-08-17T23:48:00Z AND time < 2015-08-18T00:06:00Z`|`无`|`无`|`无`
2|`time >= 2015-08-18T00:06:00Z AND time < 2015-08-18T00:24:00Z`|`同左`|`8.005,7.887,7.762`|`2015-08-18T00:06:00Z`
3|`time >= 2015-08-18T00:24:00Z AND time < 2015-08-18T00:42:00Z`|`同坐`|`7.635,7.5,7.372`|`2015-08-18T00:24:00Z`
4|`time >= 2015-08-18T00:42:00Z AND time < 2015-08-18T01:00:00Z`|`同左`|`7.234,7.11,6.982`|`2015-08-18T00:42:00Z`

负十二分钟偏移间隔向后移动预设边界的时间范围，使得边界时间范围和相关`GROUP BY time()`间隔时间范围始终相同。使用偏移量，每个间隔对三个点执行计算，返回的时间戳与边界时间范围的开始和`GROUP BY time()`间隔时间范围的开始匹配。

请注意，`offset_interval`强制第一个时间边界超出查询的时间范围，因此查询不会返回该最后一个间隔的结果。

##### 例三：查询结果按12分钟间隔group by，并将预设时间边界向前移动
这个例子是上面*基本语法的问题*的继续

```
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:06:00Z' AND time < '2015-08-18T00:18:00Z' GROUP BY time(12m,6m)

name: h2o_feet
time                   count
----                   -----
2015-08-18T00:06:00Z   2
```

该查询使用InfluxQL函数来计算平均`water_level`，将结果分组为12分钟的时间间隔，并将预设时间边界偏移六分钟。

没有`offset_interval`的查询的时间边界和返回的时间戳符合InfluxDB的预设时间边界。我们先来看看没有`offset_interval`的结果：

```
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:06:00Z' AND time < '2015-08-18T00:18:00Z' GROUP BY time(12m)

name: h2o_feet
time                   count
----                   -----
2015-08-18T00:00:00Z   1
2015-08-18T00:12:00Z   1
```

没有`offset_interval`的查询的时间边界和返回的时间戳符合InfluxDB的预设时间界限：

时间间隔序号|预设的时间边界|`GROUP BY time()`间隔|包含的数据点|返回的时间戳
---|---|---|---|---
1|`time >= 2015-08-18T00:00:00Z AND time < 2015-08-18T00:12:00Z`|`time >= 2015-08-18T00:06:00Z AND time < 2015-08-18T00:12:00Z`|`8.005`|`2015-08-18T00:00:00Z`
2|`time >= 2015-08-12T00:12:00Z AND time < 2015-08-18T00:24:00Z`|`time >= 2015-08-12T00:12:00Z AND time < 2015-08-18T00:18:00Z`|`7.887`|`2015-08-18T00:12:00Z`

第一个预设的12分钟时间边界从0`0:00`开始，在`00:12`之前结束。只有一个数据点（`8.005`）落在查询的第一个`GROUP BY time()`间隔内，并且在第一个时间边界。请注意，虽然返回的时间戳在查询的时间范围开始之前发生，但查询结果排除了查询时间范围之前发生的数据。

第二个预设的12分钟时间边界从`00:12`开始，在`00:24`之前结束。 只有一个原点（`7.887`）都在查询的第二个`GROUP BY time()`间隔内，在该第二个时间边界内。

具有`offset_interval`的查询的时间边界和返回的时间戳符合偏移时间边界：

时间间隔序号|预设的时间边界|`GROUP BY time()`间隔|包含的数据点|返回的时间戳
---|---|---|---|---
1|`time >= 2015-08-18T00:06:00Z AND time < 2015-08-18T00:18:00Z`|`同左`|`8.005，7.887`|`2015-08-18T00:06:00Z`
2|`time >= 2015-08-18T00:18:00Z AND time < 2015-08-18T00:30:00Z`|`无`|`无`|`无`

六分钟偏移间隔向前移动预设边界的时间范围，使得边界时间范围和相关`GROUP BY time()`间隔时间范围始终相同。使用偏移量，每个间隔对三个点执行计算，返回的时间戳与边界时间范围的开始和`GROUP BY time()`间隔时间范围的开始匹配。

请注意，`offset_interval`强制第二个时间边界超出查询的时间范围，因此查询不会返回该最后一个间隔的结果。

#### GROUP BY time()加fill()
`fill()`更改不含数据的时间间隔的返回值。 

##### 语法

```
SELECT <function>(<field_key>) FROM_clause WHERE <time_range> GROUP BY time(time_interval,[<offset_interval])[,tag_key] [fill(<fill_option>)]
```

##### 语法描述
默认情况下，没有数据的`GROUP BY time()`间隔返回为null作为输出列中的值。`fill()`更改不含数据的时间间隔返回的值。请注意，如果`GROUP(ing)BY`多个对象（例如，tag和时间间隔），那么`fill()`必须位于`GROUP BY`子句的末尾。

fill的参数

* 任一数值：用这个数字返回没有数据点的时间间隔
* linear：返回没有数据的时间间隔的[线性插值](https://en.wikipedia.org/wiki/Linear_interpolation)结果。
* none: 不返回在时间间隔里没有点的数据
* previous：返回时间隔间的前一个间隔的数据

##### 例子：
##### 例一：fill(100)
不带`fill(100)`:

```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-09-18T16:00:00Z' AND time <= '2015-09-18T16:42:00Z' GROUP BY time(12m)

name: h2o_feet
--------------
time                   max
2015-09-18T16:00:00Z   3.599
2015-09-18T16:12:00Z   3.402
2015-09-18T16:24:00Z   3.235
2015-09-18T16:36:00Z   
```

带`fill(100)`:

```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-09-18T16:00:00Z' AND time <= '2015-09-18T16:42:00Z' GROUP BY time(12m) fill(100)

name: h2o_feet
--------------
time                   max
2015-09-18T16:00:00Z   3.599
2015-09-18T16:12:00Z   3.402
2015-09-18T16:24:00Z   3.235
2015-09-18T16:36:00Z   100
```

##### 例二：fill(linear)

不带`fill(linear)`:

```
> SELECT MEAN("tadpoles") FROM "pond" WHERE time >= '2016-11-11T21:00:00Z' AND time <= '2016-11-11T22:06:00Z' GROUP BY time(12m)

name: pond
time                   mean
----                   ----
2016-11-11T21:00:00Z   1
2016-11-11T21:12:00Z
2016-11-11T21:24:00Z   3
2016-11-11T21:36:00Z
2016-11-11T21:48:00Z
2016-11-11T22:00:00Z   6  
```

带`fill(linear)`:

```
> SELECT MEAN("tadpoles") FROM "pond" WHERE time >= '2016-11-11T21:00:00Z' AND time <= '2016-11-11T22:06:00Z' GROUP BY time(12m) fill(linear)

name: pond
time                   mean
----                   ----
2016-11-11T21:00:00Z   1
2016-11-11T21:12:00Z   2
2016-11-11T21:24:00Z   3
2016-11-11T21:36:00Z   4
2016-11-11T21:48:00Z   5
2016-11-11T22:00:00Z   6
```

##### 例三：fill(none)

不带`fill(none)`:

```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-09-18T16:00:00Z' AND time <= '2015-09-18T16:42:00Z' GROUP BY time(12m)

name: h2o_feet
--------------
time                   max
2015-09-18T16:00:00Z   3.599
2015-09-18T16:12:00Z   3.402
2015-09-18T16:24:00Z   3.235
2015-09-18T16:36:00Z 
```

带`fill(none)`:

```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-09-18T16:00:00Z' AND time <= '2015-09-18T16:42:00Z' GROUP BY time(12m) fill(none)

name: h2o_feet
--------------
time                   max
2015-09-18T16:00:00Z   3.599
2015-09-18T16:12:00Z   3.402
2015-09-18T16:24:00Z   3.235
```

##### 例四：fill(null)

不带`fill(null)`:

```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-09-18T16:00:00Z' AND time <= '2015-09-18T16:42:00Z' GROUP BY time(12m)

name: h2o_feet
--------------
time                   max
2015-09-18T16:00:00Z   3.599
2015-09-18T16:12:00Z   3.402
2015-09-18T16:24:00Z   3.235
2015-09-18T16:36:00Z
```

带`fill(null)`:

```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-09-18T16:00:00Z' AND time <= '2015-09-18T16:42:00Z' GROUP BY time(12m) fill(null)

name: h2o_feet
--------------
time                   max
2015-09-18T16:00:00Z   3.599
2015-09-18T16:12:00Z   3.402
2015-09-18T16:24:00Z   3.235
2015-09-18T16:36:00Z   
```
##### 例五：fill(previous)

不带`fill(previous)`:

```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-09-18T16:00:00Z' AND time <= '2015-09-18T16:42:00Z' GROUP BY time(12m)

name: h2o_feet
--------------
time                   max
2015-09-18T16:00:00Z   3.599
2015-09-18T16:12:00Z   3.402
2015-09-18T16:24:00Z   3.235
2015-09-18T16:36:00Z
```

带`fill(previous)`:

```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-09-18T16:00:00Z' AND time <= '2015-09-18T16:42:00Z' GROUP BY time(12m) fill(previous)

name: h2o_feet
--------------
time                   max
2015-09-18T16:00:00Z   3.599
2015-09-18T16:12:00Z   3.402
2015-09-18T16:24:00Z   3.235
2015-09-18T16:36:00Z   3.235
```

##### `fill()`的问题
##### 问题一：`fill()`当没有数据在查询时间范围内时
目前，如果查询的时间范围内没有任何数据，查询会忽略`fill()`。 这是预期的行为。GitHub上的一个开放[feature request](https://github.com/influxdata/influxdb/issues/6967)建议，即使查询的时间范围不包含数据，`fill()`也会强制返回值。

例子：

以下查询不返回数据，因为`water_level`在查询的时间范围内没有任何点。 请注意，`fill(800)`对查询结果没有影响。

```
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE "location" = 'coyote_creek' AND time >= '2015-09-18T22:00:00Z' AND time <= '2015-09-18T22:18:00Z' GROUP BY time(12m) fill(800)
>
```
##### 问题二：`fill(previous)`当前一个结果超出查询时间范围
当前一个结果超出查询时间范围，`fill(previous)`不会填充这个时间间隔。

例子：

以下查询涵盖`2015-09-18T16：24：00Z`和`2015-09-18T16：54：00Z`之间的时间范围。 请注意，`fill(previos)`用`2015-09-18T16：24：00Z`的结果填写到了`2015-09-18T16：36：00Z`中。

```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE location = 'coyote_creek' AND time >= '2015-09-18T16:24:00Z' AND time <= '2015-09-18T16:54:00Z' GROUP BY time(12m) fill(previous)

name: h2o_feet
--------------
time                   max
2015-09-18T16:24:00Z   3.235
2015-09-18T16:36:00Z   3.235
2015-09-18T16:48:00Z   4
```

下一个查询会缩短上一个查询的时间范围。 它现在涵盖`2015-09-18T16：36：00Z`和`2015-09-18T16：54：00Z`之间的时间。请注意，`fill(previos)`不会用`2015-09-18T16：24：00Z`的结果填写到`2015-09-18T16：36：00Z`中。因为`2015-09-18T16：24：00Z`的结果在查询的较短时间范围之外。

```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE location = 'coyote_creek' AND time >= '2015-09-18T16:36:00Z' AND time <= '2015-09-18T16:54:00Z' GROUP BY time(12m) fill(previous)

name: h2o_feet
--------------
time                   max
2015-09-18T16:36:00Z
2015-09-18T16:48:00Z   4
```

##### 问题三：`fill(linear)`当前一个结果超出查询时间范围
当前一个结果超出查询时间范围，`fill(linear)`不会填充这个时间间隔。

例子：

以下查询涵盖`2016-11-11T21:24:00Z`和`2016-11-11T22:06:00Z`之间的时间范围。请注意，`fill(linear)`使用`2016-11-11T21：24：00Z`到`2016-11-11T22：00：00Z`时间间隔的值，填充到`2016-11-11T21：36：00Z`到`2016-11-11T21：48：00Z`时间间隔中。

```
> SELECT MEAN("tadpoles") FROM "pond" WHERE time > '2016-11-11T21:24:00Z' AND time <= '2016-11-11T22:06:00Z' GROUP BY time(12m) fill(linear)

name: pond
time                   mean
----                   ----
2016-11-11T21:24:00Z   3
2016-11-11T21:36:00Z   4
2016-11-11T21:48:00Z   5
2016-11-11T22:00:00Z   6
```

下一个查询会缩短上一个查询的时间范围。 它现在涵盖`2016-11-11T21:36:00Z`和`2016-11-11T22:06:00Z`之间的时间。请注意，`fill()`不会使用`2016-11-11T21：24：00Z`到`2016-11-11T22：00：00Z`时间间隔的值，填充到`2016-11-11T21：36：00Z`到`2016-11-11T21：48：00Z`时间间隔中。因为`2015-09-18T16：24：00Z`的结果在查询的较短时间范围之外。

```
> SELECT MEAN("tadpoles") FROM "pond" WHERE time >= '2016-11-11T21:36:00Z' AND time <= '2016-11-11T22:06:00Z' GROUP BY time(12m) fill(linear)
name: pond
time                   mean
----                   ----
2016-11-11T21:36:00Z
2016-11-11T21:48:00Z
2016-11-11T22:00:00Z   6
```

## INTO子句
`INTO`子句将查询的结果写入到用户自定义的measurement中。

### 语法

```
SELECT_clause INTO <measurement_name> FROM_clause [WHERE_clause] [GROUP_BY_clause]
```

### 语法描述
`INTO`支持多种格式的measurement。

`INTO <measurement_name>`

写入到特定measurement中，用CLI时，写入到用`USE`指定的数据库，保留策略为`DEFAULT`，用HTTP API时，写入到`db`参数指定的数据库，保留策略为`DEFAULT`。

`INTO <database_name>.<retention_policy_name>.<measurement_name>`

写入到完整指定的measurement中。

`INTO <database_name>..<measurement_name>`

写入到指定数据库保留策略为`DEFAULT`。

`INTO <database_name>.<retention_policy_name>.:MEASUREMENT FROM /<regular_expression>/`

将数据写入与`FROM`子句中正则表达式匹配的用户指定数据库和保留策略的所有measurement。 `:MEASUREMENT`是对`FROM`子句中匹配的每个measurement的反向引用。

### 例子
#### 例一：重命名数据库
```
> SELECT * INTO "copy_NOAA_water_database"."autogen".:MEASUREMENT FROM "NOAA_water_database"."autogen"./.*/ GROUP BY *

name: result
time written
---- -------
0    76290
```

在InfluxDB中直接重命名数据库是不可能的，因此`INTO`子句的常见用途是将数据从一个数据库移动到另一个数据库。 上述查询将`NOAA_water_database`和`autogen`保留策略中的所有数据写入`copy_NOAA_water_database`数据库和`autogen`保留策略中。

反向引用语法（`:MEASUREMENT`）维护目标数据库中的源measurement名称。 请注意，在运行`INTO`查询之前，`copy_NOAA_water_database`数据库及其`autogen`保留策略都必须存在。

 `GROUP BY *`子句将源数据库中的tag留在目标数据库中的tag中。以下查询不为tag维护series的上下文;tag将作为field存储在目标数据库（`copy_NOAA_water_database`）中：

```
SELECT * INTO "copy_NOAA_water_database"."autogen".:MEASUREMENT FROM "NOAA_water_database"."autogen"./.*/
```

当移动大量数据时，我们建议在`WHERE`子句中顺序运行不同measurement的`INTO`查询并使用时间边界。这样可以防止系统内存不足。下面的代码块提供了这些查询的示例语法：

```
SELECT * 
INTO <destination_database>.<retention_policy_name>.<measurement_name> 
FROM <source_database>.<retention_policy_name>.<measurement_name>
WHERE time > now() - 100w and time < now() - 90w GROUP BY *

SELECT * 
INTO <destination_database>.<retention_policy_name>.<measurement_name> 
FROM <source_database>.<retention_policy_name>.<measurement_name>} 
WHERE time > now() - 90w  and time < now() - 80w GROUP BY *

SELECT * 
INTO <destination_database>.<retention_policy_name>.<measurement_name> 
FROM <source_database>.<retention_policy_name>.<measurement_name>
WHERE time > now() - 80w  and time < now() - 70w GROUP BY *
```

#### 例二：将查询结果写入到一个measurement

```
> SELECT "water_level" INTO "h2o_feet_copy_1" FROM "h2o_feet" WHERE "location" = 'coyote_creek'

name: result
------------
time                   written
1970-01-01T00:00:00Z   7604

> SELECT * FROM "h2o_feet_copy_1"

name: h2o_feet_copy_1
---------------------
time                   water_level
2015-08-18T00:00:00Z   8.12
[...]
2015-09-18T16:48:00Z   4
```

该查询将其结果写入新的measurement：`h2o_feet_copy_1`。如果使用CLI，InfluxDB会将数据写入`USE`d数据库和`DEFAULT`保留策略。 如果您使用HTTP API，InfluxDB会将数据写入参数`db`指定的数据库和`rp`指定的保留策略。如果您没有设置`rp`参数，HTTP API将自动将数据写入数据库的`DEFAULT`保留策略。

响应显示InfluxDB写入`h2o_feet_copy_1`的点数（7605）。 响应中的时间戳是无意义的; InfluxDB使用epoch 0（`1970-01-01T00：00：00Z`）作为空时间戳等价物。

#### 例三：将查询结果写入到一个完全指定的measurement中
```
> SELECT "water_level" INTO "where_else"."autogen"."h2o_feet_copy_2" FROM "h2o_feet" WHERE "location" = 'coyote_creek'

name: result
------------
time                   written
1970-01-01T00:00:00Z   7604

> SELECT * FROM "where_else"."autogen"."h2o_feet_copy_2"

name: h2o_feet_copy_2
---------------------
time                   water_level
2015-08-18T00:00:00Z   8.12
[...]
2015-09-18T16:48:00Z   4
```

#### 例四：将聚合结果写入到一个measurement中(采样)
```
> SELECT MEAN("water_level") INTO "all_my_averages" FROM "h2o_feet" WHERE "location" = 'coyote_creek' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m)

name: result
------------
time                   written
1970-01-01T00:00:00Z   3

> SELECT * FROM "all_my_averages"

name: all_my_averages
---------------------
time                   mean
2015-08-18T00:00:00Z   8.0625
2015-08-18T00:12:00Z   7.8245
2015-08-18T00:24:00Z   7.5675
```

查询使用InfluxQL函数和`GROUP BY time()`子句聚合数据。它也将其结果写入`all_my_averages`measurement。

该查询是采样的示例：采用更高精度的数据，将这些数据聚合到较低的精度，并将较低精度数据存储在数据库中。 采样是`INTO`子句的常见用例。

#### 例五：将多个measurement的聚合结果写入到一个不同的数据库中(逆向引用采样)
```
> SELECT MEAN(*) INTO "where_else"."autogen".:MEASUREMENT FROM /.*/ WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:06:00Z' GROUP BY time(12m)

name: result
time                   written
----                   -------
1970-01-01T00:00:00Z   5

> SELECT * FROM "where_else"."autogen"./.*/

name: average_temperature
time                   mean_degrees   mean_index   mean_pH   mean_water_level
----                   ------------   ----------   -------   ----------------
2015-08-18T00:00:00Z   78.5

name: h2o_feet
time                   mean_degrees   mean_index   mean_pH   mean_water_level
----                   ------------   ----------   -------   ----------------
2015-08-18T00:00:00Z                                         5.07625

name: h2o_pH
time                   mean_degrees   mean_index   mean_pH   mean_water_level
----                   ------------   ----------   -------   ----------------
2015-08-18T00:00:00Z                               6.75

name: h2o_quality
time                   mean_degrees   mean_index   mean_pH   mean_water_level
----                   ------------   ----------   -------   ----------------
2015-08-18T00:00:00Z                  51.75

name: h2o_temperature
time                   mean_degrees   mean_index   mean_pH   mean_water_level
----                   ------------   ----------   -------   ----------------
2015-08-18T00:00:00Z   63.75
```

查询使用InfluxQL函数和`GROUP BY time()`子句聚合数据。它会在与`FROM`子句中的正则表达式匹配的每个measurement中聚合数据，并将结果写入`where_else`数据库和`autogen`保留策略中具有相同名称的measurement中。请注意，在运行`INTO`查询之前，`where_else`和`autogen`都必须存在。

该查询是使用反向引用进行下采样的示例。它从多个measurement中获取更高精度的数据，将这些数据聚合到较低的精度，并将较低精度数据存储在数据库中。使用反向引用进行下采样是`INTO`子句的常见用例。

### INTO子句的共同问题
#### 问题一：丢数据
如果`INTO`查询在`SELECT`子句中包含tag key，则查询将当前measurement中的tag转换为目标measurement中的字段。这可能会导致InfluxDB覆盖以前由tag value区分的点。请注意，此行为不适用于使用`TOP()`或`BOTTOM()`函数的查询。

要将当前measurement的tag保留在目标measurement中的tag中，`GROUP BY`相关tag key或`INTO`查询中的`GROUP BY *`。

#### 问题二：使用INTO子句自动查询
本文档中的`INTO`子句部分显示了如何使用`INTO`子句手动实现查询。 有关如何自动执行`INTO`子句查询实时数据，请参阅Continous Queries文档。除了其他用途之外，Continous Queries使采样过程自动化。

## ORDER BY TIME DESC
默认情况下，InfluxDB以升序的顺序返回结果; 返回的第一个点具有最早的时间戳，返回的最后一个点具有最新的时间戳。 `ORDER BY time DESC`反转该顺序，使得InfluxDB首先返回具有最新时间戳的点。

#### 语法

```
SELECT_clause [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] ORDER BY time DESC
```

#### 语法描述
如果查询包含`GROUP BY`子句,`ORDER by time DESC`必须出现在`GROUP BY`子句之后。如果查询包含一个`WHERE`子句并没有`GROUP BY`子句，`ORDER by time DESC`必须出现在`WHERE`子句之后。

#### 例子
##### 例一：首先返回最新的点
```
> SELECT "water_level" FROM "h2o_feet" WHERE "location" = 'santa_monica' ORDER BY time DESC

name: h2o_feet
time                   water_level
----                   -----------
2015-09-18T21:42:00Z   4.938
2015-09-18T21:36:00Z   5.066
[...]
2015-08-18T00:06:00Z   2.116
2015-08-18T00:00:00Z   2.064
```

该查询首先从`h2o_feet`measurement返回具有最新时间戳的点。没有`ORDER by time DESC`，查询将首先返回`2015-08-18T00：00：00Z`最后返回`2015-09-18T21：42：00Z`。

##### 例二：首先返回最新的点并包括GROUP BY time()子句

```
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:42:00Z' GROUP BY time(12m) ORDER BY time DESC

name: h2o_feet
time                   mean
----                   ----
2015-08-18T00:36:00Z   4.6825
2015-08-18T00:24:00Z   4.80675
2015-08-18T00:12:00Z   4.950749999999999
2015-08-18T00:00:00Z   5.07625
```

该查询在`GROUP BY`子句中使用InfluxQL函数和时间间隔来计算查询时间范围内每十二分钟间隔的平均`water_level`。`ORDER BY time DESC`返回最近12分钟的时间间隔。

##  LIMIT和SLIMIT子句
`LIMIT <N>`从指定的measurement中返回前`N`个数据点。
### 语法

```
SELECT_clause [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] LIMIT <N>
```

### 语法描述
`N`指定从指定measurement返回的点数。如果`N`大于measurement的点总数，InfluxDB返回该measurement中的所有点。请注意，`LIMIT`子句必须以上述语法中列出的顺序显示。

### 例子
#### 例一：限制返回的点数
```
> SELECT "water_level","location" FROM "h2o_feet" LIMIT 3

name: h2o_feet
time                   water_level   location
----                   -----------   --------
2015-08-18T00:00:00Z   8.12          coyote_creek
2015-08-18T00:00:00Z   2.064         santa_monica
2015-08-18T00:06:00Z   8.005         coyote_creek
```
这个查询从measurement`h2o_feet`中返回最旧的三个点。

#### 例二：限制返回的点数并包含一个GROUP BY子句
```
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:42:00Z' GROUP BY *,time(12m) LIMIT 2

name: h2o_feet
tags: location=coyote_creek
time                   mean
----                   ----
2015-08-18T00:00:00Z   8.0625
2015-08-18T00:12:00Z   7.8245

name: h2o_feet
tags: location=santa_monica
time                   mean
----                   ----
2015-08-18T00:00:00Z   2.09
2015-08-18T00:12:00Z   2.077
```

该查询使用InfluxQL函数和GROUP BY子句来计算每个tag以及查询时间内每隔十二分钟的间隔的平均`water_level`。 `LIMIT 2`请求两个最旧的十二分钟平均值。

请注意，没有`LIMIT 2`，查询将返回每个series四个点; 在查询的时间范围内每隔十二分钟的时间间隔一个点。

## SLIMIT子句



