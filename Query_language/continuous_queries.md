# 连续查询
## 介绍
连续查询(Continuous Queries下文统一简称CQ)是InfluxQL对实时数据自动周期运行的查询，然后把查询结果写入到指定的measurement中。

## 语法
### 基本语法
```
CREATE CONTINUOUS QUERY <cq_name> ON <database_name>
BEGIN
  <cq_query>
END
```

#### 语法描述
##### cq_query
`cq_query`需要一个函数，一个`INTO`子句和一个`GROUP BY time()`子句：

```
SELECT <function[s]> INTO <destination_measurement> FROM <measurement> [WHERE <stuff>] GROUP BY time(<interval>)[,<tag_key[s]>]
```

>注意：请注意，在`WHERE`子句中，`cq_query`不需要时间范围。 InfluxDB在执行CQ时自动生成`cq_query`的时间范围。`cq_query`的`WHERE`子句中的任何用户指定的时间范围将被系统忽略。

##### 运行时间点以及覆盖的时间范围

CQ对实时数据进行操作。他们使用本地服务器的时间戳，`GROUP BY time()`间隔和InfluxDB的预设时间边界来确定何时执行以及查询中涵盖的时间范围。

CQs以与`cq_query`的`GROUP BY time()`间隔相同的间隔执行，并且它们在InfluxDB的预设时间边界开始时运行。如果`GROUP BY time()`间隔为1小时，则CQ每小时开始执行一次。

当CQ执行时，它对于`now()`和`now()`减去`GROUP BY time()`间隔的时间范围运行单个查询。 如果`GROUP BY time()`间隔为1小时，当前时间为17:00，查询的时间范围为16:00至16:59999999999。

#### 基本语法的例子
以下例子使用数据库`transportation`中的示例数据。measurement`bus_data`数据存储有关公共汽车乘客数量和投诉数量的15分钟数据：

```
name: bus_data
--------------
time                   passengers   complaints
2016-08-28T07:00:00Z   5            9
2016-08-28T07:15:00Z   8            9
2016-08-28T07:30:00Z   8            9
2016-08-28T07:45:00Z   7            9
2016-08-28T08:00:00Z   8            9
2016-08-28T08:15:00Z   15           7
2016-08-28T08:30:00Z   15           7
2016-08-28T08:45:00Z   17           7
2016-08-28T09:00:00Z   20           7
```

##### 例一：自动采样数据
使用简单的CQ自动从单个字段中下采样数据，并将结果写入同一数据库中的另一个measurement。

```
CREATE CONTINUOUS QUERY "cq_basic" ON "transportation"
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(1h)
END
```

`cq_basic`从`bus_data`中计算乘客的平均小时数，并将结果存储在数据库`transportation`中的`average_passengers`中。 

`cq_basic`以一小时的间隔执行，与`GROUP BY time()`间隔相同的间隔。 每个小时，`cq_basic`运行一个单一的查询，覆盖了`now()`和`now()`减去`GROUP BY time()`间隔之间的时间范围，即`now()`和`now()`之前的一个小时之间的时间范围。 

下面是2016年8月28日上午的日志输出：

>在8点时，`cq_basic`执行时间范围为`time => '7:00' AND time <'08：00'`的查询。
`cq_basic`向`average_passengers`写入一个点：
>
```
name: average_passengers
------------------------
time                   mean
2016-08-28T07:00:00Z   7
```
>在9点时，`cq_basic`执行时间范围为`time => '8:00' AND time <'09：00'`的查询。
`cq_basic`向`average_passengers`写入一个点：
>
```
name: average_passengers
------------------------
time                   mean
2016-08-28T08:00:00Z   13.75
```

结果：

```
> SELECT * FROM "average_passengers"
name: average_passengers
------------------------
time                   mean
2016-08-28T07:00:00Z   7
2016-08-28T08:00:00Z   13.75
```

##### 例二：自动采样数据到另一个保留策略里
从默认的的保留策略里面采样数据到完全指定的目标measurement中：

```
CREATE CONTINUOUS QUERY "cq_basic_rp" ON "transportation"
BEGIN
  SELECT mean("passengers") INTO "transportation"."three_weeks"."average_passengers" FROM "bus_data" GROUP BY time(1h)
END
```

`cq_basic_rp`从`bus_data`中计算乘客的平均小时数，并将结果存储在数据库`tansportation`的RP为`three_weeks`的measurement`average_passengers`中。 

`cq_basic_rp`以一小时的间隔执行，与`GROUP BY time()`间隔相同的间隔。每个小时，`cq_basic_rp`运行一个单一的查询，覆盖了`now()`和`now()`减去`GROUP BY time()`间隔之间的时间段，即`now()`和`now()`之前的一个小时之间的时间范围。 

下面是2016年8月28日上午的日志输出：

>在8:00`cq_basic_rp`执行时间范围为` time >='7:00' AND time <'8:00'`的查询。`cq_basic_rp`向RP为`three_weeks`的measurement`average_passengers`写入一个点：
>
```
name: average_passengers
------------------------
time                   mean
2016-08-28T07:00:00Z   7
```
>在9:00`cq_basic_rp`执行时间范围为` time >='8:00' AND time <'9:00'`的查询。`cq_basic_rp`向RP为`three_weeks`的measurement`average_passengers`写入一个点：
>
```
name: average_passengers
------------------------
time                   mean
2016-08-28T08:00:00Z   13.75
```

结果：

```
> SELECT * FROM "transportation"."three_weeks"."average_passengers"
name: average_passengers
------------------------
time                   mean
2016-08-28T07:00:00Z   7
2016-08-28T08:00:00Z   13.75
```

`cq_basic_rp`使用CQ和保留策略自动降低样本数据，并将这些采样数据保留在不同的时间长度上。

##### 例三：使用逆向引用自动采样数据
使用带有通配符（`*`）和`INTO`查询的反向引用语法的函数可自动对数据库中所有measurement和数值字段中的数据进行采样。

```
CREATE CONTINUOUS QUERY "cq_basic_br" ON "transportation"
BEGIN
  SELECT mean(*) INTO "downsampled_transportation"."autogen".:MEASUREMENT FROM /.*/ GROUP BY time(30m),*
END
```

`cq_basic_br`计算数据库`transportation`中每个measurement的30分钟平均乘客和投诉。它将结果存储在数据库`downsampled_transportation`中。 

`cq_basic_br`以30分钟的间隔执行，与`GROUP BY time()`间隔相同的间隔。每30分钟一次，`cq_basic_br`运行一个查询，覆盖了`now()`和`now()`减去`GROUP BY time()`间隔之间的时间段，即`now()`到`now()`之前的30分钟之间的时间范围。

下面是2016年8月28日上午的日志输出：

>在7:30，`cq_basic_br`执行查询，时间间隔 `time >='7:00' AND time <'7:30'`。`cq_basic_br`向`downsampled_transportation`数据库中的measurement为`bus_data`写入两个点：
>
```
name: bus_data
--------------
time                   mean_complaints   mean_passengers
2016-08-28T07:00:00Z   9                 6.5
```
>8点时，`cq_basic_br`执行时间范围为 `time >='7:30' AND time <'8:00'`的查询。`cq_basic_br`向`downsampled_transportation`数据库中measurement为`bus_data`写入两个点：
>
```
name: bus_data
--------------
time                   mean_complaints   mean_passengers
2016-08-28T07:30:00Z   9                 7.5
```
>
[…]  
>
>9点时，`cq_basic_br`执行时间范围为 `time >='8:30' AND time <'9:00'`的查询。`cq_basic_br`向`downsampled_transportation`数据库中measurement为`bus_data`写入两个点：
>
```
name: bus_data
--------------
time                   mean_complaints   mean_passengers
2016-08-28T08:30:00Z   7                 16
```

结果为：

```
> SELECT * FROM "downsampled_transportation."autogen"."bus_data"
name: bus_data
--------------
time                   mean_complaints   mean_passengers
2016-08-28T07:00:00Z   9                 6.5
2016-08-28T07:30:00Z   9                 7.5
2016-08-28T08:00:00Z   8                 11.5
2016-08-28T08:30:00Z   7                 16
```

##### 例四：自动采样数据并配置CQ的时间边界
使用`GROUP BY time()`子句的偏移间隔来改变CQ的默认执行时间和呈现的时间边界：

```
CREATE CONTINUOUS QUERY "cq_basic_offset" ON "transportation"
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(1h,15m)
END
```

`cq_basic_offset`从`bus_data`中计算乘客的平均小时数，并将结果存储在`average_passengers`中。

`cq_basic_offset`以一小时的间隔执行，与`GROUP BY time()`间隔相同的间隔。15分钟偏移间隔迫使CQ在默认执行时间后15分钟执行; `cq_basic_offset`在8:15而不是8:00执行。

每个小时，`cq_basic_offset`运行一个单一的查询，覆盖了`now()`和`now()`减去`GROUP BY time()`间隔之间的时间段，即`now()`和`now()`之前的一个小时之间的时间范围。 15分钟偏移间隔在CQ的`WHERE`子句中向前移动生成的预设时间边界; `cq_basic_offset`在7:15和8：14.999999999而不是7:00和7：59.999999999之间进行查询。 

下面是2016年8月28日上午的日志输出：

>在8:15`cq_basic_offset`执行时间范围`time> ='7:15'AND time <'8:15'`的查询。
`cq_basic_offset`向`average_passengers`写入一个点：
>
```
name: average_passengers
------------------------
time                   mean
2016-08-28T07:15:00Z   7.75
```
>在9:15`cq_basic_offset`执行时间范围`time> ='8:15'AND time <'9:15'`的查询。
`cq_basic_offset`向`average_passengers`写入一个点：
>
```
name: average_passengers
------------------------
time                   mean
2016-08-28T08:15:00Z   16.75
```

结果为：

```
> SELECT * FROM "average_passengers"
name: average_passengers
------------------------
time                   mean
2016-08-28T07:15:00Z   7.75
2016-08-28T08:15:00Z   16.75
```

请注意，时间戳为7:15和8:15而不是7:00和8:00。

#### 基本语法的常见问题
##### 问题一：无数据处理时间间隔
如果没有数据落在该时间范围内，则CQ不会在时间间隔内写入任何结果。请注意，基本语法不支持使用`fill()`更改不含数据的间隔报告的值。如果基本语法CQs包括了`fill()`，则会忽略`fill()`。一个解决办法是使用下面的高级语法。

##### 问题二：重新采样以前的时间间隔
基本的CQ运行一个查询，覆盖了`now()`和`now()`减去`GROUP BY time()`间隔之间的时间段。有关如何配置查询的时间范围，请参阅高级语法。

##### 问题三：旧数据的回填结果
CQ对实时数据进行操作，即具有相对于`now()`发生的时间戳的数据。使用基本的`INTO`查询来回填具有较旧时间戳的数据的结果。

##### 问题四：CQ结果中缺少tag
默认情况下，所有`INTO`查询将源measurement中的任何tag转换为目标measurement中的field。

在CQ中包含`GROUP BY *`，以保留目的measurement中的tag。

### 高级语法
```
CREATE CONTINUOUS QUERY <cq_name> ON <database_name>
RESAMPLE EVERY <interval> FOR <interval>
BEGIN
  <cq_query>
END
```
#### 高级语法描述
##### cq_query
同上面基本语法里面的`cq_query`。
##### 运行时间点以及覆盖的时间范围


