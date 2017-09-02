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
>[…]  
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
CQs对实时数据进行操作。使用高级语法，CQ使用本地服务器的时间戳以及`RESAMPLE`子句中的信息和InfluxDB的预设时间边界来确定执行时间和查询中涵盖的时间范围。 

CQs以与`RESAMPLE`子句中的`EVERY`间隔相同的间隔执行，并且它们在InfluxDB的预设时间边界开始时运行。如果`EVERY`间隔是两个小时，InfluxDB将在每两小时的开始执行CQ。

当CQ执行时，它运行一个单一的查询，在`now()`和`now()`减去`RESAMPLE`子句中的`FOR`间隔之间的时间范围。如果`FOR`间隔为两个小时，当前时间为17:00，查询的时间间隔为15:00至16:59999999999。

`EVERY`间隔和`FOR`间隔都接受时间字符串。`RESAMPLE`子句适用于同时配置`EVERY`和`FOR`,或者是其中之一。如果没有提供`EVERY`间隔或`FOR`间隔，则CQ默认为相关为基本语法。

#### 高级语法例子
示例数据如下：

```
name: bus_data
--------------
time                   passengers
2016-08-28T06:30:00Z   2
2016-08-28T06:45:00Z   4
2016-08-28T07:00:00Z   5
2016-08-28T07:15:00Z   8
2016-08-28T07:30:00Z   8
2016-08-28T07:45:00Z   7
2016-08-28T08:00:00Z   8
2016-08-28T08:15:00Z   15
2016-08-28T08:30:00Z   15
2016-08-28T08:45:00Z   17
2016-08-28T09:00:00Z   20
```

##### 例一：配置执行间隔
在`RESAMPLE`中使用`EVERY`来指明CQ的执行间隔。

```
CREATE CONTINUOUS QUERY "cq_advanced_every" ON "transportation"
RESAMPLE EVERY 30m
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(1h)
END
```

`cq_advanced_every`从`bus_data`中计算`passengers`的一小时平均值，并将结果存储在数据库`transportation`中的`average_passengers`中。

`cq_advanced_every`以30分钟的间隔执行，间隔与`EVERY`间隔相同。每30分钟，`cq_advanced_every`运行一个查询，覆盖当前时间段的时间范围，即与`now()`交叉的一小时时间段。

下面是2016年8月28日上午的日志输出：

>在8:00`cq_basic_every`执行时间范围`time> ='7:00'AND time <'8:00'`的查询。
`cq_basic_every`向`average_passengers`写入一个点：
>
```
name: average_passengers
------------------------
time                   mean
2016-08-28T07:00:00Z   7
```
>在8:30`cq_basic_every`执行时间范围`time> ='8:00'AND time <'9:00'`的查询。
`cq_basic_every`向`average_passengers`写入一个点：
>
```
name: average_passengers
------------------------
time                   mean
2016-08-28T08:00:00Z   12.6667
```
>在9:00`cq_basic_every`执行时间范围`time> ='8:00'AND time <'9:00'`的查询。
`cq_basic_every`向`average_passengers`写入一个点：
>
```
name: average_passengers
------------------------
time                   mean
2016-08-28T08:00:00Z   13.75
```

结果为：

```
> SELECT * FROM "average_passengers"
name: average_passengers
------------------------
time                   mean
2016-08-28T07:00:00Z   7
2016-08-28T08:00:00Z   13.75
```

请注意，`cq_advanced_every`计算8:00时间间隔的结果两次。第一次，它运行在8:30，计算每个可用数据点在8:00和9:00（8,15和15）之间的平均值。 第二次，它运行在9:00，计算每个可用数据点在8:00和9:00（8,15,15和17）之间的平均值。由于InfluxDB处理重复点的方式，所以第二个结果只是覆盖第一个结果。

##### 例二：配置CQ的重采样时间范围
在`RESAMPLE`中使用`FOR`来指明CQ的时间间隔的长度。

```
CREATE CONTINUOUS QUERY "cq_advanced_for" ON "transportation"
RESAMPLE FOR 1h
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(30m)
END
```

`cq_advanced_for`从`bus_data`中计算`passengers`的30分钟平均值，并将结果存储在数据库`transportation`中的`average_passengers`中。

`cq_advanced_for`以30分钟的间隔执行，间隔与`GROUP BY time()`间隔相同。每30分钟，`cq_advanced_for`运行一个查询，覆盖时间段为`now()`和`now()`减去`FOR`中的间隔，即是`now()`和`now()`之前的一个小时之间的时间范围。


下面是2016年8月28日上午的日志输出：

>在8:00`cq_advanced_for`执行时间范围`time> ='7:00'AND time <'8:00'`的查询。
`cq_advanced_for`向`average_passengers`写入一个点：
>
```
name: average_passengers
------------------------
time                   mean
2016-08-28T07:00:00Z   6.5
2016-08-28T07:30:00Z   7.5
```
>在8:30`cq_advanced_for`执行时间范围`time> ='7:30'AND time <'8:30'`的查询。
`cq_advanced_for`向`average_passengers`写入两个点：
>
```
name: average_passengers
------------------------
time                   mean
2016-08-28T07:30:00Z   7.5
2016-08-28T08:00:00Z   11.5
```
>在9:00`cq_advanced_for `执行时间范围`time> ='8:00'AND time <'9:00'`的查询。
`cq_advanced_for `向`average_passengers`写入两个点：
>
```
name: average_passengers
------------------------
time                   mean
2016-08-28T08:00:00Z   11.5
2016-08-28T08:30:00Z   16
```

请注意，`cq_advanced_for`会计算每次间隔两次的结果。CQ在8:00和8:30计算7:30的平均值，在8:30和9:00计算8:00的平均值。

结果为：

```
> SELECT * FROM "average_passengers"
name: average_passengers
------------------------
time                   mean
2016-08-28T07:00:00Z   6.5
2016-08-28T07:30:00Z   7.5
2016-08-28T08:00:00Z   11.5
2016-08-28T08:30:00Z   16
```

##### 例三：配置执行间隔和CQ时间范围
在`RESAMPLE`子句中使用`EVERY`和`FOR`来指定CQ的执行间隔和CQ的时间范围长度。

```
CREATE CONTINUOUS QUERY "cq_advanced_every_for" ON "transportation"
RESAMPLE EVERY 1h FOR 90m
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(30m)
END
```

`cq_advanced_every_for`从`bus_data`中计算`passengers`的30分钟平均值，并将结果存储在数据库`transportation`中的`average_passengers`中。

`cq_advanced_every_for`以1小时的间隔执行，间隔与`EVERY`间隔相同。每1小时，`cq_advanced_every_for`运行一个查询，覆盖时间段为`now()`和`now()`减去`FOR`中的间隔，即是`now()`和`now()`之前的90分钟之间的时间范围。


下面是2016年8月28日上午的日志输出：

>在8:00`cq_advanced_every_for`执行时间范围`time>='6:30'AND time <'8:00'`的查询。
`cq_advanced_every_for`向`average_passengers`写三个个点：
>
```
name: average_passengers
------------------------
time                   mean
2016-08-28T06:30:00Z   3
2016-08-28T07:00:00Z   6.5
2016-08-28T07:30:00Z   7.5
```
>在9:00`cq_advanced_every_for`执行时间范围`time> ='7:30'AND time <'9:00'`的查询。
`cq_advanced_every_for `向`average_passengers`写入三个点：
>
```
name: average_passengers
------------------------
time                   mean
2016-08-28T07:30:00Z   7.5
2016-08-28T08:00:00Z   11.5
2016-08-28T08:30:00Z   16
```

请注意，`cq_advanced_every_for`会计算每次间隔两次的结果。CQ在8:00和9:00计算7:30的平均值。

结果为：

```
> SELECT * FROM "average_passengers"
name: average_passengers
------------------------
time                   mean
2016-08-28T06:30:00Z   3
2016-08-28T07:00:00Z   6.5
2016-08-28T07:30:00Z   7.5
2016-08-28T08:00:00Z   11.5
2016-08-28T08:30:00Z   16
```

##### 例四：配置CQ的时间范围并填充空值
使用`FOR`间隔和`fill()`来更改不含数据的时间间隔值。请注意，至少有一个数据点必须在`fill()`运行的`FOR`间隔内。 如果没有数据落在`FOR`间隔内，则CQ不会将任何点写入目标measurement。

```
CREATE CONTINUOUS QUERY "cq_advanced_for_fill" ON "transportation"
RESAMPLE FOR 2h
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(1h) fill(1000)
END
```

`cq_advanced_for_fill`从`bus_data`中计算`passengers`的1小时的平均值，并将结果存储在数据库`transportation`中的`average_passengers`中。并会在没有结果的时间间隔里写入值`1000`。

`cq_advanced_for_fill`以1小时的间隔执行，间隔与`GROUP BY time()`间隔相同。每1小时，`cq_advanced_for_fill`运行一个查询，覆盖时间段为`now()`和`now()`减去`FOR`中的间隔，即是`now()`和`now()`之前的两小时之间的时间范围。

下面是2016年8月28日上午的日志输出：

>在6:00`cq_advanced_for_fill`执行时间范围`time>='4:00'AND time <'6:00'`的查询。
`cq_advanced_for_fill`向`average_passengers`不写入任何点，因为在那个时间范围`bus_data`没有数据：
>
>在7:00`cq_advanced_for_fill`执行时间范围`time>='5:00'AND time <'7:00'`的查询。
`cq_advanced_for_fill`向`average_passengers`写入两个点：
>
```
name: average_passengers
------------------------
time                   mean
2016-08-28T05:00:00Z   1000          <------ fill(1000)
2016-08-28T06:00:00Z   3             <------ 2和4的平均值
```
> [...]
> 
>在11:00`cq_advanced_for_fill`执行时间范围`time> ='9:00'AND time <'11:00'`的查询。
`cq_advanced_for_fill`向`average_passengers`写入两个点：
>
```
name: average_passengers
------------------------
2016-08-28T09:00:00Z   20            <------ 20的平均
2016-08-28T10:00:00Z   1000          <------ fill(1000)     
```
>
> 在12:00`cq_advanced_for_fill`执行时间范围`time>='10:00'AND time <'12:00'`的查询。
`cq_advanced_for_fill`向`average_passengers`不写入任何点，因为在那个时间范围`bus_data`没有数据.

结果：

```
> SELECT * FROM "average_passengers"
name: average_passengers
------------------------
time                   mean
2016-08-28T05:00:00Z   1000
2016-08-28T06:00:00Z   3
2016-08-28T07:00:00Z   7
2016-08-28T08:00:00Z   13.75
2016-08-28T09:00:00Z   20
2016-08-28T10:00:00Z   1000
```

>注意：如果前一个值在查询时间之外，则`fill(previous)`不会在时间间隔里填充数据。


#### 高级语法的共同问题
##### 问题一：如果`EVERY`间隔大于`GROUP BY time()`的间隔
如果`EVERY`间隔大于`GROUP BY time()`间隔，则CQ以与`EVERY`间隔相同的间隔执行，并运行一个单个查询，该查询涵盖`now()`和`now()`减去`EVERY`间隔之间的时间范围(不是在`now()`和`now()`减去`GROUP BY time()`间隔之间）。

例如，如果`GROUP BY time()`间隔为5m，并且`EVERY`间隔为10m，则CQ每10分钟执行一次。每10分钟，CQ运行一个查询，覆盖`now()`和`now()`减去`EVERY`间隔之间的时间段，即`now()`到`now()`之前十分钟之间的时间范围。 

此行为是故意的，并防止CQ在执行时间之间丢失数据。

##### 问题二：如果`IF`间隔比执行的间隔少
如果`FOR`间隔比`GROUP BY time()`或者`EVERY`的间隔少，InfluxDB返回如下错误：

```
error parsing query: FOR duration must be >= GROUP BY time duration: must be a minimum of <minimum-allowable-interval> got <user-specified-interval>
```

为了避免在执行时间之间丢失数据，`FOR`间隔必须等于或大于`GROUP BY time()`或者`EVERY`间隔。

目前，这是预期的行为。GitHub上[Issue＃6963](https://github.com/influxdata/influxdb/issues/6963)要求CQ支持数据覆盖的差距。

## CQ的管理
只有admin用户允许管理CQ。

### 列出CQ
列出InfluxDB实例上的所有CQ：

```
SHOW CONTINUOUS QUERIES
```

`SHOW CONTINUOUS QUERIES`按照database作分组。

#### 例子
下面展示了`telegraf`和`mydb`的CQ：

```
> SHOW CONTINUOUS QUERIES
name: _internal
---------------
name   query


name: telegraf
--------------
name           query
idle_hands     CREATE CONTINUOUS QUERY idle_hands ON telegraf BEGIN SELECT min(usage_idle) INTO telegraf.autogen.min_hourly_cpu FROM telegraf.autogen.cpu GROUP BY time(1h) END
feeling_used   CREATE CONTINUOUS QUERY feeling_used ON telegraf BEGIN SELECT mean(used) INTO downsampled_telegraf.autogen.:MEASUREMENT FROM telegraf.autogen./.*/ GROUP BY time(1h) END


name: downsampled_telegraf
--------------------------
name   query


name: mydb
----------
name      query
vampire   CREATE CONTINUOUS QUERY vampire ON mydb BEGIN SELECT count(dracula) INTO mydb.autogen.all_of_them FROM mydb.autogen.one GROUP BY time(5m) END
```

### 删除CQ
从一个指定的database删除CQ：

```
DROP CONTINUOUS QUERY <cq_name> ON <database_name>
```

`DROP CONTINUOUS QUERY`返回一个空的结果。

#### 例子
从数据库`telegraf`中删除`idle_hands`这个CQ：

```
> DROP CONTINUOUS QUERY "idle_hands" ON "telegraf"`
>
```

### 修改CQ
CQ一旦创建就不能删除了，你必须`DROP`再`CREATE`才行。

## CQ的使用场景
### 采样和数据保留
使用CQ与InfluxDB的保留策略（RP）来减轻存储问题。结合CQ和RP自动将高精度数据降低到较低的精度，并从数据库中移除可分配的高精度数据。 

### 预先计算昂贵的查询
通过使用CQ预先计算昂贵的查询来缩短查询运行时间。使用CQ自动将普通查询的高精度数据下采样到较低的精度。较低精度数据的查询需要更少的资源并且返回更快。 

提示：预先计算首选图形工具的查询，以加速图形和仪表板的展示。

### 替换HAVING子句
InfluxQL不支持`HAVING`子句。通过创建CQ来聚合数据并查询CQ结果以达到应用`HAVING`子句相同的功能。

>注意：InfluxDB提供了子查询也可以达到类似于`HAVING`相同的功能。

#### 例子
InfluxDB不接受使用`HAVING`子句的以下查询。该查询以30分钟间隔计算平均`bees`数，并请求大于20的平均值。

```
SELECT mean("bees") FROM "farm" GROUP BY time(30m) HAVING mean("bees") > 20
```

要达到相同的结果：

##### 1. 创建一个CQ
此步骤执行以上查询的`mean("bees")`部分。因为这个步骤创建了CQ，所以只需要执行一次。 

以下CQ自动以30分钟间隔计算`bees`的平均数，并将这些平均值写入measurement`aggregate_bees`中的`mean_bees`字段。

```
CREATE CONTINUOUS QUERY "bee_cq" ON "mydb" BEGIN SELECT mean("bees") AS "mean_bees" INTO "aggregate_bees" FROM "farm" GROUP BY time(30m) END
```

##### 2. 查询CQ的结果
这一步要实现`HAVING mean("bees") > 20`部分的查询。

在`WHERE`子句中查询measurement`aggregate_bees`中的数据和大于20的`mean_bees`字段的请求值：

```
SELECT "mean_bees" FROM "aggregate_bees" WHERE "mean_bees" > 20
```

### 替换嵌套函数
一些InfluxQL函数支持嵌套其他函数，大多数是不行的。如果函数不支持嵌套，可以使用CQ获得相同的功能来计算最内部的函数。然后简单地查询CQ结果来计算最外层的函数。 

>注意：InfluxQL支持也提供与嵌套函数相同功能的子查询。 

#### 例子

InfluxDB不接受使用嵌套函数的以下查询。 该查询以30分钟间隔计算`bees`的非空值数量，并计算这些计数的平均值：

```
SELECT mean(count("bees")) FROM "farm" GROUP BY time(30m)
```

为了得到结果：

##### 1. 创建一个CQ
此步骤执行上面的嵌套函数的`count(“bees”)`部分 因为这个步骤创建了一个CQ，所以只需要执行一次。 以下CQ自动以30分钟间隔计算`bees`的非空值数，并将这些计数写入`aggregate_bees`中的`count_bees`字段。

```
CREATE CONTINUOUS QUERY "bee_cq" ON "mydb" BEGIN SELECT count("bees") AS "count_bees" INTO "aggregate_bees" FROM "farm" GROUP BY time(30m) END
```

##### 2. 查询CQ的结果
此步骤执行上面的嵌套函数的`mean([...])`部分。 在`aggregate_bees`中查询数据，以计算`count_bees`字段的平均值：

```
SELECT mean("count_bees") FROM "aggregate_bees" WHERE time >= <start_time> AND time <= <end_time>
```