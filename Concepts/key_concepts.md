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

