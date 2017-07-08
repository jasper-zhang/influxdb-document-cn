# 专业术语

## aggregation
这是一个InfluxQL的函数，可以返回一堆数据的聚合结果，可以看[InfluxQL函数]()中现有的以及即将支持的聚合函数列表。

## batch
用换行符分割的数据点的集合，这批数据可以使用HTTP请求写到数据库中。用这种HTTP接口的方式可以大幅降低HTTP的负载。尽管不同的场景下更小或更大的batch可能有更好地性能，InfluxData建议每个batch的大小在5000~10000个数据点。

## continuous query（CQ）
这个一个在数据库中自动周期运行的InfluxQL的查询。Continuous query在`select`语句里需要一个函数，并且一定会包含一个`GROUP BY time()`的语法。

## database
对于users，retention policy，continuous query以及时序数据的一个逻辑上的集合。

## duration
retention policy中的一个属性，决定InfluxDB中数据保留多长时间。在duration之前的数据会自动从database中删除掉。

## field
InfluxDB数据中记录metadata和真实数据的键值对。fields在InfluxDB的数据结构中是必须的且不会被索引。如果要用field做查询条件的话，那就必须遍历所选时间范围里面的所有数据点，这种方式对比与tag效率会差很多。

## field key
组成field的键值对里面的键的部分。field key是字符串且保存在metadata中。

## field set
数据点上field key和field value的集合。

## field value
组成field的键值对里面的值的部分。field value才是真正的数据，可以是字符串，浮点数，整数，布尔型数据。一个field value总是和一个timestamp相关联。

field value不会被索引，如果要对field value做过滤话，那就必须遍历所选时间范围里面的所有数据点，这种方式对比与tag效率会差很多。

## function
包括InfluxQL中的聚合，查询和转换，可以在[InfluxQL函数]()中查看InfluxQL中的完整函数列表。

## identifier
涉及continuous query的名字，database名字，field keys，measurement名字，retention policy名字，subscription 名字，tag keys以及user 名字的一个标记。

## line protocol
写入InfluxDB时的数据点的文本格式。

## measurement
InfluxDB数据结果中的一部分，描述了存在关联field中的数据的意义，measurement是字符串。

## metastore
包含了系统状态的内部信息。metastore包含了用户信息，database，retention policy，shard metadata，continuous query以及subscription。

## node
一个独立的`influxd`进程。

## now()
本地服务器的当前纳秒级时间戳。

## point
InfluxDB数据结构的一部分由series中的的一堆field组成。 每个点由其series和timestamp唯一标识。 

你不能在同一series中存储多个具有相同timestamp的点。 相反，当你使用与该series中现有点相同的timestamp记将新point写入同一series时，该field set将成为旧field set和新field set的并集。

## query
从InfluxDB里面获取数据的一个操作

## replication factor
retention policy的一个参数，决定在集群模式下数据的副本的个数。InfluxDB在N个数据节点上复制数据，其中N就是replication factor。

>replication factor在单节点的实例上不起作用

## retention policy(RP)
InfluxDB数据结构的一部分，描述了InfluxDB保存数据的长短(duration)，数据存在集群里面的副本数(replication factor)，以及shard group的时间范围(shard group duration)。RPs在每个database里面是唯一的，连同measurement和tag set定义一个series。

当你创建一个database的时候，InfluxDB会自动创建一个叫做`autogen`的retention policy，其duration为永远，replication factor为1，shard group的duration设为的七天。

## schema
数据在InfluxDB里面怎么组织。InfluxDB的schema的基础是database，retention policy，series，measurement，tag key，tag value以及field keys。

## selector
一个InfluxQL的函数，从特定范围的数据点中返回一个点。可以看[InfluxQL函数]()中现有的以及即将支持的selector函数列表。

## series
InfluxDB数据结构的集合，一个特定的series由measurement，tag set和retention policy组成。

>注意：field set不是series的一部分

## series cardinality
在InfluxDB实例上唯一database，measurement和tag set组合的数量。

例如，假设一个InfluxDB实例有一个单独的database，一个measurement。这个measurement有两个tag key：`email`和`status`。如果有三个不同的email，并且每个email的地址关联两个不同的`status`，那么这个measurement的series cardinality就是6(3*2=6)：

email|status
-----|-----
lorr@influxdata.com|start
lorr@influxdata.com|finish
marv@influxdata.com|start
marv@influxdata.com|finish
cliff@influxdata.com|start
cliff@influxdata.com|finish

注意到，因为所依赖的tag的存在，在某些情况下，简单地执行该乘法可能会高估series cardinality。 依赖的tag是由另一个tag限定的tag并不增加series cardinality。 如果我们将tag`firstname`添加到上面的示例中，则系列基数不会是18（3 * 2 * 3 = 18）。 它将保持不变为6，因为`firstname`已经由`email`覆盖了：

email|status|firstname
-----|-----|-----
lorr@influxdata.com|start|lorraine
lorr@influxdata.com|finish|lorraine
marv@influxdata.com|start|marvin
marv@influxdata.com|finish|marvin
cliff@influxdata.com|start|clifford
cliff@influxdata.com|finish|clifford

在[常见问题]()中可以看到怎么根据series cardinality来查询InfluxDB。

## server
一个运行InfluxDB的服务器，可以使虚拟机也可以是物理机。每个server上应该只有一个InfluxDB的进程。

## shard
shard包含实际的编码和压缩数据，并由磁盘上的TSM文件表示。 每个shard都属于唯一的一个shard group。多个shard可能存在于单个shard group中。每个shard包含一组特定的series。给定shard group中的给定series上的所有点将存储在磁盘上的相同shard（TSM文件）中。

## shard duration
shard duration决定了每个shard group跨越多少时间。具体间隔由retention policy中的`SHARD DURATION`决定。 

例如，如果retention policy的`SHARD DURATION`设置为1w，则每个shard group将跨越一周，并包含时间戳在该周内的所有点。

## shard group
shard group是shard的逻辑组合。shard group由时间和retention policy组织。包含数据的每个retention policy至少包含一个关联的shard group。给定的shard group包含shard group覆盖的间隔的数据的所有shard。每个shard group跨越的间隔是shard的持续时间。

## subscription
subscription允许[Kapacitor](https://docs.influxdata.com/kapacitor/latest/)在push model中接收来自InfluxDB的数据，而不是基于查询数据的pull model。当Kapacitor配置为使用InfluxDB时，subscription将自动将订阅的数据库的每个写入从InfluxDB推送到Kapacitor。subscription可以使用TCP或UDP传输写入。

## tag
InfluxDB数据结构中的键值对，tags在InfluxDB的数据中是可选的，但是它们可用于存储常用的metadata; tags会被索引，因此tag上的查询是很高效的。

## tag key
组成tag的键值对中的键部分，tag key是字符串，存在metadata中。

## tag set
数据点上tag key和tag value的集合。

## tag value
组成tag的键值对中的值部分，tag value是字符串，存在metadata中。

## timestamp
数据点关联的日期和时间，在InfluxDB里的所有时间都是UTC的。

## transformation
一个InfluxQL的函数，返回一个值或是从特定数据点计算后的一组值。但是不是返回这些数据的聚合值。

## tsm(Time Structured Merge tree)
InfluxDB的专用数据存储格式。 TSM可以比现有的B+或LSM树实现更大的压缩和更高的写入和读取吞吐量。

## user
在InfluxDB里有两种类型的用户：

* admin用户对所有数据库都有读写权限，并且有管理查询和管理用户的全部权限。
* 非admin用户有针对database的可读，可写或者二者兼有的权限。

当认证开启之后，InfluxDB只执行使用有效的用户名和密码发送的HTTP请求。

## values per second
对数据持续到InfluxDB的速率的度量，写入速度通常以values per second表示。 

要计算每秒速率的值，将每秒写入的点数乘以每点存储的值数。 例如，如果这些点各有四个field，并且每秒写入batch是5000个点，那么values per second是每点4个fieldx每batch 5000个点x10个batch/秒=每秒200,000个值。

## wal(Write Ahead Log)
最近写的点数的临时缓存。为了减少访问永久存储文件的频率，InfluxDB将最新的数据点缓冲进WAL中，直到其总大小或时间触发然后flush到长久的存储空间。这样可以有效地将写入batch处理到TSM中。

可以查询WAL中的点，并且系统重启后仍然保留。在进程开始时，在系统接受新的写入之前，WAL中的所有点都必须flushed。

