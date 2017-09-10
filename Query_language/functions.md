# 函数

InfluxDB的函数可以分成Aggregate，select和predict类型。

## Aggregations
### COUNT()
返回非空字段值得数目
#### 语法
```
SELECT COUNT( [ * | <field_key> | /<regular_expression>/ ] ) [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```

#### 嵌套语法
```
SELECT COUNT(DISTINCT( [ * | <field_key> | /<regular_expression>/ ] )) [...]
```

#### 语法描述
`COUNT(field_key)`

返回field key对应的field values的数目。

`COUNT(/regular_expression/)`

返回匹配正则表达式的field key对应的field values的数目。

`COUNT(*)`

返回measurement中的每个field key对应的field value的数目。

`COUNT()`支持所有数据类型的field value，InfluxQL支持`COUNT()`嵌套`DISTINCT()`。

#### 例子
##### 例一：计数指定field key的field value的数目
```
> SELECT COUNT("water_level") FROM "h2o_feet"

name: h2o_feet
time                   count
----                   -----
1970-01-01T00:00:00Z   15258
```

该查询返回measurement`h2o_feet`中的`water_level`的非空字段值的数量。

##### 例二：计数measurement中每个field key关联的field value的数量

```
> SELECT COUNT(*) FROM "h2o_feet"

name: h2o_feet
time                   count_level description   count_water_level
----                   -----------------------   -----------------
1970-01-01T00:00:00Z   15258                     15258
```

该查询返回与measurement`h2o_feet`相关联的每个字段键的非空字段值的数量。`h2o_feet`有两个字段键：`level_description`和`water_level`。

##### 例三：计数匹配一个正则表达式的每个field key关联的field value的数目
```
> SELECT COUNT(/water/) FROM "h2o_feet"

name: h2o_feet
time                   count_water_level
----                   -----------------
1970-01-01T00:00:00Z   15258
```

该查询返回measurement`h2o_feet`中包含`water`单词的每个field key的非空字段值的数量。

##### 例四：计数包括多个子句的field key的field value的数目
```
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(200) LIMIT 7 SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   count
----                   -----
2015-08-17T23:48:00Z   200
2015-08-18T00:00:00Z   2
2015-08-18T00:12:00Z   2
2015-08-18T00:24:00Z   2
2015-08-18T00:36:00Z   2
2015-08-18T00:48:00Z   2
```

该查询返回`water_level`字段键中的非空字段值的数量。它涵盖`2015-08-17T23：48：00Z`和`2015-08-18T00：54：00Z`之间的时间段，并将结果分组为12分钟的时间间隔和每个tag。并用`200`填充空的时间间隔，并将点数返回7measurement返回1。

##### 例五：计数一个field key的distinct的field value的数量
```
> SELECT COUNT(DISTINCT("level description")) FROM "h2o_feet"

name: h2o_feet
time                   count
----                   -----
1970-01-01T00:00:00Z   4
```

查询返回measurement为`h2o_feet`field key为`level description`的唯一field value的数量。

#### COUNT()的常见问题
##### 问题一：COUNT()和fill()
大多数InfluxQL函数对于没有数据的时间间隔返回`null`值，`fill(<fill_option>)`将该`null`值替换为`fill_option`。 `COUNT()`针对没有数据的时间间隔返回`0`，`fill(<fill_option>)`用`fill_option`替换0值。

例如

下面的代码块中的第一个查询不包括`fill()`。最后一个时间间隔没有数据，因此该时间间隔的值返回为零。第二个查询包括`fill(800000)`; 它将最后一个间隔中的零替换为800000。

```
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE time >= '2015-09-18T21:24:00Z' AND time <= '2015-09-18T21:54:00Z' GROUP BY time(12m)

name: h2o_feet
time                   count
----                   -----
2015-09-18T21:24:00Z   2
2015-09-18T21:36:00Z   2
2015-09-18T21:48:00Z   0

> SELECT COUNT("water_level") FROM "h2o_feet" WHERE time >= '2015-09-18T21:24:00Z' AND time <= '2015-09-18T21:54:00Z' GROUP BY time(12m) fill(800000)

name: h2o_feet
time                   count
----                   -----
2015-09-18T21:24:00Z   2
2015-09-18T21:36:00Z   2
2015-09-18T21:48:00Z   800000
```

### DISTINCT()
返回field value的不同值列表。

#### 语法
```
SELECT DISTINCT( [ * | <field_key> | /<regular_expression>/ ] ) FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```

#### 嵌套语法
```
SELECT COUNT(DISTINCT( [ * | <field_key> | /<regular_expression>/ ] )) [...]
```

#### 语法描述
`DISTINCT(field_key)`

返回field key对应的不同field values。

`DISTINCT(/regular_expression/)`

返回匹配正则表达式的field key对应的不同field values。

`DISTINCT(*)`

返回measurement中的每个field key对应的不同field value。

`DISTINCT()`支持所有数据类型的field value，InfluxQL支持`COUNT()`嵌套`DISTINCT()`。

#### 例子
##### 例一：列出一个field key的不同的field value

```
> SELECT DISTINCT("level description") FROM "h2o_feet"

name: h2o_feet
time                   distinct
----                   --------
1970-01-01T00:00:00Z   between 6 and 9 feet
1970-01-01T00:00:00Z   below 3 feet
1970-01-01T00:00:00Z   between 3 and 6 feet
1970-01-01T00:00:00Z   at or greater than 9 feet
```

查询返回`level description`的所有的不同的值。

##### 例二：列出一个measurement中每个field key的不同值
```
> SELECT DISTINCT(*) FROM "h2o_feet"

name: h2o_feet
time                   distinct_level description   distinct_water_level
----                   --------------------------   --------------------
1970-01-01T00:00:00Z   between 6 and 9 feet         8.12
1970-01-01T00:00:00Z   between 3 and 6 feet         8.005
1970-01-01T00:00:00Z   at or greater than 9 feet    7.887
1970-01-01T00:00:00Z   below 3 feet                 7.762
[...]
```

查询返回`h2o_feet`中每个字段的唯一字段值的列表。`h2o_feet`有两个字段：`description`和`water_level`。

##### 例三：列出匹配正则表达式的field的不同field value
```
> SELECT DISTINCT(/description/) FROM "h2o_feet"

name: h2o_feet
time                   distinct_level description
----                   --------------------------
1970-01-01T00:00:00Z   below 3 feet
1970-01-01T00:00:00Z   between 6 and 9 feet
1970-01-01T00:00:00Z   between 3 and 6 feet
1970-01-01T00:00:00Z   at or greater than 9 feet
```

查询返回`h2o_feet`中含有`description`的字段的唯一字段值的列表。

##### 例四：列出包含多个子句的field key关联的不同值得列表
```
>  SELECT DISTINCT("level description") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   distinct
----                   --------
2015-08-18T00:00:00Z   between 6 and 9 feet
2015-08-18T00:12:00Z   between 6 and 9 feet
2015-08-18T00:24:00Z   between 6 and 9 feet
2015-08-18T00:36:00Z   between 6 and 9 feet
2015-08-18T00:48:00Z   between 6 and 9 feet
```

该查询返回`level description`字段键中不同字段值的列表。它涵盖`2015-08-17T23：48：00Z`和`2015-08-18T00：54：00Z`之间的时间段，并将结果按12分钟的时间间隔和每个tag分组。查询限制返回一个series。

##### 例五：对一个字段的不同值作计数
```
> SELECT COUNT(DISTINCT("level description")) FROM "h2o_feet"

name: h2o_feet
time                   count
----                   -----
1970-01-01T00:00:00Z   4
```

查询返回`h2o_feet`这个measurement中字段`level description`的不同值的数目。

#### DISTINCT()的常见问题
##### 问题一：DISTINCT()和INTO子句
使用`DISTINCT()`与`INTO`子句可能导致InfluxDB覆盖目标measurement中的点。`DISTINCT()`通常返回多个具有相同时间戳的结果; InfluxDB假设具有相同series的点，时间戳是重复的点，并且仅覆盖目的measurement中最近一个点的任何重复点。

例如

下面的代码中的第一个查询使用`DISTINCT()`函数，返回四个结果。请注意，每个结果具有相同的时间戳。第二个查询将`INTO`子句添加到初始查询中，并将查询结果写入measurement`distincts`中。代码中的最后一个查询选择`distincts`中的所有数据。最后一个查询返回一个点，因为四个初始结果是重复点; 它们属于同一series，具有相同的时间戳。 当系统遇到重复点时，它会用最近一个点覆盖上一个点。

```
>  SELECT DISTINCT("level description") FROM "h2o_feet"

name: h2o_feet
time                   distinct
----                   --------
1970-01-01T00:00:00Z   below 3 feet
1970-01-01T00:00:00Z   between 6 and 9 feet
1970-01-01T00:00:00Z   between 3 and 6 feet
1970-01-01T00:00:00Z   at or greater than 9 feet

>  SELECT DISTINCT("level description") INTO "distincts" FROM "h2o_feet"

name: result
time                   written
----                   -------
1970-01-01T00:00:00Z   4

> SELECT * FROM "distincts"

name: distincts
time                   distinct
----                   --------
1970-01-01T00:00:00Z   at or greater than 9 feet
```

### INTEGRAL()
返回字段曲线下的面积，即是积分。
#### 语法
```
SELECT INTEGRAL( [ * | <field_key> | /<regular_expression>/ ] [ , <unit> ]  ) [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```

#### 语法描述
InfluxDB计算字段曲线下的面积，并将这些结果转换为每`unit`的总和面积。`unit`参数是一个整数，后跟一个时间字符串，它是可选的。如果查询未指定单位，则单位默认为1秒（`1s`）。

`INTEGRAL(field_key)`

返回field key关联的值之下的面积。

`INTEGRAL(/regular_expression/)`

返回满足正则表达式的每个field key关联的值之下的面积。

`INTEGRAL(*)`

返回measurement中每个field key关联的值之下的面积。

`INTEGRAL()`不支持`fill()`，`INTEGRAL()`支持int64和float64两个数据类型。

#### 例子
下面的五个例子，使用数据库`NOAA_water_database`中的数据：

```
> SELECT "water_level" FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z'

name: h2o_feet
time                   water_level
----                   -----------
2015-08-18T00:00:00Z   2.064
2015-08-18T00:06:00Z   2.116
2015-08-18T00:12:00Z   2.028
2015-08-18T00:18:00Z   2.126
2015-08-18T00:24:00Z   2.041
2015-08-18T00:30:00Z   2.051
```

##### 例一：计算指定的field key的值得积分
```
> SELECT INTEGRAL("water_level") FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z'

name: h2o_feet
time                 integral
----                 --------
1970-01-01T00:00:00Z 3732.66
```

该查询返回`h2o_feet`中的字段`water_level`的曲线下的面积（以秒为单位）。

##### 例二：计算指定的field key和时间单位的值得积分
```
> SELECT INTEGRAL("water_level",1m) FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z'

name: h2o_feet
time                 integral
----                 --------
1970-01-01T00:00:00Z 62.211
```

该查询返回`h2o_feet`中的字段`water_level`的曲线下的面积（以分钟为单位）。

##### 例三：计算measurement中每个field key在指定时间单位的值得积分
```
> SELECT INTEGRAL(*,1m) FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z'

name: h2o_feet
time                 integral_water_level
----                 --------------------
1970-01-01T00:00:00Z 62.211
```

查询返回measurement`h2o_feet`中存储的每个数值字段相关的字段值的曲线下面积（以分钟为单位）。 `h2o_feet`的数值字段为`water_level`。

##### 例四：计算measurement中匹配正则表达式的field key在指定时间单位的值得积分
```
> SELECT INTEGRAL(/water/,1m) FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z'

name: h2o_feet
time                 integral_water_level
----                 --------------------
1970-01-0
```

查询返回field key包括单词`water`的每个数值类型的字段相关联的字段值的曲线下的区域（以分钟为单位）。

##### 例五：在含有多个子句中计算指定字段的积分
```
> SELECT INTEGRAL("water_level",1m) FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m) LIMIT 1

name: h2o_feet
time                 integral
----                 --------
2015-08-18T00:00:00Z 24.972
```

查询返回与字段`water_level`相关联的字段值的曲线下面积（以分钟为单位）。 它涵盖`2015-08-18T00：00：00Z`和`2015-08-18T00：30：00Z`之间的时间段，分组结果间隔12分钟，并将结果数量限制为1。

### MEAN()
返回字段的平均值
#### 语法
```
SELECT MEAN( [ * | <field_key> | /<regular_expression>/ ] ) [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```
#### 语法描述

`MEAN(field_key)`

返回field key关联的值的平均值。

`MEAN(/regular_expression/)`

返回满足正则表达式的每个field key关联的值的平均值。

`MEAN(*)`

返回measurement中每个field key关联的值的平均值。

`MEAN()`支持int64和float64两个数据类型。

#### 例子
##### 例一：计算指定字段的平均值
```
> SELECT MEAN("water_level") FROM "h2o_feet"

name: h2o_feet
time                   mean
----                   ----
1970-01-01T00:00:00Z   4.442107025822522
```

该查询返回measurement`h2o_feet`的字段`water_level`的平均值。

##### 例二：计算measurement中每个字段的平均值
```
> SELECT MEAN(*) FROM "h2o_feet"

name: h2o_feet
time                   mean_water_level
----                   ----------------
1970-01-01T00:00:00Z   4.442107025822522
```

查询返回在`h2o_feet`中数值类型的每个字段的平均值。`h2o_feet`有一个数值字段：`water_level`。

##### 例三：计算满足正则表达式的字段的平均值
```
> SELECT MEAN(/water/) FROM "h2o_feet"

name: h2o_feet
time                   mean_water_level
----                   ----------------
1970-01-01T00:00:00Z   4.442107025822523
```

查询返回在`h2o_feet`中字段中含有`water`的数值类型字段的平均值。

##### 例四：计算含有多个子句字段的平均值
```
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(9.01) LIMIT 7 SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   mean
----                   ----
2015-08-17T23:48:00Z   9.01
2015-08-18T00:00:00Z   8.0625
2015-08-18T00:12:00Z   7.8245
2015-08-18T00:24:00Z   7.5675
2015-08-18T00:36:00Z   7.303
2015-08-18T00:48:00Z   7.046
```

查询返回字段`water_level`中的值的平均值。它涵盖`2015-08-17T23：48：00Z`和`2015-08-18T00：54：00Z`之间的时间段，并将结果按12分钟的时间间隔和每个tag分组。该查询用`9.01`填充空时间间隔，并将点数和series分别限制到7和1。

### MEDIAN()