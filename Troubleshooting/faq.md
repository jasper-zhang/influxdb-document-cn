# FAQ
## 管理
### 如何在密码中包含单引号？
在创建密码和发送身份验证请求时使用反斜杠（\）来转义单引号。
### 如何查看InfluxDB的版本？
有几种方法可以查看InfluxDB的版本：
#### 在终端运行`influxd version`:
```
$ influxd version

InfluxDB ✨ v1.3.0 ✨ (git: master b7bb7e8359642b6e071735b50ae41f5eb343fd42)
```
#### `curl`路径`/ping`：
```
$ curl -i 'http://localhost:8086/ping'

HTTP/1.1 204 No Content
Content-Type: application/json
Request-Id: 1e08aeb6-fec0-11e6-8486-000000000000
✨ X-Influxdb-Version: 1.3.x ✨
Date: Wed, 01 Mar 2017 20:46:17 GMT
```
#### 运行InfluxDB的命令行：
```
$ influx

Connected to http://localhost:8086✨ version 1.3.x ✨  
InfluxDB shell version: 1.3.x
```
#### 在日志里查看HTTP返回结果：
```
$ journald-ctl -u influxdb.service

Mar 01 20:49:45 rk-api influxd[29560]: [httpd] 127.0.0.1 - - [01/Mar/2017:20:49:45 +0000] "POST /query?db=&epoch=ns&q=SHOW+DATABASES HTTP/1.1" 200 151 "-" ✨ "InfluxDBShell/1.3.x" ✨ 9a4371a1-fec0-11e6-84b6-000000000000 1709
```

### 怎样查看InfluxDB的日志？
在System V操作系统上，日志存储在/var/log/influxdb/下。   
在systemd操作系统上，可以使用`journalctl`访问日志。 使用`journalctl -u influxdb`查看日志或`journalctl -u influxdb> influxd.log`将日志打印到文本文件。使用systemd时，日志保留取决于系统的日记设置。

### 分片组的保留时间和保留策略之间的关系？
nfluxDB将数据存储在分片组中。一个分片组覆盖特定的时间间隔; InfluxDB通过查看相关保留策略（RP）的`DURATION`来确定时间间隔。下表列出了RP的`DURATION`和分片组的时间间隔之间的默认关系：

RP持续时间|分片组间隔
---- | ---
<2天|1小时
>= 2天，<= 6个月|1天
> 6个月|7天

用户还可以使用`CREATE RETENTION POLICY`和`ALTER RETENTION POLICY`语句配置分片组持续时间。使用`SHOW RETENTION POLICY`语句检查保留策略的分片组持续时间。

### 为什么在修改了RP之后数据没有被删除？
有几个因素可以解释为什么保留策略（RP）更改后数据可能不会立即丢失。

第一个也是最可能的原因是，默认情况下，InfluxDB每30分钟检查一次强制RP。可能需要等待下一次RP检查InfluxDB才能删除RP的新`DURATION`设置之外的数据。30分钟的间隔是可配置的。

其次，改变RP的`DURATION`和`SHARD DURATION`可能会导致意外的数据保留。InfluxDB将数据存储在包含特定RP和时间间隔的分片组中。当InfluxDB执行一个RP时，它会删除整个分片组，而不是单个数据点。InfluxDB不能拆分分片组。

如果RP的新`DURATION`小于旧的`SHARD DURATION`，并且InfluxDB正在将数据写入其中一个较旧的分片组，则系统将被迫将所有数据保留在该分片组中。即使该分片组中的某些数据不在新的`DURATION`中，也会发生这种情况。一旦所有的数据都在新的`DURATION`之外，InfluxDB将删除该分片组。系统将开始将数据写入新的分片组，这些分片组具有新的更短的`SHARD DURATION`，以防止写入不被期望的数据。

### 为什么InfluxDB无法在配置文件中解析微秒单位？
用于指定微秒持续时间单位的语法在配置设置，和写入，查询以及在InfluxDB命令行界面（CLI）中设置精度方面有所不同。 下表显示了每个类别支持的语法：

 单位|配置文件|HTTP API写入|所有的查询|CLI精度命令行
---- | ---|----|----|----
u	|❌|	👍|	👍|	👍
us|	👍|	❌|	❌|	❌
µ	|❌|	❌|	👍|	❌
µs	|👍|	❌|	❌|	❌

如果配置选项指定u或μ语法，InfluxDB将无法启动并在日志中报告以下错误：

```
run: parse config: time: unknown unit [µ|u] in duration [<integer>µ|<integer>u]
```

## 命令行
### 怎么让InfluxDB的CLI返回用户可读的时间戳?
当你第一次连CLI，可以指定rfc3339精度：

```
$ influx -precision rfc3339
```

此外，如果你已经连接了CLI，则可以通过指令来指定：

```
$ influx
Connected to http://localhost:8086 version 0.xx.x
InfluxDB shell 0.xx.x
> precision rfc3339
>
```

### 一个非admin用户怎么在InfluxDB的CLI下USE一个数据库？
在v1.3之前的版本中，非admin用户是不能在CLI执行`USE <database_name>`的查询的，即使拥有这个数据库的`READ`或/和`Write`权限。

从1.3开始，非admin用户也可以执行`USE <database_name>`的查询只要拥有这个数据库的`READ`或/和`Write`权限。如果非admin使用`USE`一个数据库，但是没有权限时，系统会返回一个错误：

```
ERR: Database <database_name> doesn't exist. Run SHOW DATABASES for a list of existing databases.
```

>注意：非admin用户执行`SHOW DATABASES`，只返回有权限的数据库。

### 在InfluxDB的CLI下，如何写入一个非默认的retention policy？
使用语法`INSERT INTO [<database>.]<retention_policy> <line_protocol>`可以在CLI下把数据写入非`DEFAULT`的retention policy里面。（如果使用HTTP来写入数据的话，可以通过参数`db`和`rp`来指定数据库和retention policy）。

例如：

```
> INSERT INTO one_day mortality bool=true
Using retention policy one_day
> SELECT * FROM "mydb"."one_day"."mortality"
name: mortality
---------------
time                             bool
2016-09-13T22:29:43.229530864Z   true
```

注意如果要查询出非`DEFAULT`的retention policy，需要指定完整的measurement路径：

```
"<database>"."<retention_policy>"."<measurement>"
```

### 怎么取消一个长期运行的查询？
在CLI下，你可以用`Ctrl+C`来取消执行的查询。对于其他的长时间运行的查询，你可以先使用`SHOW QUERIES`列出来，然后使用`KILL QUERY`来停止对应的查询。

### 为什么不能查询布尔型的field value？
对于写和读，接受布尔型的数据的语法不一样。

 布尔语法|写|读
---- | ---|----
t,f	|👍|	❌
T,F|	👍|	❌
true,false	|👍|	👍
True,False	|👍|	👍
TRUE,FALSE	|👍|	👍

例如，`SELECT * FROM "hamlet" WHERE "bool"=True`返回所有`bool`的值为`TRUE`的，但是`SELECT * FROM "hamlet" WHERE "bool"=T`什么都不会返回。
[Github Issue #3939](https://github.com/influxdb/influxdb/issues/3939)

### InfluxDB怎么处理不同shard里面的字段类型冲突？
字段的值类型可以是浮点，整数，字符串和布尔型。字段的类型在同一个shard里面是一致的，但是在不同的shard里面可以不一样。

#### SELECT语句
如果所有值都具有相同的类型，`SELECT`语句会返回所有字段值。如果字段的值类型在不同的shard里面不一样，InfluxDB首先执行正常操作并将所有值返回为出现在下面的列表的第一个类型：浮点，整数，字符串和布尔型。

如果你的数据字段的值类型不符，使用语法`<field_key>::<type>`查询不同的数据类型。

##### 例子
measurement`just_my_type`有一个字段叫作`my_field`。`my_field`在四个不同shard里面有四个不同的类型分别为（浮点，整数，字符串和布尔型）。

`SELECT*`只返回浮点数和整数字段值。注意InfluxDB在返回时把整数转化为了浮点类型。

```
SELECT * FROM just_my_type

name: just_my_type
------------------
time		                	my_field
2016-06-03T15:45:00Z	  9.87034
2016-06-03T16:45:00Z	  7
```

`SELECT<field_key>::<type>[…]`返回所有值类型。我们可以在InfluxDB列名里增加自己的列输出的值类型。在可能的情况下，InfluxDB字段值会转到另一个类型；它把整数7在第一列中转化为了浮点数，而且把9.879034在第二列中转化为了整数。InfluxDB不能把浮点或整数转化为字符串或布尔值。

```golang
SELECT "my_field"::float,"my_field"::integer,"my_field"::string,"my_field"::boolean FROM just_my_type

name: just_my_type
------------------
time			               my_field	 my_field_1	 my_field_2		 my_field_3
2016-06-03T15:45:00Z	 9.87034	  9
2016-06-03T16:45:00Z	 7	        7
2016-06-03T17:45:00Z			                     a string
2016-06-03T18:45:00Z					                                true
```

#### SHOW FIELD KEYS查询
`SHOW FIELD KEYS`会返回相关field在不同的shard里面的每种类型。

measurement`just_my_type`有一个字段叫作`my_field`。`my_field`在四个不同shard里面有四个不同的类型分别为（浮点，整数，字符串和布尔型）。`SHOW FIELD KEYS`返回所有四种数据类型：

```
> SHOW FIELD KEYS

name: just_my_type
fieldKey   fieldType
--------   ---------
my_field   float
my_field   string
my_field   integer
my_field   boolean
```

### InfluxDB可以存储的最大最小整数是什么？




