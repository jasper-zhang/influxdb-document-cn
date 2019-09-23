# 入门指南

InfluxDB安装完成之后，我们开始来做一些有意思的事。在这一章里面我们将会用到`influx`这个命令行工具，这个工具包含在InfluxDB的安装包里，是一个操作数据库的轻量级工具。它直接通过InfluxDB的HTTP接口(如果没有修改，默认是8086)来和InfluxDB通信。

>说明：也可以直接发送裸的HTTP请求来操作数据库，例如`curl`


## 创建数据库
如果你已经在本地安装运行了InfluxDB，你就可以直接使用`influx`命令行，执行`influx`连接到本地的InfluxDB实例上。输出就像下面这样：

```
$ influx -precision rfc3339
Connected to http://localhost:8086 version 1.2.x
InfluxDB shell 1.2.x
>
```

>说明:  
* InfluxDB的HTTP接口默认起在`8086`上，所以`influx`默认也是连的本地的`8086`端口，你可以通过`influx --help`来看怎么修改默认值。
* `-precision`参数表明了任何返回的时间戳的格式和精度，在上面的例子里，`rfc3339`是让InfluxDB返回[RFC339](https://www.ietf.org/rfc/rfc3339.txt)格式(YYYY-MM-DDTHH:MM:SS.nnnnnnnnnZ)的时间戳。

这样这个命令行已经准备好接收influx的查询语句了(简称InfluxQL)，用`exit`可以退出命令行。

第一次安装好InfluxDB之后是没有数据库的(除了系统自带的`_internal`)，因此创建一个数据库是我们首先要做的事，通过`CREATE DATABASE <db-name>`这样的InfluxQL语句来创建，其中`<db-name>`就是数据库的名字。数据库的名字可以是被双引号引起来的任意Unicode字符。 如果名称只包含ASCII字母，数字或下划线，并且不以数字开头，那么也可以不用引起来。

我们来创建一个`mydb`数据库：

```
> CREATE DATABASE mydb
>
```

>说明：在输入上面的语句之后，并没有看到任何信息，这在CLI里，表示语句被执行并且没有错误，如果有错误信息展示，那一定是哪里出问题了，这就是所谓的`没有消息就是好消息`。

现在数据库`mydb`已经创建好了，我们可以用`SHOW DATABASES`语句来看看已存在的数据库：

```
> SHOW DATABASES
name: databases
---------------
name
_internal
mydb

>
```

>说明：`_internal`数据库是用来存储InfluxDB内部的实时监控数据的。

不像`SHOW DATABASES`，大部分InfluxQL需要作用在一个特定的数据库上。你当然可以在每一个查询语句上带上你想查的数据库的名字，但是CLI提供了一个更为方便的方式`USE <db-name>`，这会为你后面的所以的请求设置到这个数据库上。例如：

```
> USE mydb
Using database mydb
>
```
以下的操作都作用于`mydb`这个数据库之上。

## 读写数据
现在我们已经有了一个数据库，那么InfluxDB就可以开始接收读写了。

首先对数据存储的格式来个入门介绍。InfluxDB里存储的数据被称为`时间序列数据`，其包含一个数值，就像CPU的load值或是温度值类似的。时序数据有零个或多个数据点，每一个都是一个指标值。数据点包括`time`(一个时间戳)，`measurement`(例如cpu_load)，至少一个k-v格式的`field`(也即指标的数值例如 “value=0.64”或者“temperature=21.2”)，零个或多个`tag`，其一般是对于这个指标值的元数据(例如“host=server01”, “region=EMEA”, “dc=Frankfurt)。

在概念上，你可以将`measurement`类比于SQL里面的table，其主键索引总是时间戳。`tag`和`field`是在table里的其他列，`tag`是被索引起来的，`field`没有。不同之处在于，在InfluxDB里，你可以有几百万的measurements，你不用事先定义数据的scheme，并且null值不会被存储。

将数据点写入InfluxDB，只需要遵守如下的行协议：

```
<measurement>[,<tag-key>=<tag-value>...] <field-key>=<field-value>[,<field2-key>=<field2-value>...] [unix-nano-timestamp]
```

下面是数据写入InfluxDB的格式示例：

```
cpu,host=serverA,region=us_west value=0.64
payment,device=mobile,product=Notepad,method=credit billed=33,licenses=3i 1434067467100293230
stock,symbol=AAPL bid=127.46,ask=127.48
temperature,machine=unit42,type=assembly external=25,internal=37 1434067467000000000
```

>说明：关于写入格式的更多语法，请参考[写入语法]()这一章。

使用CLI插入单条的时间序列数据到InfluxDB中，用`INSERT`后跟数据点：

```
> INSERT cpu,host=serverA,region=us_west value=0.64
>
```
这样一个measurement为`cpu`，tag是`host`和`region`，`value`值为`0.64`的数据点被写入了InfluxDB中。

现在我们查出写入的这笔数据：

```
> SELECT "host", "region", "value" FROM "cpu"
name: cpu
---------
time		    	                     host     	region   value
2015-10-21T19:28:07.580664347Z  serverA	  us_west	 0.64

>
```

>说明：我们在写入的时候没有包含时间戳，当没有带时间戳的时候，InfluxDB会自动添加本地的当前时间作为它的时间戳。

让我们来写入另一笔数据，它包含有两个字段：

```
> INSERT temperature,machine=unit42,type=assembly external=25,internal=37
>
```
查询的时候想要返回所有的字段和tag，可以用`*`：

```

> SELECT * FROM "temperature"
name: temperature
-----------------
time		                        	 external	  internal	 machine	type
2015-10-21T19:28:08.385013942Z  25	        	37     		unit42  assembly

>
```

InfluxQL还有很多特性和用法没有被提及，包括支持golang样式的正则，例如：

```
> SELECT * FROM /.*/ LIMIT 1
--
> SELECT * FROM "cpu_load_short"
--
> SELECT * FROM "cpu_load_short" WHERE "value" > 0.9
```

这就是你需要知道的读写InfluxDB的方法，想要学习更多的写的知识请移步[写数据]()章节，读的知识请到[读数据]()章节。要获取更多InfluxDB的概念的信息，请查看[关键概念]()章节。
