# 关键概念

在深入InfluxDB之前，最好是了解数据库的一些关键概念。 本文档简要介绍了这些概念和常用的InfluxDB术语。 我们在下面列出了所有涵盖的术语，但是我们建议您从头到尾阅读本文档，以获得对我们最喜爱的时间序列数据库的更全面了解。

|**database**|**field key**|**field set**|
|:--:| :--:|:--: |
|**field value**|**measurement**|**point**|
|**retention policy**|**series**|**tag key**|
|**tag set**|**tag value**| **timestamp**|

## 示例数据 
下一节将参考下面列出的数据。 虽然数据是伪造的，但在InfluxDB中是一个很通用的场景。 数据展示了在2015年8月18日午夜至2015年8月18日上午6时12分在两个地点`location`（地点`1`和地点`2`）显示两名科学家`scientists`（`langstroth`和`perpetua`）计数的蝴蝶(`butterflies`)和蜜蜂(`honeybees`)数量。 假设数据存在名为`my_database`的数据库中，而且存储策略是`autogen`。

```
name: census
-————————————
time                                      butterflies     honeybees     location     scientist
2015-08-18T00:00:00Z      12                   23                    1                 langstroth
2015-08-18T00:00:00Z      1                     30                    1                 perpetua
2015-08-18T00:06:00Z      11                   28                    1                 langstroth
2015-08-18T00:06:00Z   3                     28                    1                 perpetua
2015-08-18T05:54:00Z      2                     11                    2                 langstroth
2015-08-18T06:00:00Z      1                     10                    2                 langstroth
2015-08-18T06:06:00Z      8                     23                    2                 perpetua
2015-08-18T06:12:00Z      7                     22                    2                 perpetua
```

其中census是`measurement`，butterflies和honeybees是`field key`，location和scientist是`tag key`。

## 讨论
现在您已经在InfluxDB中看到了一些示例数据，本节将详细分析这些数据。

InfluxDB是一个时间序列数据库，因此我们开始一切的根源就是——时间。在上面的数据中有一列是`time`，在InfluxDB中所有的数据都有这一列。`time`存着时间戳，这个时间戳以[RFC3339](https://www.ietf.org/rfc/rfc3339.txt)格式展示了与特定数据相关联的UTC日期和时间。

接下来两个列叫作`butterflies`和`honeybees`，称为fields。fields由field key和field value组成。field key(`butterflies`和`honeybees`)都是字符串，他们存储元数据；field key `butterflies`告诉我们蝴蝶的计数从12到7；field key `honeybees`告诉我们蜜蜂的计数从23变到22。

field value就是你的数据，它们可以是字符串、浮点数、整数、布尔值，因为InfluxDB是时间序列数据库，所以field value总是和时间戳相关联。

在示例中，field value如下：

```
12   23
1    30
11   28
3    28
2    11
1    10
8    23
7    22
```

在上面的数据中，每组field key和field value的集合组成了`field set`，在示例数据中，有八个`field set`：

```
butterflies = 12 honeybees = 23
butterflies = 1 honeybees = 30
butterflies = 11 honeybees = 28
butterflies = 3 honeybees = 28
butterflies = 2 honeybees = 11
butterflies = 1 honeybees = 10
butterflies = 8 honeybees = 23
butterflies = 7 honeybees = 22
```

field是InfluxDB数据结构所必需的一部分——在InfluxDB中不能没有field。还要注意，field是没有索引的。如果使用field value作为过滤条件来查询，则必须扫描其他条件匹配后的所有值。因此，这些查询相对于tag上的查询（下文会介绍tag的查询）性能会低很多。 一般来说，字段不应包含常用来查询的元数据。

样本数据中的最后两列（`location`和`scientist`）就是tag。 tag由tag key和tag value组成。tag key和tag value都作为字符串存储，并记录在元数据中。示例数据中的tag key是`location`和`scientist`。 `location`有两个tag value：`1`和`2`。`scientist`还有两个tag value：`langstroth`和`perpetua`。

在上面的数据中，tag set是不同的每组tag key和tag value的集合，示例数据里有四个tag set：

```
location = 1, scientist = langstroth
location = 2, scientist = langstroth
location = 1, scientist = perpetua
location = 2, scientist = perpetua
```

tag不是必需的字段，但是在你的数据中使用tag总是大有裨益，因为不同于field， tag是索引起来的。这意味着对tag的查询更快，tag是存储常用元数据的最佳选择。

>#### 不同场景下的数据结构设计
>如果你说你的大部分的查询集中在字段`honeybees`和`butterflies`上：
>
```
SELECT * FROM "census" WHERE "butterflies" = 1
SELECT * FROM "census" WHERE "honeybees" = 23
>```
>因为field是没有索引的，在第一个查询里面InfluxDB会扫描所有的`butterflies`的值，第二个查询会扫描所有`honeybees`的值。这样会使请求时间很长，特别在规模很大时。为了优化你的查询，你应该重新设计你的数据结果，把field(`butterflies`和`honeybees`)改为tag，而将tag（`location`和`scientist`）改为field。
>
```
name: census
-————————————
time                                      location     scientist      butterflies     honeybees
2015-08-18T00:00:00Z      1                 langstroth    12                   23
2015-08-18T00:00:00Z      1                 perpetua      1                     30
2015-08-18T00:06:00Z      1                 langstroth    11                   28
2015-08-18T00:06:00Z   1                 perpetua      3                     28
2015-08-18T05:54:00Z      2                 langstroth    2                     11
2015-08-18T06:00:00Z      2                 langstroth    1                     10
2015-08-18T06:06:00Z      2                 perpetua      8                     23
2015-08-18T06:12:00Z      2                 perpetua      7                     22
```
>现在`butterflies`和`honeybees`是tag了，当你再用上面的查询语句时，就不会扫描所有的值了，这也意味着查询更快了。

measurement作为tag，fields和time列的容器，measurement的名字是存储在相关fields数据的描述。 measurement的名字是字符串，对于一些SQL用户，measurement在概念上类似于表。样本数据中唯一的测量是`census`。 名称`census`告诉我们，fields值记录了`butterflies`和`honeybees`的数量，而不是不是它们的大小，方向或某种幸福指数。 

单个measurement可以有不同的retention policy。 retention policy描述了InfluxDB保存数据的时间（DURATION）以及这些存储在集群中数据的副本数量（REPLICATION）。 如果您有兴趣阅读有关retention policy的更多信息，请查看[数据库管理]()章节。

>注意：在单节点的实例下，Replication系数不管用。

在样本数据中，measurement `census`中的所有内容都属于`autogen`的retention policy。 InfluxDB自动创建该存储策略; 它具有无限的持续时间和复制因子设置为1。

现在你已经熟悉了measurement，tag set和retention policy，那么现在是讨论series的时候了。 在InfluxDB中，series是共同retention policy，measurement和tag set的集合。 以上数据由四个series组成：

任意series编号|retention policy|measurement|tag set
------|------|-----|----
series 1| `autogen`|`census`|`location = 1,scientist = langstroth`
series 2| `autogen`|`census`|`location = 2,scientist = langstroth`
series 3| `autogen`|`census`|`location = 1,scientist = perpetua`
series 4| `autogen`|`census`|`location = 2,scientist = perpetua`

理解series对于设计数据schema以及对于处理InfluxDB里面的数据都是很有必要的。

最后，point就是具有相同timestamp的相同series的field集合。例如，这就是一个point：

```
name: census
-----------------
time			               butterflies	 honeybees	 location	 scientist
2015-08-18T00:00:00Z	 1		          30		       1		       perpetua
```

例子里的series的retention policy为`autogen`，measurement为`census`，tag set为`location = 1, scientist = perpetua`。point的timestamp为`2015-08-18T00:00:00Z`。

我们刚刚涵盖的所有内容都存储在数据库（database）中——示例数据位于数据库`my_database`中。 InfluxDB数据库与传统的关系数据库类似，并作为users，retention policy，continuous以及point的逻辑上的容器。 有关这些主题的更多信息，请参阅[身份验证和授权]()和[连续查询(continuous query)]()。

数据库可以有多个users，retention policy，continuous和measurement。 InfluxDB是一个无模式数据库，意味着可以随时添加新的measurement，tag和field。 它旨在使时间序列数据的工作变得非常棒。

你做到了！你已经知道了InfluxDB中的基本概念和术语。如果你是初学者，我们建议您查看[入门指南](Introduction/getting_start.md)和[写入数据](Guide/writing_data.md)和[查询数据](Guide/querying_data.md)指南。 愿我们的时间序列数据库可以为您服务🕔。

