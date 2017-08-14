# schema查询语法
InfluxQL是一种类似SQL的查询语言，用于与InfluxDB中的数据进行交互。下面我们要介绍一些有用的查询schema的语法：

* SHOW DATABASES
* SHOW RETENTION POLICIES
* SHOW SERIES
* SHOW MEASUREMENTS
* SHOW TAG KEYS
* SHOW TAG VALUES
* SHOW FIELD KEYS

在开始之前，默认已经登入了CLI：

```
$ influx -precision rfc3339 
Connected to http://localhost:8086 version 1.3.x
InfluxDB shell 1.3.x
>
```

## SHOW DATABASES
返回当前实例上的所有的数据库。
### 语法
```
SHOW DATABASES
```
### 例子
#### 例一：运行`SHOW DATABASES`查询
```
> SHOW DATABASES

name: databases
name
----
NOAA_water_database
_internal
```

## SHOW RETENTION POLICIES
返回指定数据库的保留策略的列表。
### 语法
```
SHOW RETENTION POLICIES [ON <database_name>]
```
### 语法描述
`ON <database_name>`是可选的。如果查询不包括`ON <database_name>`，则必须在CLI中使用`USE <database_name>`指定数据库，或者在HTTP API请求中指定`db`查询字符串参数。

### 例子
#### 例一：运行带有`ON`子句的`SHOW RETENTION POLICIES`
```
> SHOW RETENTION POLICIES ON NOAA_water_database

name      duration   shardGroupDuration   replicaN   default
----      --------   ------------------   --------   -------
autogen   0s         168h0m0s             1          true
```

该查询以表格格式返回数据库`NOAA_water_database`中的保留策略列表。 该数据库有一个名为`autogen`的保留策略。该保留策略具有无限持续时间，持续时间七天的shard group，副本数为1，并且是数据库的`DEFAULT`保留策略。

#### 例二：运行不带`ON`子句的`SHOW RETENTION POLICIES`
##### CLI
使用`USE <database_name>`指定数据库：

```
> USE NOAA_water_database
Using database NOAA_water_database

> SHOW RETENTION POLICIES

name      duration   shardGroupDuration   replicaN   default
----      --------   ------------------   --------   -------
autogen   0s         168h0m0s             1          true
```

##### HTTP API
用`db`参数指定数据库：

```
~# curl -G "http://localhost:8086/query?db=NOAA_water_database&pretty=true" --data-urlencode "q=SHOW RETENTION POLICIES"

{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "columns": [
                        "name",
                        "duration",
                        "shardGroupDuration",
                        "replicaN",
                        "default"
                    ],
                    "values": [
                        [
                            "autogen",
                            "0s",
                            "168h0m0s",
                            1,
                            true
                        ]
                    ]
                }
            ]
        }
    ]
}
```

## SHOW SERIES
返回指定数据库的series列表。
### 语法
```
SHOW SERIES [ON <database_name>] [FROM_clause] [WHERE <tag_key> <operator> [ '<tag_value>' | <regular_expression>]] [LIMIT_clause] [OFFSET_clause]
```

### 语法描述
`ON <database_name>`是可选的。如果查询不包括`ON <database_name>`，则必须在CLI中使用`USE <database_name>`指定数据库，或者在HTTP API请求中指定`db`查询字符串参数。

`FROM`，`WHERE`，`LIMIT`和`OFFSET`子句是可选的。 `WHERE`子句支持tag比较; field比较对`SHOW SERIES`查询无效。

`WHERE`子句中支持的运算符：

`=` 等于  
`<>` 不等于  
`!=` 不等于  
`=~` 匹配  
`!~` 不匹配

### 例子
#### 例一：运行带`ON`子句的`SHOW SERIES`

```
> SHOW SERIES ON NOAA_water_database

key
---
average_temperature,location=coyote_creek
average_temperature,location=santa_monica
h2o_feet,location=coyote_creek
h2o_feet,location=santa_monica
h2o_pH,location=coyote_creek
h2o_pH,location=santa_monica
h2o_quality,location=coyote_creek,randtag=1
h2o_quality,location=coyote_creek,randtag=2
h2o_quality,location=coyote_creek,randtag=3
h2o_quality,location=santa_monica,randtag=1
h2o_quality,location=santa_monica,randtag=2
h2o_quality,location=santa_monica,randtag=3
h2o_temperature,location=coyote_creek
h2o_temperature,location=santa_monica
```

查询的输出类似于行协议格式。第一个逗号之前的所有内容都是measurement名称。第一个逗号后的所有内容都是tag key或tag value。 `NOAA_water_database`有五个不同的measurement和14个不同的series。

#### 例二：运行不带`ON`子句的`SHOW SERIES`

##### CLI 
用`USE <database_name>`指定数据库：

```
> USE NOAA_water_database
Using database NOAA_water_database

> SHOW SERIES

key
---
average_temperature,location=coyote_creek
average_temperature,location=santa_monica
h2o_feet,location=coyote_creek
h2o_feet,location=santa_monica
h2o_pH,location=coyote_creek
h2o_pH,location=santa_monica
h2o_quality,location=coyote_creek,randtag=1
h2o_quality,location=coyote_creek,randtag=2
h2o_quality,location=coyote_creek,randtag=3
h2o_quality,location=santa_monica,randtag=1
h2o_quality,location=santa_monica,randtag=2
h2o_quality,location=santa_monica,randtag=3
h2o_temperature,location=coyote_creek
h2o_temperature,location=santa_monica
```

##### HTTP API
用`db`参数指定数据库：

```
~# curl -G "http://localhost:8086/query?db=NOAA_water_database&pretty=true" --data-urlencode "q=SHOW SERIES"

{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "columns": [
                        "key"
                    ],
                    "values": [
                        [
                            "average_temperature,location=coyote_creek"
                        ],
                        [
                            "average_temperature,location=santa_monica"
                        ],
                        [
                            "h2o_feet,location=coyote_creek"
                        ],
                        [
                            "h2o_feet,location=santa_monica"
                        ],
                        [
                            "h2o_pH,location=coyote_creek"
                        ],
                        [
                            "h2o_pH,location=santa_monica"
                        ],
                        [
                            "h2o_quality,location=coyote_creek,randtag=1"
                        ],
                        [
                            "h2o_quality,location=coyote_creek,randtag=2"
                        ],
                        [
                            "h2o_quality,location=coyote_creek,randtag=3"
                        ],
                        [
                            "h2o_quality,location=santa_monica,randtag=1"
                        ],
                        [
                            "h2o_quality,location=santa_monica,randtag=2"
                        ],
                        [
                            "h2o_quality,location=santa_monica,randtag=3"
                        ],
                        [
                            "h2o_temperature,location=coyote_creek"
                        ],
                        [
                            "h2o_temperature,location=santa_monica"
                        ]
                    ]
                }
            ]
        }
    ]
}
```

#### 例三：运行带有多个子句的`SHOW SERIES`

```
> SHOW SERIES ON NOAA_water_database FROM "h2o_quality" WHERE "location" = 'coyote_creek' LIMIT 2

key
---
h2o_quality,location=coyote_creek,randtag=1
h2o_quality,location=coyote_creek,randtag=2
```

查询返回数据库`NOAA_water_database`中与measurement `h2o_quality`相关联的并且tag为`location = coyote_creek`的两个series。

## SHOW MEASUREMENTS
返回指定数据库的measurement列表。
### 语法
```
SHOW MEASUREMENTS [ON <database_name>] [WITH MEASUREMENT <regular_expression>] [WHERE <tag_key> <operator> ['<tag_value>' | <regular_expression>]] [LIMIT_clause] [OFFSET_clause]
```

### 语法描述
`ON <database_name>`是可选的。如果查询不包括`ON <database_name>`，则必须在CLI中使用`USE <database_name>`指定数据库，或者在HTTP API请求中指定`db`查询字符串参数。

`WITH`，`WHERE`，`LIMIT`和`OFFSET`子句是可选的。 `WHERE`子句支持tag比较; field比较对`SHOW MEASUREMENTS`查询无效。

`WHERE`子句中支持的运算符：

`=` 等于  
`<>` 不等于  
`!=` 不等于  
`=~` 匹配  
`!~` 不匹配

### 例子
#### 例一：运行带`ON`子句的`SHOW MEASUREMENTS`

```
> SHOW MEASUREMENTS ON NOAA_water_database

name: measurements
name
----
average_temperature
h2o_feet
h2o_pH
h2o_quality
h2o_temperature
```

查询返回数据库`NOAA_water_database`中的measurement列表。数据库有五个measurement：`average_temperature`，`h2o_feet`，`h2o_pH`，`h2o_quality`和`h2o_temperature`。

#### 例二：运行不带`ON`子句的`SHOW MEASUREMENTS`
##### CLI
用`USE <database_name>`指定数据库。
```
> USE NOAA_water_database
Using database NOAA_water_database

> SHOW MEASUREMENTS
name: measurements
name
----
average_temperature
h2o_feet
h2o_pH
h2o_quality
h2o_temperature
```
#### HTTP API
使用参数`db`指定数据库：

```
~# curl -G "http://localhost:8086/query?db=NOAA_water_database&pretty=true" --data-urlencode "q=SHOW MEASUREMENTS"

{
  {
      "results": [
          {
              "statement_id": 0,
              "series": [
                  {
                      "name": "measurements",
                      "columns": [
                          "name"
                      ],
                      "values": [
                          [
                              "average_temperature"
                          ],
                          [
                              "h2o_feet"
                          ],
                          [
                              "h2o_pH"
                          ],
                          [
                              "h2o_quality"
                          ],
                          [
                              "h2o_temperature"
                          ]
                      ]
                  }
              ]
          }
      ]
  }
```

#### 例三：运行有多个子句的`SHOW MEASUREMENTS`(1)
```
> SHOW MEASUREMENTS ON NOAA_water_database WITH MEASUREMENT =~ /h2o.*/ LIMIT 2 OFFSET 1

name: measurements
name
----
h2o_pH
h2o_quality
```

该查询返回以以`h2o`开头的`NOAA_water_database`数据库中的measurement。 `LIMIT`和`OFFSET`子句将返回的measurement名称,并且数量限制为两个，再将结果偏移一个，所以跳过了measurement`h2o_feet`。

#### 例四：运行有多个子句的`SHOW MEASUREMENTS`(2)
```
> SHOW MEASUREMENTS ON NOAA_water_database WITH MEASUREMENT =~ /h2o.*/ WHERE "randtag"  =~ /\d/

name: measurements
name
----
h2o_quality
```

该查询返回`NOAA_water_database`中以`h2o`开头，并且tag`randtag`包含一个整数的所有measurement。

## SHOW TAG KEYS

