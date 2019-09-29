# 写入数据

有很多可以向InfluxDB写数据的方式，包括命令行、客户端还有一些像`Graphite`有一样数据格式的插件。这篇文章将会展示怎样创建数据库，并使用內建的HTTP接口写入数据。

## 使用HTTP接口创建数据库
使用`POST`方式发送到URL的`/query`路径，参数`q`为`CREATE DATABASE <new_database_name>`，下面的例子发送一个请求到本地运行的InfluxDB创建数据库`mydb`:

```
curl -i -XPOST http://localhost:8086/query --data-urlencode "q=CREATE DATABASE mydb"
```

## 使用HTTP接口写数据
通过HTTP接口`POST`数据到`/write`路径是我们往InfluxDB写数据的主要方式。下面的例子写了一条数据到`mydb`数据库。这条数据的组成部分是measurement为`cpu_load_short`，tag的key为host和region，对应tag的value是`server01`和`us-west`，field的key是`value`，对应的数值为`0.64`，而时间戳是`1434055562000000000`。

```
curl -i -XPOST 'http://localhost:8086/write?db=mydb' --data-binary 'cpu_load_short,host=server01,region=us-west value=0.64 1434055562000000000'
```

当写入这条数据点的时候，你必须明确存在一个数据库对应名字是`db`参数的值。如果你没有通过`rp`参数设置retention policy的话，那么这个数据会写到`db`默认的retention policy中。想要获取更多参数的完整信息，请移步到`API参考`章节。

POST的请求体我们称之为`Line Protocol`，包含了你希望存储的时间序列数据。它的组成部分有measurement，tags，fields和timestamp。measurement是InfluxDB必须的，严格地说，tags是可选的，但是对于大部分数据都会包含tags用来区分数据的来源，让查询变得容易和高效。tag的key和value都必须是字符串。fields的key也是必须的，而且是字符串，默认情况下field的value是float类型的。timestamp在这个请求行的最后，是一个从1/1/1970 UTC开始到现在的一个纳秒级的Unix time，它是可选的，如果不传，InfluxDB会使用服务器的本地的纳米级的timestamp来作为数据的时间戳，注意无论哪种方式，在InfluxDB中的timestamp只能是UTC时间。

### 同时写入多个点
要想同时发送多个数据点到多个series(在InfluxDB中measurement加tags组成了一个series)，可以用新的行来分开这些数据点。这种批量发送的方式可以获得更高的性能。

下面的例子就是写了三个数据点到`mydb`数据库中。第一个点所属series的measurement为`cpu_load_short`，tag是`host=server02`，timestamp是server本地的时间戳；第二个点同样是measurement为`cpu_load_short`，但是tag为`host=server02,region=us-west`,且有明确timestamp为`1422568543702900257`的series；第三个数据点和第二个的timestamp是一样的，但是series不一样，其measurement为`cpu_load_short`，tag为`direction=in,host=server01,region=us-west`。

```
curl -i -XPOST 'http://localhost:8086/write?db=mydb' --data-binary 'cpu_load_short,host=server02 value=0.67
cpu_load_short,host=server02,region=us-west value=0.55 1422568543702900257
cpu_load_short,direction=in,host=server01,region=us-west value=2.0 1422568543702900257'
```

### 写入文件中的数据
可以通过`curl`的`@filename`来写入文件中的数据，且这个文件里的数据的格式需要满足InfluxDB那种行的语法。

给一个正确的文件(cpu_data.txt)的例子：

```
cpu_load_short,host=server02 value=0.67
cpu_load_short,host=server02,region=us-west value=0.55 1422568543702900257
cpu_load_short,direction=in,host=server01,region=us-west value=2.0 1422568543702900257
```
看我们如何把`cpu_data.txt`里的数据写入`mydb`数据库：

```
curl -i -XPOST 'http://localhost:8086/write?db=mydb' --data-binary @cpu_data.txt
```

>说明：如果你的数据文件的数据点大于5000时，你必须把他们拆分到多个文件再写入InfluxDB。因为默认的HTTP的timeout的值为5秒，虽然5秒之后，InfluxDB仍然会试图把这批数据写进去，但是会有数据丢失的风险。

### 无模式设计
InfluxDB是一个无模式(schemaless)的数据库，你可以在任意时间添加measurement，tags和fields。注意：如果你试图写入一个和之前的类型不一样的数据(例如，filed字段之前接收的是数字类型，现在写了个字符串进去)，那么InfluxDB会拒绝这个数据。

### 对于REST的一个说明
InfluxDB使用HTTP作为方便和广泛支持的数据传输协议。

现代web的APIs都基于REST的设计，因为这样解决了一个共同的需求。因为随着终端数量的增长，组织系统的需求变得越来越迫切。REST是为了组织大量终端的一个业内认可的标准。这种一致性对于开发者和API的消费者都是一件好事：所有的参与者都知道期望的是什么。

REST的确是很方便的，而InfluxDB也只提供了三个API，这使得InfluxQL在翻译为HTTP请求的时候很简单便捷。 所以InfluxDB API并不是RESTful的。

### HTTP返回值概要
* 2xx：如果你写了数据后收到`HTTP 204 No Content`，说明写入成功了！
* 4xx：表示InfluxDB不知道你发的是什么鬼。
* 5xx：系统过载或是应用受损。

举几个返回错误的例子：

* 之前接收的是布尔值，现在你写入一个浮点值：

```
curl -i -XPOST 'http://localhost:8086/write?db=hamlet' --data-binary 'tobeornottobe booleanonly=true'  

curl -i -XPOST 'http://localhost:8086/write?db=hamlet' --data-binary 'tobeornottobe booleanonly=5'
```
这时系统返回：

```
HTTP/1.1 400 Bad Request
Content-Type: application/json
Request-Id: [...]
X-Influxdb-Version: 1.2.x
Date: Wed, 01 Mar 2017 19:38:01 GMT
Content-Length: 150

{"error":"field type conflict: input field \"booleanonly\" on measurement \"tobeornottobe\" is type float, already exists as type boolean dropped=1"}
```

* 写入数据到一个不存在的数据库中：

```
curl -i -XPOST 'http://localhost:8086/write?db=atlantis' --data-binary 'liters value=10'
```

返回值：

```
HTTP/1.1 404 Not Found
Content-Type: application/json
Request-Id: [...]
X-Influxdb-Version: 1.2.x
Date: Wed, 01 Mar 2017 19:38:35 GMT
Content-Length: 45

{"error":"database not found: \"atlantis\""}
```

### 下一步
现在你已经知道了如何通过内置HTTP API写入数据了，下面我们来看看怎么通过[读数据](querying_data.md)指南来读出它们。想要了解更多关于如何使用HTTP API写数据，请参考[API参考文档]()。
