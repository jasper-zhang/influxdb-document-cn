# 行协议

InfluxDB的行协议是一种写入数据点到InfluxDB的文本格式。必须要是这样的格式的数据点才能被Influxdb解析和写入成功，当然除非你使用一些其他服务插件。

使用虚构的温度数据，本页面介绍了行协议。 它涵盖：

|语法|数据类型|引号|特殊字符和关键字|
|:--:| :--:|:--: |:--: |

最后一节，将数据写入InfluxDB，介绍如何将数据存入InfluxDB，以及InfluxDB如何处理行协议重复问题。

## 语法
一行Line Protocol表示InfluxDB中的一个数据点。它向InfluxDB通知点的measurement，tag set，field set和timestamp。以下代码块显示了行协议的示例，并将其分解为其各个组件：

```
weather,location=us-midwest temperature=82 1465839830100400200
  |    -------------------- --------------  |
  |             |             |             |
  |             |             |             |
+-----------+--------+-+---------+-+---------+
|measurement|,tag_set| |field_set| |timestamp|
+-----------+--------+-+---------+-+---------+
```
### measurement
你想要写入数据的measurement，这在行协议中是必需的，例如这里的measurement是`weather`。

### Tag set
你想要数据点中包含的tag，tag在行协议里是可选的。注意measurement和tag set是用不带空格的逗号分开的。

用不带空格的`=`来分割一组tag的键值：

```
<tag_key>=<tag_value>
```

多组tag直接用不带空格的逗号分开：

```
<tag_key>=<tag_value>,<tag_key>=<tag_value>
```

例如上面的tag set由一个tag组成`location=us-midwest`，现在加另一个tag(`season=summer`)，就变成了这样：

```
weather,location=us-midwest,season=summer temperature=82 1465839830100400200
```

为了获得最佳性能，您应该在将它们发送到数据库之前按键进行排序。排序应该与[Go bytes.Compare function](http://golang.org/pkg/bytes/#Compare)的结果相匹配。

### 空格1
分离measurement和field set，或者如果您使用数据点包含tag set，则使用空格分隔tag set和field set。行协议中空格是必需的。

没有tag set的有效行协议：

```
weather temperature=82 1465839830100400200
```

### Field set
每个数据点在行协议中至少需要一个field。使用无空格的`=`分隔field的键值对：

```
<field_key>=<field_value>
```

多组field直接用不带空格的逗号分开：

```
<field_key>=<field_value>,<field_key>=<field_value>
```

例如上面的field set由一个field组成`temperature=82`，现在加另一个field(`bug_concentration=98`)，就变成了这样：

```
weather,location=us-midwest temperature=82,bug_concentration=98 1465839830100400200
```

### 空格2
使用空格分隔field set和可选的时间戳。如果你包含时间戳，则行协议中需要空格。

### Timestamp
数据点的时间戳记以毫秒精度Unix时间。行协议中的时间戳是可选的。 如果没有为数据点指定时间戳，InfluxDB会使用服务器的本地纳秒时间戳。

在这个例子中，时间戳记是`1465839830100400200`（这就是RFC6393格式的`2016-06-13T17：43：50.1004002Z`）。下面的行协议是相同的数据点，但没有时间戳。当InfluxDB将其写入数据库时，它将使用您的服务器的本地时间戳而不是`2016-06-13T17：43：50.1004002Z`。

```
weather,location=us-midwest temperature=82
```

使用HTTP API来指定精度超过纳秒的时间戳，例如微秒，毫秒或秒。我们建议使用最粗糙的精度，因为这样可以显着提高压缩率。 有关详细信息，请参阅[API参考]。

>小贴士：  
>使用网络时间协议（NTP）来同步主机之间的时间。InfluxDB使用主机在UTC的本地时间为数据分配时间戳; 如果主机的时钟与NTP不同步，写入InfluxDB的数据的时间戳可能不正确。

## 数据类型
本节介绍行协议的主要组件的数据类型：measurement，tag keys，tag values，field keys，field values和timestamp。

其中measurement，tag keys，tag values，field keys始终是字符串。

>注意：因为InfluxDB将tag value存储为字符串，所以InfluxDB无法对tag value进行数学运算。此外，InfluxQL函数不接受tag value作为主要参数。 在设计架构时要考虑到这些信息。

Timestamps是UNIX时间戳。 最小有效时间戳为`-9223372036854775806`或`1677-09-21T00：12：43.145224194Z`。最大有效时间戳为`9223372036854775806`或`2262-04-11T23：47：16.854775806Z`。 如上所述，默认情况下，InfluxDB假定时间戳具有纳秒精度。有关如何指定替代精度，请参阅[API参考]()。

Field value可以是整数、浮点数、字符串和布尔值：

* 浮点数 —— 默认是浮点数，InfluxDB假定收到的所有field value都是浮点数。  
 以浮点类型存储上面的`82`：
 
 ```
 weather,location=us-midwest temperature=82 1465839830100400200
 ```

* 整数 —— 添加一个`i`在field之后，告诉InfluxDB以整数类型存储：  
 以整数类型存储上面的`82`：
 
 ```
 weather,location=us-midwest temperature=82i 1465839830100400200
 ```
 
* 字符串 —— 双引号把字段值引起来表示字符串:  
 以字符串类型存储值`too warm`：
 
 ```
 weather,location=us-midwest temperature="too warm" 1465839830100400200
 ```
 
* 布尔型 —— 表示TRUE可以用`t`,`T`,`true`,`True`,`TRUE`;表示FALSE可以用`f`,`F`,`false`,`False`或者`FALSE`：   
 以布尔类型存储值`true`：

```
weather,location=us-midwest too_hot=true 1465839830100400200
```

>注意：数据写入和数据查询可接受的布尔语法不同。 有关详细信息，请参阅[常见问题]()。

在measurement中，field value的类型在分片内不会有差异，但在分片之间可能会有所不同。例如，如果InfluxDB尝试将整数写入到与浮点数相同的分片中，则写入会失败：

```
> INSERT weather,location=us-midwest temperature=82 1465839830100400200
> INSERT weather,location=us-midwest temperature=81i 1465839830100400300
ERR: {"error":"field type conflict: input field \"temperature\" on measurement \"weather\" is type int64, already exists as type float"}
```

但是，如果InfluxDB将整数写入到一个新的shard中，虽然之前写的是浮点数，那依然可以写成功：

```
> INSERT weather,location=us-midwest temperature=82 1465839830100400200
> INSERT weather,location=us-midwest temperature=81i 1467154750000000000
>
```

有关字段值类型差异如何影响`SELECT *`查询的，请参阅[常见问题]()。

## 引号
本节涵盖何时不要和何时不要在行协议中使用双（`“`）或单（`'`）引号。

* 时间戳不要双或单引号。下面这是无效的行协议。 例：

```
> INSERT weather,location=us-midwest temperature=82 "1465839830100400200"
ERR: {"error":"unable to parse 'weather,location=us-midwest temperature=82 \"1465839830100400200\"': bad timestamp"}
```

* field value不要单引号,即时是字符串类型。下面这是无效的行协议。 例：

```
> INSERT weather,location=us-midwest temperature='too warm'
ERR: {"error":"unable to parse 'weather,location=us-midwest temperature='too warm'': invalid boolean"}
```

* measurement名称，tag keys，tag value和field key不用单双引号。InfluxDB会假定引号是名称的一部分。例如：

```
> INSERT weather,location=us-midwest temperature=82 1465839830100400200
> INSERT "weather",location=us-midwest temperature=87 1465839830100400200
> SHOW MEASUREMENTS
name: measurements
------------------
name
"weather"
weather
```

查询数据中的`"weather"`，你需要为measurement名称中的引号转义：

```
> SELECT * FROM "\"weather\""
name: "weather"
---------------
time                            location     temperature
2016-06-13T17:43:50.1004002Z    us-midwest   87
```

* 当field value是整数，浮点数或是布尔型时，不要使用双引号，不然InfluxDB会假定值是字符串类型：

```
> INSERT weather,location=us-midwest temperature="82"
> SELECT * FROM weather WHERE temperature >= 70
>
```

* 当Field value是字符串时，使用双引号：

```
> INSERT weather,location=us-midwest temperature="too warm"
> SELECT * FROM weather
name: weather
-------------
time                            location     temperature
2016-06-13T19:10:09.995766248Z  us-midwest   too warm
```

## 特殊字符和关键字
### 特殊字符
对于tag key，tag value和field key，始终使用反斜杠字符\来进行转义：

* 逗号`,`：

```
weather,location=us\,midwest temperature=82 1465839830100400200
```

* 等号`=`：

```
weather,location=us-midwest temp\=rature=82 1465839830100400200
```

* 空格：

```
weather,location\ place=us-midwest temperature=82 1465839830100400200
```

对于measurement，也要反斜杠\来转义。

* 逗号`,`：

```
wea\,ther,location=us-midwest temperature=82 1465839830100400200
```

* 空格：

```
wea\ ther,location=us-midwest temperature=82 1465839830100400200
```

字符串类型的field value，也要反斜杠\来转义。

* 双引号`"`：

```
weather,location=us-midwest temperature="too\"hot\"" 1465839830100400200
```

行协议不要求用户转义反斜杠字符\。所有其他特殊字符也不需要转义。例如，行协议处理emojis没有问题：

```
> INSERT we⛅️ther,location=us-midwest temper🔥ture=82 1465839830100400200
> SELECT * FROM "we⛅️ther"
name: we⛅️ther
------------------
time			              location	   temper🔥ture
1465839830100400200	 us-midwest	 82
```

### 关键字
行协议接受InfluxQL关键字作为标识符名称。一般来说，我们建议避免在schema中使用InfluxQL关键字，因为它可能会在查询数据时引起混淆。 

关键字`time`是特殊情况。`time`可以是cq的名称，数据库名称，measurement名称，RP名称，subscription名称和用户名 在这种情况下，查询`time`不需要双引号。`time`不能是field key或tag key; 当把`time`作为field key或是tag key写入时，InfluxDB会拒绝并返回错误。有关详细信息，请参阅[常见问题]()。

## 写数据到InfluxDB
### 写入数据的方法

现在你知道所有关于行协议的信息，你如何在使用中用行协议写入数据到InfluxDB呢？在这里，我们将给出两个快速示例，然后可以到[工具]()部分以获取更多信息。

#### HTTP API
使用HTTP API将数据写入InfluxDB。 向`/write`端点发送`POST`请求，并在请求主体中提供您的行协议：

```
curl -i -XPOST "http://localhost:8086/write?db=science_is_cool" --data-binary 'weather,location=us-midwest temperature=82 1465839830100400200'
```

有关查询字符串参数，状态码，响应和更多示例的深入描述，请参阅[API参考]()。

#### CLI
使用InfluxDB的命令行界面（CLI）将数据写入InfluxDB。启动CLI，使用相关数据库，并将`INSERT`放在行协议之前：

```
INSERT weather,location=us-midwest temperature=82 1465839830100400200
```

你还可以使用CLI从文件导入行协议。 

还有几种将数据写入InfluxDB的方式。有关HTTP API，CLI和可用的服务插件（UDP，Graphite，CollectD和OpenTSDB）的更多信息，请参阅[工具]()部分。

### 重复数据
一个点由measurement名称，tag set和timestamp唯一标识。如果您提交具有相同measurement，tag set和timestamp，但具有不同field set的行协议，则field set将变为旧field set与新field set的合并，并且如果有任何冲突以新field set为准。 

有关此行为的完整示例以及如何避免此问题，请参阅[常见问题]()。
