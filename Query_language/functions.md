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
返回排好序的字段的中位数。
#### 语法
```
SELECT MEDIAN( [ * | <field_key> | /<regular_expression>/ ] ) [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```
#### 语法描述

`MEDIAN(field_key)`

返回field key关联的值的中位数。

`MEDIAN(/regular_expression/)`

返回满足正则表达式的每个field key关联的值的中位数。

`MEDIAN(*)`

返回measurement中每个field key关联的值的中位数。

`MEDIAN()`支持int64和float64两个数据类型。

>注意：`MEDIAN()`近似于`PERCENTILE（field_key，50）`，除了如果该字段包含偶数个值，`MEDIAN()`返回两个中间字段值的平均值之外。


#### 例子
##### 例一：计算指定字段的中位数
```
> SELECT MEDIAN("water_level") FROM "h2o_feet"

name: h2o_feet
time                   median
----                   ------
1970-01-01T00:00:00Z   4.124
```

该查询返回measurement`h2o_feet`的字段`water_level`的中位数。

##### 例二：计算measurement中每个字段的中位数
```
> SELECT MEDIAN(*) FROM "h2o_feet"

name: h2o_feet
time                   median_water_level
----                   ------------------
1970-01-01T00:00:00Z   4.124
```

查询返回在`h2o_feet`中数值类型的每个字段的中位数。`h2o_feet`有一个数值字段：`water_level`。

##### 例三：计算满足正则表达式的字段的中位数
```
> SELECT MEDIAN(/water/) FROM "h2o_feet"

name: h2o_feet
time                   median_water_level
----                   ------------------
1970-01-01T00:00:00Z   4.124
```

查询返回在`h2o_feet`中字段中含有`water`的数值类型字段的中位数。

##### 例四：计算含有多个子句字段的中位数
```
> SELECT MEDIAN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(700) LIMIT 7 SLIMIT 1 SOFFSET 1

name: h2o_feet
tags: location=santa_monica
time                   median
----                   ------
2015-08-17T23:48:00Z   700
2015-08-18T00:00:00Z   2.09
2015-08-18T00:12:00Z   2.077
2015-08-18T00:24:00Z   2.0460000000000003
2015-08-18T00:36:00Z   2.0620000000000003
2015-08-18T00:48:00Z   700
```

查询返回字段`water_level`中的值的中位数。它涵盖`2015-08-17T23：48：00Z`和`2015-08-18T00：54：00Z`之间的时间段，并将结果按12分钟的时间间隔和每个tag分组。该查询用`700`填充空时间间隔，并将点数和series分别限制到7和1，并将series的返回偏移1。

### MODE()
返回字段中出现频率最高的值。
#### 语法
```
SELECT MODE( [ * | <field_key> | /<regular_expression>/ ] ) [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```
#### 语法描述

`MODE(field_key)`

返回field key关联的值的出现频率最高的值。

`MODE(/regular_expression/)`

返回满足正则表达式的每个field key关联的值的出现频率最高的值。

`MODE(*)`

返回measurement中每个field key关联的值的出现频率最高的值。

`MODE()`支持所有数据类型。

>注意：`MODE()`如果最多出现次数有两个或多个值，则返回具有最早时间戳的字段值。

#### 例子
##### 例一：计算指定字段的最常出现的值
```
> SELECT MODE("level description") FROM "h2o_feet"

name: h2o_feet
time                   mode
----                   ----
1970-01-01T00:00:00Z   between 3 and 6 feet
```

该查询返回measurement`h2o_feet`的字段`level description`的最常出现的值。

##### 例二：计算measurement中每个字段最常出现的值
```
> SELECT MODE(*) FROM "h2o_feet"

name: h2o_feet
time                   mode_level description   mode_water_level
----                   ----------------------   ----------------
1970-01-01T00:00:00Z   between 3 and 6 feet     2.69
```

查询返回在`h2o_feet`中数值类型的每个字段的最常出现的值。`h2o_feet`有两个字段：`water_level`和`level description`。

##### 例三：计算满足正则表达式的字段的最常出现的值
```
> SELECT MODE(/water/) FROM "h2o_feet"

name: h2o_feet
time                   mode_water_level
----                   ----------------
1970-01-01T00:00:00Z   2.69
```

查询返回在`h2o_feet`中字段中含有`water`的字段的最常出现的值。

##### 例四：计算含有多个子句字段的最常出现的值
```
> SELECT MODE("level description") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* LIMIT 3 SLIMIT 1 SOFFSET 1

name: h2o_feet
tags: location=santa_monica
time                   mode
----                   ----
2015-08-17T23:48:00Z
2015-08-18T00:00:00Z   below 3 feet
2015-08-18T00:12:00Z   below 3 feet
```

查询返回字段`water_level`中的值的最常出现的值。它涵盖`2015-08-17T23：48：00Z`和`2015-08-18T00：54：00Z`之间的时间段，并将结果按12分钟的时间间隔和每个tag分组。，并将点数和series分别限制到3和1，并将series的返回偏移1。

### SPREAD()
返回字段中最大和最小值的差值。
#### 语法
```
SELECT SPREAD( [ * | <field_key> | /<regular_expression>/ ] ) [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```
#### 语法描述

`SPREAD(field_key)`

返回field key最大和最小值的差值。

`SPREAD(/regular_expression/)`

返回满足正则表达式的每个field key最大和最小值的差值。

`SPREAD(*)`

返回measurement中每个field key最大和最小值的差值。

`SPREAD()`支持所有的数值类型的field。

#### 例子
##### 例一：计算指定字段最大和最小值的差值
```
> SELECT SPREAD("water_level") FROM "h2o_feet"

name: h2o_feet
time                   spread
----                   ------
1970-01-01T00:00:00Z   10.574
```

该查询返回measurement`h2o_feet`的字段`water_level`的最大和最小值的差值。

##### 例二：计算measurement中每个字段最大和最小值的差值
```
> SELECT SPREAD(*) FROM "h2o_feet"

name: h2o_feet
time                   spread_water_level
----                   ------------------
1970-01-01T00:00:00Z   10.574
```

查询返回在`h2o_feet`中数值类型的每个数值字段的最大和最小值的差值。`h2o_feet`有一个数值字段：`water_level`。

##### 例三：计算满足正则表达式的字段最大和最小值的差值
```
> SELECT SPREAD(/water/) FROM "h2o_feet"

name: h2o_feet
time                   spread_water_level
----                   ------------------
1970-01-01T00:00:00Z   10.574
```

查询返回在`h2o_feet`中字段中含有`water`的所有数值字段的最大和最小值的差值。

##### 例四：计算含有多个子句字段最大和最小值的差值
```
> SELECT SPREAD("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(18) LIMIT 3 SLIMIT 1 SOFFSET 1

name: h2o_feet
tags: location=santa_monica
time                   spread
----                   ------
2015-08-17T23:48:00Z   18
2015-08-18T00:00:00Z   0.052000000000000046
2015-08-18T00:12:00Z   0.09799999999999986
```

查询返回字段`water_level`中的最大和最小值的差值。它涵盖`2015-08-17T23：48：00Z`和`2015-08-18T00：54：00Z`之间的时间段，并将结果按12分钟的时间间隔和每个tag分组，空值用18来填充，并将点数和series分别限制到3和1，并将series的返回偏移1。

### STDDEV()
返回字段的标准差。
#### 语法
```
SELECT STDDEV( [ * | <field_key> | /<regular_expression>/ ] ) [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```
#### 语法描述

`STDDEV(field_key)`

返回field key的标准差。

`STDDEV(/regular_expression/)`

返回满足正则表达式的每个field key的标准差。

`STDDEV(*)`

返回measurement中每个field key的标准差。

`STDDEV()`支持所有的数值类型的field。

#### 例子
##### 例一：计算指定字段的标准差
```
> SELECT STDDEV("water_level") FROM "h2o_feet"

name: h2o_feet
time                   stddev
----                   ------
1970-01-01T00:00:00Z   2.279144584196141
```

该查询返回measurement`h2o_feet`的字段`water_level`的标准差。

##### 例二：计算measurement中每个字段的标准差
```
> SELECT STDDEV(*) FROM "h2o_feet"

name: h2o_feet
time                   stddev_water_level
----                   ------------------
1970-01-01T00:00:00Z   2.279144584196141
```

查询返回在`h2o_feet`中数值类型的每个数值字段的标准差。`h2o_feet`有一个数值字段：`water_level`。

##### 例三：计算满足正则表达式的字段的标准差
```
> SELECT STDDEV(/water/) FROM "h2o_feet"

name: h2o_feet
time                   stddev_water_level
----                   ------------------
1970-01-01T00:00:00Z   2.279144584196141
```

查询返回在`h2o_feet`中字段中含有`water`的所有数值字段的标准差。

##### 例四：计算含有多个子句字段的标准差
```
> SELECT STDDEV("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(18000) LIMIT 2 SLIMIT 1 SOFFSET 1

name: h2o_feet
tags: location=santa_monica
time                   stddev
----                   ------
2015-08-17T23:48:00Z   18000
2015-08-18T00:00:00Z   0.03676955262170051
```

查询返回字段`water_level`的标准差。它涵盖`2015-08-17T23：48：00Z`和`2015-08-18T00：54：00Z`之间的时间段，并将结果按12分钟的时间间隔和每个tag分组，空值用18000来填充，并将点数和series分别限制到2和1，并将series的返回偏移1。

### SUM()
返回字段值的和。
#### 语法
```
SELECT SUM( [ * | <field_key> | /<regular_expression>/ ] ) [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```
#### 语法描述

`SUM(field_key)`

返回field key的值的和。

`SUM(/regular_expression/)`

返回满足正则表达式的每个field key的值的和。

`SUM(*)`

返回measurement中每个field key的值的和。

`SUM()`支持所有的数值类型的field。

#### 例子
##### 例一：计算指定字段的值的和
```
> SELECT SUM("water_level") FROM "h2o_feet"

name: h2o_feet
time                   sum
----                   ---
1970-01-01T00:00:00Z   67777.66900000004
```

该查询返回measurement`h2o_feet`的字段`water_level`的值的和。

##### 例二：计算measurement中每个字段的值的和
```
> SELECT SUM(*) FROM "h2o_feet"

name: h2o_feet
time                   sum_water_level
----                   ---------------
1970-01-01T00:00:00Z   67777.66900000004
```

查询返回在`h2o_feet`中数值类型的每个数值字段的值的和。`h2o_feet`有一个数值字段：`water_level`。

##### 例三：计算满足正则表达式的字段的值的和
```
> SELECT SUM(/water/) FROM "h2o_feet"

name: h2o_feet
time                   sum_water_level
----                   ---------------
1970-01-01T00:00:00Z   67777.66900000004
```

查询返回在`h2o_feet`中字段中含有`water`的所有数值字段的值的和。

##### 例四：计算含有多个子句字段的值的和
```
> SELECT SUM("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(18000) LIMIT 4 SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   sum
----                   ---
2015-08-17T23:48:00Z   18000
2015-08-18T00:00:00Z   16.125
2015-08-18T00:12:00Z   15.649
2015-08-18T00:24:00Z   15.135
```

查询返回字段`water_level`的值的和。它涵盖`2015-08-17T23：48：00Z`和`2015-08-18T00：54：00Z`之间的时间段，并将结果按12分钟的时间间隔和每个tag分组，空值用18000来填充，并将点数和series分别限制到2和1，并将series的返回偏移1。

## Selectors
### BOTTOM()
返回最小的N个field值。
#### 语法
```
SELECT BOTTOM(<field_key>[,<tag_key(s)>],<N> )[,<tag_key(s)>|<field_key(s)>] [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```
#### 语法描述
`BOTTOM(field_key,N)`

返回field key的最小的N个field value。

`BOTTOM(field_key,tag_key(s),N)`

返回某个tag key的N个tag value的最小的field value。

`BOTTOM(field_key,N),tag_key(s),field_key(s)`

返回括号里的字段的最小N个field value，以及相关的tag或field，或者两者都有。

`BOTTOM()`支持所有的数值类型的field。

>说明： 
> 
> * 如果一个field有两个或多个相等的field value，`BOTTOM()`返回时间戳最早的那个。
> * `BOTTOM()`和`INTO`子句一起使用的时候，和其他的函数有些不一样。

#### 例子
##### 例一：选择一个field的最小的三个值
```
> SELECT BOTTOM("water_level",3) FROM "h2o_feet"

name: h2o_feet
time                   bottom
----                   ------
2015-08-29T14:30:00Z   -0.61
2015-08-29T14:36:00Z   -0.591
2015-08-30T15:18:00Z   -0.594
```

该查询返回measurement`h2o_feet`的字段`water_level`的最小的三个值。

##### 例二：选择一个field的两个tag的分别最小的值
```
> SELECT BOTTOM("water_level","location",2) FROM "h2o_feet"

name: h2o_feet
time                   bottom   location
----                   ------   --------
2015-08-29T10:36:00Z   -0.243   santa_monica
2015-08-29T14:30:00Z   -0.61    coyote_creek
```

该查询返回和tag`location`相关的两个tag值的字段`water_level`的分别最小值。

##### 例三：选择一个field的最小的四个值，以及其关联的tag和field
```
> SELECT BOTTOM("water_level",4),"location","level description" FROM "h2o_feet"

name: h2o_feet
time                  bottom  location      level description
----                  ------  --------      -----------------
2015-08-29T14:24:00Z  -0.587  coyote_creek  below 3 feet
2015-08-29T14:30:00Z  -0.61   coyote_creek  below 3 feet
2015-08-29T14:36:00Z  -0.591  coyote_creek  below 3 feet
2015-08-30T15:18:00Z  -0.594  coyote_creek  below 3 feet
```

查询返回`water_level`中最小的四个字段值以及tag`location`和field`level description`的相关值。

##### 例四：选择一个field的最小的三个值，并且包括了多个子句
```
> SELECT BOTTOM("water_level",3),"location" FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(24m) ORDER BY time DESC

name: h2o_feet
time                  bottom  location
----                  ------  --------
2015-08-18T00:48:00Z  1.991   santa_monica
2015-08-18T00:54:00Z  2.054   santa_monica
2015-08-18T00:54:00Z  6.982   coyote_creek
2015-08-18T00:24:00Z  2.041   santa_monica
2015-08-18T00:30:00Z  2.051   santa_monica
2015-08-18T00:42:00Z  2.057   santa_monica
2015-08-18T00:00:00Z  2.064   santa_monica
2015-08-18T00:06:00Z  2.116   santa_monica
2015-08-18T00:12:00Z  2.028   santa_monica
```

查询将返回在`2015-08-18T00：00：00Z`和`2015-08-18T00：54：00Z`之间的每24分钟间隔内，`water_level`最小的三个值。它还以降序的时间戳顺序返回结果。 

请注意，`GROUP BY time()`子句不会覆盖点的原始时间戳。有关该行为的更详细解释，请参阅下面的问题一。

#### `BOTTOM()`的常见问题
##### 问题一：`BOTTOM()`和`GROUP BY time()`子句
`BOTTOM()`和`GROUP BY time()`子句的查询返回每个`GROUP BY time()`间隔指定的点数。对于大多数`GROUP BY time()`查询，返回的时间戳记标记`GROUP BY time()`间隔的开始。`GROUP BY time()`查询与`BOTTOM()`函数的行为不同; 它们保留原始数据点的时间戳。

例如

下面的查询返回每18分钟·`GROUP BY time()`间隔的两点。请注意，返回的时间戳是点的原始时间戳; 它们不会被强制匹配`GROUP BY time()`间隔的开始。

```
> SELECT BOTTOM("water_level",2) FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' AND "location" = 'santa_monica' GROUP BY time(18m)

name: h2o_feet
time                   bottom
----                   ------
                           __
2015-08-18T00:00:00Z  2.064 |
2015-08-18T00:12:00Z  2.028 | <------- Smallest points for the first time interval
                           --
                           __
2015-08-18T00:24:00Z  2.041 |
2015-08-18T00:30:00Z  2.051 | <------- Smallest points for the second time interval
                           --
```

##### 问题二：`BOTTOM()`和一个少于N个值得tag key
使用语法`SELECT BOTTOM（<field_key>，<tag_key>，<N>）`的查询可以返回比预期少的点。如果tag具有X标签值，则查询指定N个值，当X小于N，则查询返回X点。

例如

下面的查询将要求tag`location`的三个值的`water_level`的最小字段值。由于`location`具有两个值（`santa_monica`和`coyote_creek`），所以查询返回两点而不是三个。

```
> SELECT BOTTOM("water_level","location",3) FROM "h2o_feet"

name: h2o_feet
time                   bottom   location
----                   ------   --------
2015-08-29T10:36:00Z   -0.243   santa_monica
2015-08-29T14:30:00Z   -0.61    coyote_creek
```

##### 问题三：`BOTTOM()`，tags和`INTO`子句
当与`INTO`子句和`GROUP BY tag`子句结合使用时，大多数InfluxQL函数将初始数据中的任何tag转换为新写入的数据中的field。此行为也适用于`BOTTOM()`函数，除非`BOTTOM()`包含一个tag key作为参数：`BOTTOM(field_key，tag_key(s)，N)`。在这些情况下，系统将指定的tag作为新写入的数据中的tag。

例如

下面的代码块中的第一个查询返回与tag `location`相关联的两个tag value的field`water_level`中最小的字段值。它也将这些结果写入measurement`bottom_water_levels`。 第二个查询显示InfluxDB在`bottom_water_levels`中将`location`保存为tag。

```
> SELECT BOTTOM("water_level","location",2) INTO "bottom_water_levels" FROM "h2o_feet"

name: result
time                 written
----                 -------
1970-01-01T00:00:00Z 2

> SHOW TAG KEYS FROM "bottom_water_levels"

name: bottom_water_levels
tagKey
------
location
```

### FIRST()
返回时间戳最早的值
#### 语法
```
SELECT FIRST(<field_key>)[,<tag_key(s)>|<field_key(s)>] [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```
#### 语法描述
`FIRST(field_key)`

返回field key时间戳最早的值。

`FIRST(/regular_expression/)`

返回满足正则表达式的每个field key的时间戳最早的值。

`FIRST(*)`

返回measurement中每个field key的时间戳最早的值。

`FIRST(field_key),tag_key(s),field_key(s)`

返回括号里的字段的时间戳最早的值，以及相关联的tag或field，或者两者都有。

`FIRST()`支持所有类型的field。

#### 例子
##### 例一：返回field key时间戳最早的值
```
> SELECT FIRST("level description") FROM "h2o_feet"

name: h2o_feet
time                   first
----                   -----
2015-08-18T00:00:00Z   between 6 and 9 feet
```

查询返回`level description`的时间戳最早的值。

##### 例二：列出一个measurement中每个field key的时间戳最早的值
```
> SELECT FIRST(*) FROM "h2o_feet"

name: h2o_feet
time                   first_level description   first_water_level
----                   -----------------------   -----------------
1970-01-01T00:00:00Z   between 6 and 9 feet      8.12
```

查询返回`h2o_feet`中每个字段的时间戳最早的值。`h2o_feet`有两个字段：`level description`和`water_level`。

##### 例三：列出匹配正则表达式的field的时间戳最早的值
```
> SELECT FIRST(/level/) FROM "h2o_feet"

name: h2o_feet
time                   first_level description   first_water_level
----                   -----------------------   -----------------
1970-01-01T00:00:00Z   between 6 and 9 feet      8.12
```

查询返回`h2o_feet`中含有`level`的字段的时间戳最早的值。

##### 例四：返回field的最早的值，以及其相关的tag和field
```
> SELECT FIRST("level description"),"location","water_level" FROM "h2o_feet"

name: h2o_feet
time                  first                 location      water_level
----                  -----                 --------      -----------
2015-08-18T00:00:00Z  between 6 and 9 feet  coyote_creek  8.12
```

查询返回`level description`的时间戳最早的值，以及其相关的tag`location`和field`water_level`。

##### 例五：列出包含多个子句的field key的时间戳最早的值
```
> SELECT FIRST("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(9.01) LIMIT 4 SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   first
----                   -----
2015-08-17T23:48:00Z   9.01
2015-08-18T00:00:00Z   8.12
2015-08-18T00:12:00Z   7.887
2015-08-18T00:24:00Z   7.635
```

查询返回字段`water_level`中最早的字段值。它涵盖`2015-08-17T23：48：00Z`和`2015-08-18T00：54：00Z`之间的时间段，并将结果按12分钟的时间间隔和每个tag分组。查询用`9.01`填充空时间间隔，并将点数和measurement限制到4和1。

请注意，`GROUP BY time()`子句覆盖点的原始时间戳。结果中的时间戳表示每12分钟时间间隔的开始; 结果的第一点涵盖`2015-08-17T23：48：00Z`和`2015-08-18T00：00：00Z`之间的时间间隔，结果的最后一点涵盖`2015-08-18T00:24:00Z`和`2015-08-18T00：36：00Z`之间的间隔。

### LAST()
返回时间戳最近的值
#### 语法
```
SELECT LAST(<field_key>)[,<tag_key(s)>|<field_keys(s)>] [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]

```
#### 语法描述
`LAST(field_key)`

返回field key时间戳最近的值。

`LAST(/regular_expression/)`

返回满足正则表达式的每个field key的时间戳最近的值。

`LAST(*)`

返回measurement中每个field key的时间戳最近的值。

`LAST(field_key),tag_key(s),field_key(s)`

返回括号里的字段的时间戳最近的值，以及相关联的tag或field，或者两者都有。

`LAST()`支持所有类型的field。

#### 例子
##### 例一：返回field key时间戳最近的值
```
> SELECT LAST("level description") FROM "h2o_feet"

name: h2o_feet
time                   last
----                   ----
2015-09-18T21:42:00Z   between 3 and 6 feet
```

查询返回`level description`的时间戳最近的值。

##### 例二：列出一个measurement中每个field key的时间戳最近的值
```
> SELECT LAST(*) FROM "h2o_feet"

name: h2o_feet
time                   first_level description   first_water_level
----                   -----------------------   -----------------
1970-01-01T00:00:00Z   between 3 and 6 feet      4.938
```

查询返回`h2o_feet`中每个字段的时间戳最近的值。`h2o_feet`有两个字段：`level description`和`water_level`。

##### 例三：列出匹配正则表达式的field的时间戳最近的值
```
> SELECT LAST(/level/) FROM "h2o_feet"

name: h2o_feet
time                   first_level description   first_water_level
----                   -----------------------   -----------------
1970-01-01T00:00:00Z   between 3 and 6 feet      4.938
```

查询返回`h2o_feet`中含有`level`的字段的时间戳最近的值。

##### 例四：返回field的最近的值，以及其相关的tag和field
```
> SELECT LAST("level description"),"location","water_level" FROM "h2o_feet"

name: h2o_feet
time                  last                  location      water_level
----                  ----                  --------      -----------
2015-09-18T21:42:00Z  between 3 and 6 feet  santa_monica  4.938
```

查询返回`level description`的时间戳最近的值，以及其相关的tag`location`和field`water_level`。

##### 例五：列出包含多个子句的field key的时间戳最近的值
```
> SELECT LAST("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(9.01) LIMIT 4 SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   last
----                   ----
2015-08-17T23:48:00Z   9.01
2015-08-18T00:00:00Z   8.005
2015-08-18T00:12:00Z   7.762
2015-08-18T00:24:00Z   7.5
```

查询返回字段`water_level`中最近的字段值。它涵盖`2015-08-17T23：48：00Z`和`2015-08-18T00：54：00Z`之间的时间段，并将结果按12分钟的时间间隔和每个tag分组。查询用`9.01`填充空时间间隔，并将点数和measurement限制到4和1。

请注意，`GROUP BY time()`子句覆盖点的原始时间戳。结果中的时间戳表示每12分钟时间间隔的开始; 结果的第一点涵盖`2015-08-17T23：48：00Z`和`2015-08-18T00：00：00Z`之间的时间间隔，结果的最后一点涵盖`2015-08-18T00:24:00Z`和`2015-08-18T00：36：00Z`之间的间隔。

### MAX()
返回最大的字段值
#### 语法
```
SELECT MAX(<field_key>)[,<tag_key(s)>|<field__key(s)>] [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```
#### 语法描述
`MAX(field_key)`

返回field key的最大值。

`MAX(/regular_expression/)`

返回满足正则表达式的每个field key的最大值。

`MAX(*)`

返回measurement中每个field key的最大值。

`MAX(field_key),tag_key(s),field_key(s)`

返回括号里的字段的最大值，以及相关联的tag或field，或者两者都有。

`MAX()`支持所有数值类型的field。

#### 例子
##### 例一：返回field key的最大值
```
> SELECT MAX("water_level") FROM "h2o_feet"

name: h2o_feet
time                   max
----                   ---
2015-08-29T07:24:00Z   9.964
```

查询返回`water_level`的最大值。

##### 例二：列出一个measurement中每个field key的最大值
```
> SELECT MAX(*) FROM "h2o_feet"

name: h2o_feet
time                   max_water_level
----                   ---------------
2015-08-29T07:24:00Z   9.964
```

查询返回`h2o_feet`中每个字段的最大值。`h2o_feet`有一个数值类型的字段：`water_level`。

##### 例三：列出匹配正则表达式的field的最大值
```
> SELECT MAX(/level/) FROM "h2o_feet"

name: h2o_feet
time                   max_water_level
----                   ---------------
2015-08-29T07:24:00Z   9.964
```

查询返回`h2o_feet`中含有`level`的数值字段的最大值。

##### 例四：返回field的最大值，以及其相关的tag和field
```
> SELECT MAX("water_level"),"location","level description" FROM "h2o_feet"

name: h2o_feet
time                  max    location      level description
----                  ---    --------      -----------------
2015-08-29T07:24:00Z  9.964  coyote_creek  at or greater than 9 feet
```

查询返回`water_level`的最大值，以及其相关的tag`location`和field`level description`。

##### 例五：列出包含多个子句的field key的最大值
```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(9.01) LIMIT 4 SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   max
----                   ---
2015-08-17T23:48:00Z   9.01
2015-08-18T00:00:00Z   8.12
2015-08-18T00:12:00Z   7.887
2015-08-18T00:24:00Z   7.635
```

查询返回字段`water_level`的最大值。它涵盖`2015-08-17T23：48：00Z`和`2015-08-18T00：54：00Z`之间的时间段，并将结果按12分钟的时间间隔和每个tag分组。查询用`9.01`填充空时间间隔，并将点数和measurement限制到4和1。

请注意，`GROUP BY time()`子句覆盖点的原始时间戳。结果中的时间戳表示每12分钟时间间隔的开始; 结果的第一点涵盖`2015-08-17T23：48：00Z`和`2015-08-18T00：00：00Z`之间的时间间隔，结果的最后一点涵盖`2015-08-18T00:24:00Z`和`2015-08-18T00：36：00Z`之间的间隔。

### MIN()
返回最小的字段值
#### 语法
```
SELECT MIN(<field_key>)[,<tag_key(s)>|<field__key(s)>] [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```
#### 语法描述
`MIN(field_key)`

返回field key的最小值。

`MIN(/regular_expression/)`

返回满足正则表达式的每个field key的最小值。

`MIN(*)`

返回measurement中每个field key的最小值。

`MIN(field_key),tag_key(s),field_key(s)`

返回括号里的字段的最小值，以及相关联的tag或field，或者两者都有。

`MIN()`支持所有数值类型的field。

#### 例子
##### 例一：返回field key的最小值
```
> SELECT MIN("water_level") FROM "h2o_feet"

name: h2o_feet
time                   min
----                   ---
2015-08-29T14:30:00Z   -0.61
```

查询返回`water_level`的最小值。

##### 例二：列出一个measurement中每个field key的最小值
```
> SELECT MIN(*) FROM "h2o_feet"

name: h2o_feet
time                   min_water_level
----                   ---------------
2015-08-29T14:30:00Z   -0.61
```

查询返回`h2o_feet`中每个字段的最小值。`h2o_feet`有一个数值类型的字段：`water_level`。

##### 例三：列出匹配正则表达式的field的最小值
```
> SELECT MIN(/level/) FROM "h2o_feet"

name: h2o_feet
time                   min_water_level
----                   ---------------
2015-08-29T14:30:00Z   -0.61
```

查询返回`h2o_feet`中含有`level`的数值字段的最小值。

##### 例四：返回field的最小值，以及其相关的tag和field
```
> SELECT MIN("water_level"),"location","level description" FROM "h2o_feet"

name: h2o_feet
time                  min    location      level description
----                  ---    --------      -----------------
2015-08-29T14:30:00Z  -0.61  coyote_creek  below 3 feet
```

查询返回`water_level`的最小值，以及其相关的tag`location`和field`level description`。

##### 例五：列出包含多个子句的field key的最小值
```
> SELECT MIN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(9.01) LIMIT 4 SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   min
----                   ---
2015-08-17T23:48:00Z   9.01
2015-08-18T00:00:00Z   8.005
2015-08-18T00:12:00Z   7.762
2015-08-18T00:24:00Z   7.5
```

查询返回字段`water_level`的最小值。它涵盖`2015-08-17T23：48：00Z`和`2015-08-18T00：54：00Z`之间的时间段，并将结果按12分钟的时间间隔和每个tag分组。查询用`9.01`填充空时间间隔，并将点数和measurement限制到4和1。

请注意，`GROUP BY time()`子句覆盖点的原始时间戳。结果中的时间戳表示每12分钟时间间隔的开始; 结果的第一点涵盖`2015-08-17T23：48：00Z`和`2015-08-18T00：00：00Z`之间的时间间隔，结果的最后一点涵盖`2015-08-18T00:24:00Z`和`2015-08-18T00：36：00Z`之间的间隔。

### PERCENTILE()
返回较大百分之N的字段值
#### 语法
```
SELECT PERCENTILE(<field_key>, <N>)[,<tag_key(s)>|<field_key(s)>] [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```
#### 语法描述
`PERCENTILE(field_key,N)`

返回field key较大的百分之N的值。

`PERCENTILE(/regular_expression/,N)`

返回满足正则表达式的每个field key较大的百分之N的值。

`PERCENTILE(*,N)`

返回measurement中每个field key较大的百分之N的值。

`PERCENTILE(field_key,N),tag_key(s),field_key(s)`

返回括号里的字段较大的百分之N的值，以及相关联的tag或field，或者两者都有。

`N`必须是0到100的整数或者浮点数。

`PERCENTILE()`支持所有数值类型的field。

#### 例子
##### 例一：返回field key较大的百分之5的值
```
> SELECT PERCENTILE("water_level",5) FROM "h2o_feet"

name: h2o_feet
time                   percentile
----                   ----------
2015-08-31T03:42:00Z   1.122
```

查询返回`water_level`中值在总的field value中比较大的百分之五。

##### 例二：列出一个measurement中每个field key较大的百分之5的值
```
> SELECT PERCENTILE(*,5) FROM "h2o_feet"

name: h2o_feet
time                   percentile_water_level
----                   ----------------------
2015-08-31T03:42:00Z   1.122
```

查询返回`h2o_feet`中每个字段中值在总的field value中比较大的百分之五。`h2o_feet`有一个数值类型的字段：`water_level`。

##### 例三：列出匹配正则表达式的field较大的百分之5的值
```
> SELECT PERCENTILE(/level/,5) FROM "h2o_feet"

name: h2o_feet
time                   percentile_water_level
----                   ----------------------
2015-08-31T03:42:00Z   1.122
```

查询返回`h2o_feet`中含有`water`的数值字段的较大的百分之5的值。

##### 例四：返回field较大的百分之5的值，以及其相关的tag和field
```
> SELECT PERCENTILE("water_level",5),"location","level description" FROM "h2o_feet"

name: h2o_feet
time                  percentile  location      level description
----                  ----------  --------      -----------------
2015-08-31T03:42:00Z  1.122       coyote_creek  below 3 feet
```

查询返回`water_level`的较大的百分之5的值，以及其相关的tag`location`和field`level description`。

##### 例五：列出包含多个子句的field key的较大的百分之20的值
```
> SELECT PERCENTILE("water_level",20) FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(24m) fill(15) LIMIT 2

name: h2o_feet
time                   percentile
----                   ----------
2015-08-17T23:36:00Z   15
2015-08-18T00:00:00Z   2.064
```

查询返回字段`water_level`较大的百分之20的值。它涵盖`2015-08-17T23：48：00Z`和`2015-08-18T00：54：00Z`之间的时间段，并将结果按24分钟的时间间隔分组。查询用`15`填充空时间间隔，并将点数限制到2。

请注意，`GROUP BY time()`子句覆盖点的原始时间戳。结果中的时间戳表示每24分钟时间间隔的开始; 结果的第一点涵盖`2015-08-17T23：36：00Z`和`2015-08-18T00：00：00Z`之间的时间间隔，结果的最后一点涵盖`2015-08-18T00:00:00Z`和`2015-08-18T00：24：00Z`之间的间隔。

#### PERCENTILE()的常见问题
##### 问题一：PERCENTILE()和其他函数的比较

* `PERCENTILE(<field_key>,100)`相当于`MAX(<field_key>)`。
* `PERCENTILE(<field_key>，50)`几乎等于`MEDIAN(<field_key>)`，除了如果字段键包含偶数个字段值,`MEDIAN()`函数返回两个中间值的平均值.
* `PERCENTILE(<field_key>,0)`相当于`MIN(<field_key>)`

### SAMPLE()
返回`N`个随机抽样的字段值。`SAMPLE()`使用[reservoir sampling](https://en.wikipedia.org/wiki/Reservoir_sampling)来生成随机点。

#### 语法
```
SELECT SAMPLE(<field_key>, <N>)[,<tag_key(s)>|<field_key(s)>] [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```

`SAMPLE(field_key,N)`

返回field key的N个随机抽样的字段值。

`SAMPLE(/regular_expression/,N)`

返回满足正则表达式的每个field key的N个随机抽样的字段值。

`SAMPLE(*,N)`

返回measurement中每个field key的N个随机抽样的字段值。

`SAMPLE(field_key,N),tag_key(s),field_key(s)`

返回括号里的字段的N个随机抽样的字段值，以及相关联的tag或field，或者两者都有。

`N`必须是整数。

`SAMPLE()`支持所有类型的field。

#### 例子
##### 例一：返回field key的两个随机抽样的字段值
```
> SELECT SAMPLE("water_level",2) FROM "h2o_feet"

name: h2o_feet
time                   sample
----                   ------
2015-09-09T21:48:00Z   5.659
2015-09-18T10:00:00Z   6.939
```

查询返回`water_level`的两个随机抽样的字段值。

##### 例二：列出一个measurement中每个field key的两个随机抽样的字段值
```
> SELECT SAMPLE(*,2) FROM "h2o_feet"

name: h2o_feet
time                   sample_level description   sample_water_level
----                   ------------------------   ------------------
2015-08-25T17:06:00Z                              3.284
2015-09-03T04:30:00Z   below 3 feet
2015-09-03T20:06:00Z   between 3 and 6 feet
2015-09-08T21:54:00Z                              3.412
```

查询返回`h2o_feet`中每个字段的两个随机抽样的字段值。`h2o_feet`有两个字段：`water_level`和`level description`。

##### 例三：列出匹配正则表达式的field的两个随机抽样的字段值
```
> SELECT SAMPLE(/level/,2) FROM "h2o_feet"

name: h2o_feet
time                   sample_level description   sample_water_level
----                   ------------------------   ------------------
2015-08-30T05:54:00Z   between 6 and 9 feet
2015-09-07T01:18:00Z                              7.854
2015-09-09T20:30:00Z                              7.32
2015-09-13T19:18:00Z   between 3 and 6 feet
```

查询返回`h2o_feet`中含有`level`的字段的两个随机抽样的字段值。

##### 例四：返回field两个随机抽样的字段值，以及其相关的tag和field
```
> SELECT SAMPLE("water_level",2),"location","level description" FROM "h2o_feet"

name: h2o_feet
time                  sample  location      level description
----                  ------  --------      -----------------
2015-08-29T10:54:00Z  5.689   coyote_creek  between 3 and 6 feet
2015-09-08T15:48:00Z  6.391   coyote_creek  between 6 and 9 feet
```

查询返回`water_level`的两个随机抽样的字段值，以及其相关的tag`location`和field`level description`。

##### 例五：列出包含多个子句的field key的一个随机抽样的字段值
```
> SELECT SAMPLE("water_level",1) FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' AND "location" = 'santa_monica' GROUP BY time(18m)

name: h2o_feet
time                   sample
----                   ------
2015-08-18T00:12:00Z   2.028
2015-08-18T00:30:00Z   2.051
```

查询返回字段`water_level`的一个随机抽样的字段值。它涵盖`2015-08-18T00：00：00Z`和`2015-08-18T00：30：00Z`之间的时间段，并将结果按18分钟的时间间隔分组。

请注意，`GROUP BY time()`子句没有覆盖点的原始时间戳。有关该行为的更详细解释，请参阅下面的问题一。

#### SAMPLE()的常见问题
##### 问题一：`SAMPLE()`和`GROUP BY time()`
使用`SAMPLE()`和`GROUP BY time()`子句的查询返回每个`GROUP BY time()`间隔的指定点数(`N`)。对于大多数`GROUP BY time()`查询，返回的时间戳是每个`GROUP BY time()`间隔的开始。`GROUP BY time()`查询与`SAMPLE()`函数的行为不同; 它们保留原始数据点的时间戳。

例如

下面的查询每18分钟`GROUP BY time()`间隔返回两个随机的点。请注意，返回的时间戳是点的原始时间戳; 它们不会被强制置为`GROUP BY time()`间隔的开始。

```
> SELECT SAMPLE("water_level",2) FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' AND "location" = 'santa_monica' GROUP BY time(18m)

name: h2o_feet
time                   sample
----                   ------
                           __
2015-08-18T00:06:00Z   2.116 |
2015-08-18T00:12:00Z   2.028 | <------- Randomly-selected points for the first time interval
                           --
                           __
2015-08-18T00:18:00Z   2.126 |
2015-08-18T00:30:00Z   2.051 | <------- Randomly-selected points for the second time interval
 
```

### TOP()
返回最大的N个field值。
#### 语法
```
SELECT TOP(<field_key>[,<tag_key(s)>],<N> )[,<tag_key(s)>|<field_key(s)>] [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```
#### 语法描述
`TOP(field_key,N)`

返回field key的最大的N个field value。

`TOP(field_key,tag_key(s),N)`

返回某个tag key的N个tag value的最大的field value。

`TOP(field_key,N),tag_key(s),field_key(s)`

返回括号里的字段的最大N个field value，以及相关的tag或field，或者两者都有。

`TOP()`支持所有的数值类型的field。

>说明： 
> 
> * 如果一个field有两个或多个相等的field value，`TOP()`返回时间戳最早的那个。
> * `TOP()`和`INTO`子句一起使用的时候，和其他的函数有些不一样。

#### 例子
##### 例一：选择一个field的最大的三个值
```
> SELECT TOP("water_level",3) FROM "h2o_feet"

name: h2o_feet
time                   top
----                   ---
2015-08-29T07:18:00Z   9.957
2015-08-29T07:24:00Z   9.964
2015-08-29T07:30:00Z   9.954
```

该查询返回measurement`h2o_feet`的字段`water_level`的最大的三个值。

##### 例二：选择一个field的两个tag的分别最大的值
```
> SELECT TOP("water_level","location",2) FROM "h2o_feet"

name: h2o_feet
time                   top     location
----                   ---     --------
2015-08-29T03:54:00Z   7.205   santa_monica
2015-08-29T07:24:00Z   9.964   coyote_creek
```

该查询返回和tag`location`相关的两个tag值的字段`water_level`的分别最大值。

##### 例三：选择一个field的最大的四个值，以及其关联的tag和field
```
> SELECT TOP("water_level",4),"location","level description" FROM "h2o_feet"

name: h2o_feet
time                  top    location      level description
----                  ---    --------      -----------------
2015-08-29T07:18:00Z  9.957  coyote_creek  at or greater than 9 feet
2015-08-29T07:24:00Z  9.964  coyote_creek  at or greater than 9 feet
2015-08-29T07:30:00Z  9.954  coyote_creek  at or greater than 9 feet
2015-08-29T07:36:00Z  9.941  coyote_creek  at or greater than 9 feet
```

查询返回`water_level`中最大的四个字段值以及tag`location`和field`level description`的相关值。

##### 例四：选择一个field的最大的三个值，并且包括了多个子句
```
> SELECT TOP("water_level",3),"location" FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(24m) ORDER BY time DESC

name: h2o_feet
time                  top    location
----                  ---    --------
2015-08-18T00:48:00Z  7.11   coyote_creek
2015-08-18T00:54:00Z  6.982  coyote_creek
2015-08-18T00:54:00Z  2.054  santa_monica
2015-08-18T00:24:00Z  7.635  coyote_creek
2015-08-18T00:30:00Z  7.5    coyote_creek
2015-08-18T00:36:00Z  7.372  coyote_creek
2015-08-18T00:00:00Z  8.12   coyote_creek
2015-08-18T00:06:00Z  8.005  coyote_creek
2015-08-18T00:12:00Z  7.887  coyote_creek
```

查询将返回在`2015-08-18T00：00：00Z`和`2015-08-18T00：54：00Z`之间的每24分钟间隔内，`water_level`最大的三个值。它还以降序的时间戳顺序返回结果。 

请注意，`GROUP BY time()`子句不会覆盖点的原始时间戳。有关该行为的更详细解释，请参阅下面的问题一。

#### `TOP()`的常见问题
##### 问题一：`TOP()`和`GROUP BY time()`子句
`TOP()`和`GROUP BY time()`子句的查询返回每个`GROUP BY time()`间隔指定的点数。对于大多数`GROUP BY time()`查询，返回的时间戳被置为`GROUP BY time()`间隔的开始。`GROUP BY time()`查询与`TOP()`函数的行为不同; 它们保留原始数据点的时间戳。

例如

下面的查询返回每18分钟·`GROUP BY time()`间隔的两点。请注意，返回的时间戳是点的原始时间戳; 它们不会被强制匹配`GROUP BY time()`间隔的开始。

```
> SELECT TOP("water_level",2) FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' AND "location" = 'santa_monica' GROUP BY time(18m)

name: h2o_feet
time                   top
----                   ------
                           __
2015-08-18T00:00:00Z  2.064 |
2015-08-18T00:06:00Z  2.116 | <------- Greatest points for the first time interval
                           --
                           __
2015-08-18T00:18:00Z  2.126 |
2015-08-18T00:30:00Z  2.051 | <------- Greatest points for the second time interval
                           --
```

##### 问题二：`TOP()`和一个少于N个值得tag key
使用语法`SELECT TOP（<field_key>，<tag_key>，<N>）`的查询可以返回比预期少的点。如果tag具有X标签值，则查询指定N个值，当X小于N，则查询返回X点。

例如

下面的查询将要求tag`location`的三个值的`water_level`的最大字段值。由于`location`具有两个值（`santa_monica`和`coyote_creek`），所以查询返回两点而不是三个。

```
> SELECT TOP("water_level","location",3) FROM "h2o_feet"

name: h2o_feet
time                  top    location
----                  ---    --------
2015-08-29T03:54:00Z  7.205  santa_monica
2015-08-29T07:24:00Z  9.964  coyote_creek
```

##### 问题三：`TOP()`，tags和`INTO`子句
当与`INTO`子句和`GROUP BY tag`子句结合使用时，大多数InfluxQL函数将初始数据中的任何tag转换为新写入的数据中的field。此行为也适用于`TOP()`函数，除非`TOP()`包含一个tag key作为参数：`TOP(field_key，tag_key(s)，N)`。在这些情况下，系统将指定的tag作为新写入的数据中的tag。

例如

下面的代码块中的第一个查询返回与tag `location`相关联的两个tag value的field`water_level`中最大的字段值。它也将这些结果写入measurement`top_water_levels`。 第二个查询显示InfluxDB在`top_water_levels`中将`location`保存为tag。

```
> SELECT TOP("water_level","location",2) INTO "top_water_levels" FROM "h2o_feet"

name: result
time                 written
----                 -------
1970-01-01T00:00:00Z 2

> SHOW TAG KEYS FROM "top_water_levels"

name: top_water_levels
tagKey
------
location
```

## Transformations
### CEILING()
`CEILING()`已经不再是一个函数了，具体请查看[Issue #5930](https://github.com/influxdata/influxdb/issues/5930)。

### CUMULATIVE_SUM()
返回字段实时前序字段值的和。

#### 基本语法
```
SELECT CUMULATIVE_SUM( [ * | <field_key> | /<regular_expression>/ ] ) [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```
#### 基本语法描述
`CUMULATIVE_SUM(field_key)`

返回field key实时前序字段值的和。

`CUMULATIVE_SUM(/regular_expression/)`

返回满足正则表达式的所有字段的实时前序字段值的和。

`CUMULATIVE_SUM(*)`

返回measurement的所有字段的实时前序字段值的和。

`CUMULATIVE_SUM()`支持所有的数值类型的field。

基本语法支持`GROUP BY`tags子句，但是不支持`GROUP BY`时间。在高级语法中，`CUMULATIVE_SUM`支持`GROUP BY time()`子句。

#### 基本语法的例子
下面的1~4例子使用如下的数据：

```
> SELECT "water_level" FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' AND "location" = 'santa_monica'

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
##### 例一：计算一个字段的实时前序字段值的和。
```
> SELECT CUMULATIVE_SUM("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' AND "location" = 'santa_monica'

name: h2o_feet
time                   cumulative_sum
----                   --------------
2015-08-18T00:00:00Z   2.064
2015-08-18T00:06:00Z   4.18
2015-08-18T00:12:00Z   6.208
2015-08-18T00:18:00Z   8.334
2015-08-18T00:24:00Z   10.375
2015-08-18T00:30:00Z   12.426
```

该查询返回measurement`h2o_feet`的字段`water_level`的实时前序字段值的和。

##### 例二：计算measurement中每个字段的实时前序字段值的和

```
> SELECT CUMULATIVE_SUM(*) FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' AND "location" = 'santa_monica'

name: h2o_feet
time                   cumulative_sum_water_level
----                   --------------------------
2015-08-18T00:00:00Z   2.064
2015-08-18T00:06:00Z   4.18
2015-08-18T00:12:00Z   6.208
2015-08-18T00:18:00Z   8.334
2015-08-18T00:24:00Z   10.375
2015-08-18T00:30:00Z   12.426
```

该查询返回`h2o_feet`中每个数值类型的字段的实时前序字段值的和。`h2o_feet`只有一个数值类型的字段`water_level`。

##### 例三：计算measurement中满足正则表达式的每个字段的实时前序字段值的和。

```
> SELECT CUMULATIVE_SUM(/water/) FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' AND "location" = 'santa_monica'

name: h2o_feet
time                   cumulative_sum_water_level
----                   --------------------------
2015-08-18T00:00:00Z   2.064
2015-08-18T00:06:00Z   4.18
2015-08-18T00:12:00Z   6.208
2015-08-18T00:18:00Z   8.334
2015-08-18T00:24:00Z   10.375
2015-08-18T00:30:00Z   12.426
```

查询返回measurement中含有单词`word`的每个数值字段的实时前序字段值的和。

##### 例四：计算一个字段的实时前序字段值的和，并且包括了多个子句
```
> SELECT CUMULATIVE_SUM("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' AND "location" = 'santa_monica' ORDER BY time DESC LIMIT 4 OFFSET 2

name: h2o_feet
time                  cumulative_sum
----                  --------------
2015-08-18T00:18:00Z  6.218
2015-08-18T00:12:00Z  8.246
2015-08-18T00:06:00Z  10.362
2015-08-18T00:00:00Z  12.426
```

查询将返回在`2015-08-18T00：00：00Z`和`2015-08-18T00：30：00Z`之间的实时前序字段值的和，以降序的时间戳顺序返回结果。并且限制返回的数据点为4，偏移数据点2个。

#### 高级语法
```
SELECT CUMULATIVE_SUM(<function>( [ * | <field_key> | /<regular_expression>/ ] )) [INTO_clause] FROM_clause [WHERE_clause] GROUP_BY_clause [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
``` 

#### 高级语法描述
高级语法要求一个`GROUP BY time()`子句和一个嵌套的InfluxQL函数。查询首先计算在指定时间区间嵌套函数的结果，然后应用`CUMULATIVE_SUM()`函数的结果。

`CUMULATIVE_SUM()`支持以下嵌套函数：`COUNT(), MEAN(), MEDIAN(), MODE(), SUM(), FIRST(), LAST(), MIN(), MAX(), PERCENTILE()`。

#### 高级语法的例子
##### 例一：计算平均值的cumulative和
```
> SELECT CUMULATIVE_SUM(MEAN("water_level")) FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' AND "location" = 'santa_monica' GROUP BY time(12m)

name: h2o_feet
time                   cumulative_sum
----                   --------------
2015-08-18T00:00:00Z   2.09
2015-08-18T00:12:00Z   4.167
2015-08-18T00:24:00Z   6.213
```

该查询返回每隔12分钟的`water_level`的平均值的实时和。

为得到这个结果，InfluxDB首先计算每隔12分钟的平均`water_level`值：

```
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' AND "location" = 'santa_monica' GROUP BY time(12m)

name: h2o_feet
time                   mean
----                   ----
2015-08-18T00:00:00Z   2.09
2015-08-18T00:12:00Z   2.077
2015-08-18T00:24:00Z   2.0460000000000003
```

下一步，InfluxDB计算这些平均值的实时和。第二个点`4.167`是`2.09`和`2.077`的和，第三个点`6.213`是`2.09`,`2.077`和`2.04600000000003`的和。

### DERIVATIVE