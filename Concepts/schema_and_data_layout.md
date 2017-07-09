# schema设计

每个InfluxDB用例都可能是不一样的，schema将反映出这种独特性。 但是，在设计schema时，有一些遵循的一般准和可以避免的陷阱。

## 一般建议

### 推崇的schema设计

没有特定的顺序，我们有如下建议：

#### 哪些情况下使用tag
一般来说，你的查询可以指引你哪些数据放在tag中，哪些放在field中。

* 把你经常查询的字段作为tag
* 如果你要对其使用`GROUP BY()`，也要放在tag中
* 如果你要对其使用InfluxQL函数，则将其放到field中
* 如果你需要存储的值不是字符串，则需要放到field中，因为tag value只能是字符串

#### 避免InfluxQL中关键字作为标识符名称
这不是必需的，但它简化了写查询; 您不必将这些标识符包装在双引号中。 标识符有database名称，retention policy名称，user名，measurement名称，tag key和field key。请参阅[InfluxQL关键词]()看下哪些单词需要被避免。

请注意，如果查询中包含除[A-z，_]以外的字符，则还需要将它们用双引号括起来。

### 避免的schema设计

没有特定的顺序，我们有如下建议：

#### 不要有太多的series
tags包含高度可变的信息，如UUID，哈希值和随机字符串，这将导致数据库中的大量measurement，通俗地说是高series cardinality。series cardinality高是许多数据库高内存使用的主要原因。

请参阅[硬件指南](/Guide/hardware_sizing.md)中基于你的硬件的series cardinality的建议。如果系统有内存限制，请考虑将高cardinality数据存储为field而不是tag。

#### 如何设计measurement
一般来说，谈论这一步可以简化你的查询。InfluxDB的查询会合并属于同一measurement范围内的数据; 用tag区分数据比使用详细的measurement名字更好。

例如：考虑如下的schema：

```
Schema 1 - Data encoded in the measurement name
-------------
blueberries.plot-1.north temp=50.1 1472515200000000000
blueberries.plot-2.midwest temp=49.8 1472515200000000000
```

这个没有tag的长长的measurement名字(`blueberries.plot-1.north`)有些类似于Graphite的metric。像`plot``region`这样的信息放在measurement名字里面将会使数据很难去查询。

例如，使用schema 1计算两个图1和2的平均`temp`是不可能的。将其与如下schema进行比较：

```
Schema 2 - Data encoded in tags
-------------
weather_sensor,crop=blueberries,plot=1,region=north temp=50.1 1472515200000000000
weather_sensor,crop=blueberries,plot=2,region=midwest temp=49.8 1472515200000000000
```

以下查询计算了落在北部地区的蓝莓的平均`temp`。 虽然这两个查询都比较简单，但使用正则表达式使得某些查询更加复杂或根本不可能实现。

```
# Schema 1 - Query for data encoded in the measurement name
> SELECT mean("temp") FROM /\.north$/

# Schema 2 - Query for data encoded in tags
> SELECT mean("temp") FROM "weather_sensor" WHERE "region" = 'north'
```

#### 不要把多条信息放到一个tag里面
与上述相似，将具有多条信息的单个tag拆分为多个单独的tag将简化查询并减少对正则表达式的需求。

例如，考虑如下的schema：

```
Schema 1 - Multiple data encoded in a single tag
-------------
weather_sensor,crop=blueberries,location=plot-1.north temp=50.1 1472515200000000000
weather_sensor,crop=blueberries,location=plot-2.midwest temp=49.8 1472515200000000000
```

上述数据将多个单独的参数`plot``region`放到了一个长tag value里面（`plot-1.north`）。 将其与如下schema进行比较：

```
Schema 2 - Data encoded in multiple tags
-------------
weather_sensor,crop=blueberries,plot=1,region=north temp=50.1 1472515200000000000
weather_sensor,crop=blueberries,plot=2,region=midwest temp=49.8 1472515200000000000
```

以下查询计算了落在`north`地区的蓝莓的平均`temp`。 虽然这两个查询都是相似的，但在Schema 2中使用多个tag避免了使用正则表达式。

```
# Schema 1 - Query for multiple data encoded in a single tag
> SELECT mean("temp") FROM "weather_sensor" WHERE location =~ /\.north$/

# Schema 2 - Query for data encoded in multiple tags
> SELECT mean("temp") FROM "weather_sensor" WHERE region = 'north'
```

## shard group的保留时间(duration)的管理
### shard group的保留时间(duration)预览
InfluxDB将数据存储在shard group中。shard group由存储策略（RP）管理，并存储具有特定时间间隔内的时间戳的数据。 该时间间隔的长度称为shard group的duration。

默认情况下，shard group的duration是由RP的duration决定：

RP duration|shard group duration
---------|-------
< 2 days | a hour
>= 2 days and <= 6 months | 1 day
> 6 months | 7 days

在每个RP上shard group的duration也是可以配置的，可以看[Retention Policy管理]()看如何配置shard group的duration。

### shard group的duration的建议
通常，较短的shard group的duration允许系统有效地丢弃数据。 当InfluxDB强制执行RP时，它会丢弃整个shard group，而不是单个数据点。 例如，如果您的RP有一天的持续时间，一个shard group持续时间为一小时; 则InfluxDB每小时将丢弃一小时的数据。

如果您的RP的持续时间大于6个月，则不需要具有较短的shard group持续时间。 事实上，将shard group的持续时间提高到默认的七天值以上可以改善压缩，提高写入速度，并减少每个shard group的固定迭代器开销。 例如，50年及以上的shard group持续时间是可接受的配置。

>说明：当配置shard group的duration的时候，`INF`(infinite)是一个不合理的配置。在实际工作中，特定的duration`1000w`就足够达到非常长的shard group持续时间。

我们建议shard group的配置如下：

* 是你一般查询时间范围的两倍
* 每个shard group里至少有100000个数据点
* 每个shard group里面的每个series至少有1000个数据点