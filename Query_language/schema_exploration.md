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
返回指定数据库的tag key列表。
### 语法
```
SHOW TAG KEYS [ON <database_name>] [FROM_clause] [WHERE <tag_key> <operator> ['<tag_value>' | <regular_expression>]] [LIMIT_clause] [OFFSET_clause]
```

### 语法描述
`ON <database_name>`是可选的。如果查询不包括`ON <database_name>`，则必须在CLI中使用`USE <database_name>`指定数据库，或者在HTTP API请求中指定`db`查询字符串参数。

`FROM`和`WHERE`子句是可选的。 `WHERE`子句支持tag比较; field比较对`SHOW TAG KEYS`查询无效。

`WHERE`子句中支持的运算符：

`=` 等于  
`<>` 不等于  
`!=` 不等于  
`=~` 匹配  
`!~` 不匹配

### 例子
#### 例一：运行带有`ON`子句的`SHOW TAG KEYS`
```
> SHOW TAG KEYS ON "NOAA_water_database"

name: average_temperature
tagKey
------
location

name: h2o_feet
tagKey
------
location

name: h2o_pH
tagKey
------
location

name: h2o_quality
tagKey
------
location
randtag

name: h2o_temperature
tagKey
------
location
```

查询返回数据库`NOAA_water_database`中的tag key列表。输出按measurement名称给tag key分组; 它显示每个measurement都具有tag key`location`，并且measurement`h2o_quality`具有额外的tag key `randtag`。

#### 例二：运行不带`ON`子句的`SHOW TAG KEYS`
##### CLI

用`USE <database_name>`指定数据库：

```
> USE NOAA_water_database
Using database NOAA_water_database

> SHOW TAG KEYS

name: average_temperature
tagKey
------
location

name: h2o_feet
tagKey
------
location

name: h2o_pH
tagKey
------
location

name: h2o_quality
tagKey
------
location
randtag

name: h2o_temperature
tagKey
------
location
```

##### HTTP API
用参数`db`指定数据库：

```
~# curl -G "http://localhost:8086/query?db=NOAA_water_database&pretty=true" --data-urlencode "q=SHOW TAG KEYS"

{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "average_temperature",
                    "columns": [
                        "tagKey"
                    ],
                    "values": [
                        [
                            "location"
                        ]
                    ]
                },
                {
                    "name": "h2o_feet",
                    "columns": [
                        "tagKey"
                    ],
                    "values": [
                        [
                            "location"
                        ]
                    ]
                },
                {
                    "name": "h2o_pH",
                    "columns": [
                        "tagKey"
                    ],
                    "values": [
                        [
                            "location"
                        ]
                    ]
                },
                {
                    "name": "h2o_quality",
                    "columns": [
                        "tagKey"
                    ],
                    "values": [
                        [
                            "location"
                        ],
                        [
                            "randtag"
                        ]
                    ]
                },
                {
                    "name": "h2o_temperature",
                    "columns": [
                        "tagKey"
                    ],
                    "values": [
                        [
                            "location"
                        ]
                    ]
                }
            ]
        }
    ]
}
```

#### 例三：运行带有多个子句的`SHOW TAG KEYS`
```
> SHOW TAG KEYS ON "NOAA_water_database" FROM "h2o_quality" LIMIT 1 OFFSET 1

name: h2o_quality
tagKey
------
randtag
```

该查询从数据库`NOAA_water_database`的measurement`h2o_quality`中返回tag key。 `LIMIT`和`OFFSET`子句限制返回到一个tag key，再将结果偏移一个。

## SHOW TAG VALUES
返回数据库中指定tag key的tag value列表。
### 语法
```
SHOW TAG VALUES [ON <database_name>][FROM_clause] WITH KEY [ [<operator> "<tag_key>" | <regular_expression>] | [IN ("<tag_key1>","<tag_key2")]] [WHERE <tag_key> <operator> ['<tag_value>' | <regular_expression>]] [LIMIT_clause] [OFFSET_clause]
```

### 语法描述
`ON <database_name>`是可选的。如果查询不包括`ON <database_name>`，则必须在CLI中使用`USE <database_name>`指定数据库，或者在HTTP API请求中指定`db`查询字符串参数。

`WITH`子句是必须的，它支持指定一个单独的tag key、一个表达式或是多个tag key。

`FROM`、`WHERE`、`LIMIT`和`OFFSET`子句是可选的。 `WHERE`子句支持tag比较; field比较对`SHOW TAG KEYS`查询无效。

`WHERE`子句中支持的运算符：

`=` 等于  
`<>` 不等于  
`!=` 不等于  
`=~` 匹配  
`!~` 不匹配

### 例子
#### 例一：运行带有`ON`子句的`SHOW TAG VALUES`
```
> SHOW TAG VALUES ON "NOAA_water_database" WITH KEY = "randtag"

name: h2o_quality
key       value
---       -----
randtag   1
randtag   2
randtag   3
```

该查询返回数据库`NOAA_water_database`，tag key为`randtag`的所有tag value。`SHOW TAG VALUES`将结果按measurement名字分组。

#### 例二：运行不带`ON`子句的`SHOW TAG VALUES`
#### CLI

用`USE <database_name>`指定数据库：

```
> USE NOAA_water_database
Using database NOAA_water_database

> SHOW TAG VALUES WITH KEY = "randtag"

name: h2o_quality
key       value
---       -----
randtag   1
randtag   2
randtag   3
```

##### HTTP API
用参数`db`指定数据库：

```
~# curl -G "http://localhost:8086/query?db=NOAA_water_database&pretty=true" --data-urlencode 'q=SHOW TAG VALUES WITH KEY = "randtag"'

{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "h2o_quality",
                    "columns": [
                        "key",
                        "value"
                    ],
                    "values": [
                        [
                            "randtag",
                            "1"
                        ],
                        [
                            "randtag",
                            "2"
                        ],
                        [
                            "randtag",
                            "3"
                        ]
                    ]
                }
            ]
        }
    ]
}
```
#### 例三：运行带有多个子句的`SHOW TAG VALUES`
```
> SHOW TAG VALUES ON "NOAA_water_database" WITH KEY IN ("location","randtag") WHERE "randtag" =~ /./ LIMIT 3

name: h2o_quality
key        value
---        -----
location   coyote_creek
location   santa_monica
randtag	   1
```

该查询从数据库`NOAA_water_database`的所有measurement中返回tag key为`location`或者`randtag`，并且`randtag`的tag value不为空的tag value。 `LIMIT`子句限制返回三个tag value。

## SHOW FIELD KEYS
返回field key以及其field value的数据类型。
### 语法
```
SHOW FIELD KEYS [ON <database_name>] [FROM <measurement_name>]
```

### 语法描述
`ON <database_name>`是可选的。如果查询不包括`ON <database_name>`，则必须在CLI中使用`USE <database_name>`指定数据库，或者在HTTP API请求中指定`db`查询字符串参数。

`FROM`子句也是可选的。

### 例子
#### 例一：运行一个带`ON`子句的`SHOW FIELD KEYS`
```
> SHOW FIELD KEYS ON "NOAA_water_database"

name: average_temperature
fieldKey            fieldType
--------            ---------
degrees             float

name: h2o_feet
fieldKey            fieldType
--------            ---------
level description   string
water_level         float

name: h2o_pH
fieldKey            fieldType
--------            ---------
pH                  float

name: h2o_quality
fieldKey            fieldType
--------            ---------
index               float

name: h2o_temperature
fieldKey            fieldType
--------            ---------
degrees             float
```

该查询返回数据库`NOAA_water_database`中的每个measurement对应的field key以及其数据类型。

#### 例二：运行一个不带`ON`子句的`SHOW FIELD KEYS`
##### CLI
用`USE <database_name>`指定数据库：

```
> USE NOAA_water_database
Using database NOAA_water_database

> SHOW FIELD KEYS

name: average_temperature
fieldKey            fieldType
--------            ---------
degrees             float

name: h2o_feet
fieldKey            fieldType
--------            ---------
level description   string
water_level         float

name: h2o_pH
fieldKey            fieldType
--------            ---------
pH                  float

name: h2o_quality
fieldKey            fieldType
--------            ---------
index               float

name: h2o_temperature
fieldKey            fieldType
--------            ---------
degrees             float
```

##### HTTP API
用参数`db`指定数据库：

```
~# curl -G "http://localhost:8086/query?db=NOAA_water_database&pretty=true" --data-urlencode 'q=SHOW FIELD KEYS'

{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "average_temperature",
                    "columns": [
                        "fieldKey",
                        "fieldType"
                    ],
                    "values": [
                        [
                            "degrees",
                            "float"
                        ]
                    ]
                },
                {
                    "name": "h2o_feet",
                    "columns": [
                        "fieldKey",
                        "fieldType"
                    ],
                    "values": [
                        [
                            "level description",
                            "string"
                        ],
                        [
                            "water_level",
                            "float"
                        ]
                    ]
                },
                {
                    "name": "h2o_pH",
                    "columns": [
                        "fieldKey",
                        "fieldType"
                    ],
                    "values": [
                        [
                            "pH",
                            "float"
                        ]
                    ]
                },
                {
                    "name": "h2o_quality",
                    "columns": [
                        "fieldKey",
                        "fieldType"
                    ],
                    "values": [
                        [
                            "index",
                            "float"
                        ]
                    ]
                },
                {
                    "name": "h2o_temperature",
                    "columns": [
                        "fieldKey",
                        "fieldType"
                    ],
                    "values": [
                        [
                            "degrees",
                            "float"
                        ]
                    ]
                }
            ]
        }
    ]
}
```

#### 例三：运行带有`FROM`子句的`SHOW FIELD KEYS`
```
> SHOW FIELD KEYS ON "NOAA_water_database" FROM "h2o_feet"

name: h2o_feet
fieldKey            fieldType
--------            ---------
level description   string
water_level         float
```

该查询返回数据库`NOAA_water_database`中measurement为`h2o_feet`的对应的field key以及其数据类型。

### `SHOW FIELD KEYS`的常见问题
#### 问题一：`SHOW FIELD KEYS`和field 类型的差异
field value的数据类型在同一个shard里面一样但是在多个shard里面可以不同，`SHOW FIELD KEYS`遍历每个shard返回与field key相关的每种数据类型。

##### 例子
field `all_the_types`中存储了四个不同的数据类型

```
> SHOW FIELD KEYS

name: mymeas
fieldKey        fieldType
--------        ---------
all_the_types   integer
all_the_types   float
all_the_types   string
all_the_types   boolean
```

注意`SHOW FIELD KEYS`处理field的类型差异和`SELECT`语句不一样。
