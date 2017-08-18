# 数据库管理

InfluxQL提供了一整套管理命令，包括数据管理和保留策略的管理。

下面的示例使用InfluxDB的命令行界面（CLI）。 您还可以使用HTTP API执行命令; 只需向`/query`发送GET请求，并将该命令包含在URL参数`q`中。

>注意：如果启用了身份验证，只有管理员可以执行本页上列出的大部分命令。

## 数据管理
### 创建数据库
#### 语法
```
CREATE DATABASE <database_name> [WITH [DURATION <duration>] [REPLICATION <n>] [SHARD DURATION <duration>] [NAME <retention-policy-name>]]
```

#### 语法描述
CREATE DATABASE需要一个数据库名称。 

`WITH`，`DURATION`，`REPLICATION`，`SHARD DURATION`和`NAME`子句是可选的用来创建与数据库相关联的单个保留策略。如果您没有在`WITH`之后指定其中一个子句，将默认为`autogen`保留策略。创建的保留策略将自动用作数据库的默认保留策略。

一个成功的`CREATE DATABASE`查询返回一个空的结果。如果您尝试创建已存在的数据库，InfluxDB什么都不做，也不会返回错误。

#### 例子
##### 例一：创建数据库
```
> CREATE DATABASE "NOAA_water_database"
>
```
该语句创建了一个叫做`NOAA_water_database`的数据库，默认InfluxDB也会创建`autogen`保留策略，并和数据库`NOAA_water_database`关联起来。

##### 例二：创建一个有特定保留策略的数据库
```
> CREATE DATABASE "NOAA_water_database" WITH DURATION 3d REPLICATION 1 SHARD DURATION 1h NAME "liquid"
>
```
该语句创建了一个叫做`NOAA_water_database`的数据库，并且创建了`liquid`作为数据库的默认保留策略，其持续时间为3天，副本数是1，shard group的持续时间为一个小时。

### 删除数据库
`DROP DATABASE`从指定数据库删除所有的数据，以及measurement，series，continuous queries, 和retention policies。语法为：

```
DROP DATABASE <database_name>
```

一个成功的`DROP DATABASE`查询返回一个空的结果。如果您尝试删除不存在的数据库，InfluxDB什么都不做，也不会返回错误。

### 用DROP从索引中删除series
`DROP SERIES`删除一个数据库里的一个series的所有数据，并且从索引中删除series。

>`DROP SERIES`不支持`WHERE`中带时间间隔。

该查询采用以下形式，您必须指定`FROM`子句或`WHERE`子句：

```
DROP SERIES FROM <measurement_name[,measurement_name]> WHERE <tag_key>='<tag_value>'
```

从单个measurement删除所有series：

```
> DROP SERIES FROM "h2o_feet"
```

从单个measurement删除指定tag的series：

```
> DROP SERIES FROM "h2o_feet" WHERE "location" = 'santa_monica'
```

从数据库删除有指定tag的所有measurement中的所有数据：

```
> DROP SERIES WHERE "location" = 'santa_monica'
```

### 用DELETE删除series
`DELETE`删除数据库中的measurement中的所有点。与`DROP SERIES`不同，它不会从索引中删除series，并且它支持`WHERE`子句中的时间间隔。 

该查询采用以下格式，必须包含`FROM`子句或`WHERE`子句，或两者都有：

```
DELETE FROM <measurement_name> WHERE [<tag_key>='<tag_value>'] | [<time interval>]
```

删除measurement`h2o_feet`的所有相关数据：

```
> DELETE FROM "h2o_feet"
```

删除measurement`h2o_quality`并且tag`randtag`等于3的所有数据：

```
> DELETE FROM "h2o_quality" WHERE "randtag" = '3'
```

删除数据库中2016年一月一号之前的所有数据：

```
> DELETE WHERE time < '2016-01-01'
```

一个成功的`DELETE`返回一个空的结果。 关于DELETE的注意事项：

* 当指定measurement名称时，`DELETE`在`FROM子`句中支持正则表达式，并在指定tag时支持`WHERE`子句中的正则表达式。
* `DELETE`不支持`WHERE`子句中的field。
* 如果你需要删除之后的数据点，则必须指定`DELETE SERIES`的时间间隔，因为其默认运行的时间为`time <now()`。

### 删除measurement
`DROP MEASUREMENT`删除指定measurement的所有数据和series，并且从索引中删除measurement。

该语法格式为：

```
DROP MEASUREMENT <measurement_name>
```

>注意：`DROP MEASUREMENT`删除measurement中的所有数据和series，但是不会删除相关的continuous queries。

*目前，InfluxDB不支持在`DROP MEASUREMENT `中使用正则表达式，具体在[#4275](https://github.com/influxdb/influxdb/issues/4275)中查看详情。*

### 删除shard
`DORP SHARD`删除一个shard，也会从metastore中删除shard。格式如下：

```
DROP SHARD <shard_id_number>
```

## 保留策略管理
以下部分介绍如何创建，更改和删除保留策略。 请注意，创建数据库时，InfluxDB会自动创建一个名为`autogen`的保留策略，该保留策略保留时间为无限。您可以重命名该保留策略或在配置文件中禁用其自动创建。

### 创建保留策略
#### 语法
```
CREATE RETENTION POLICY <retention_policy_name> ON <database_name> DURATION <duration> REPLICATION <n> [SHARD DURATION <duration>] [DEFAULT]
```

#### 语法描述
`DURATION`

`DURATION`子句确定InfluxDB保留数据的时间。 `<duration>`是持续时间字符串或INF（无限）。 保留策略的最短持续时间为1小时，最大持续时间为INF。

`REPLICATION`

`REPLICATION`子句确定每个点的多少独立副本存储在集群中，其中`n`是数据节点的数量。该子句不能用于单节点实例。

`SHARD DURATION`

`SHARD DURATION`子句确定shard group覆盖的时间范围。 `<duration>`是一个持续时间字符串，不支持INF（无限）持续时间。此设置是可选的。 默认情况下，shard group持续时间由保留策略的`DURATION`决定：

保留策略的持续时间|shard group的持续时间
--------|------
< 2天 | 1小时
>= 2天并<=6个月| 1天
> 6个月|7天

最小允许`SHARD GROUP DURATION`为1小时。 如果`CREATE RETENTION POLICY`查询尝试将`SHARD GROUP DURATION`设置为小于1小时且大于0，则InfluxDB会自动将`SHARD GROUP DURATION`设置为1h。 如果`CREATE RETENTION POLICY`查询尝试将`SHARD GROUP DURATION`设置为0，InfluxDB会根据上面列出的默认设置自动设置`SHARD GROUP DURATION`。

`DEFAULT`

将新的保留策略设置为数据库的默认保留策略。此设置是可选的。

#### 例子
##### 创建一个保留策略
```
> CREATE RETENTION POLICY "one_day_only" ON "NOAA_water_database" DURATION 1d REPLICATION 1
>
```

该语句给数据库`NOAA_water_database`创建一个保留策略`one_day_only`，持续时间为1天，副本数为1。

##### 创建一个默认的保留策略

```
> CREATE RETENTION POLICY "one_day_only" ON "NOAA_water_database" DURATION 23h60m REPLICATION 1 DEFAULT
>
```

该查询创建与上述示例中相同的保留策略，但将其设置为数据库的默认保留策略。 

成功的`CREATE RETENTION POLICY`执行返回为空。如果您尝试创建与已存在的保留策略相同的保留策略，InfluxDB不会返回错误。 如果您尝试创建与现有保留策略名称相同但具有不同属性的保留策略，InfluxDB会返回错误。

>注意：也可以在`CREATE DATABASE`时指定一个新的保留策略。

### 修改保留策略
`ALTER RETENTION POLICY`形式如下，你必须至少指定一个属性：`DURATION`, `REPLICATION`, `SHARD DURATION`,或者`DEFAULT`:

```
ALTER RETENTION POLICY <retention_policy_name> ON <database_name> DURATION <duration> REPLICATION <n> SHARD DURATION <duration> DEFAULT
```

现在我们来创建一个保留策略`what_is_time`其持续时间为两天：

```
> CREATE RETENTION POLICY "what_is_time" ON "NOAA_water_database" DURATION 2d REPLICATION 1
>
```

修改`what_is_time`的持续时间为3个星期，shard group的持续时间为30分钟，并将其作为数据库`NOAA_water_database`的默认保留策略：

```
> ALTER RETENTION POLICY "what_is_time" ON "NOAA_water_database" DURATION 3w SHARD DURATION 30m DEFAULT
>
```

在这个例子中，`what_is_time`将保留其原始副本数为1。

### 删除保留策略
删除指定保留策略的所有measurement和数据：

```
DROP RETENTION POLICY <retention_policy_name> ON <database_name>
```

成功的`DROP RETENTION POLICY`返回一个空的结果。如果您尝试删除不存在的保留策略，InfluxDB不会返回错误。

