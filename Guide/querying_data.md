# 读数据

## 使用HTTP接口查询数据
HTTP接口是InfluxDB查询数据的主要方式。通过发送一个`GET`请求到`/query`路径，并设置URL的`db`参数为目标数据库，设置URL参数`q`为查询语句。下面的例子是查询在[写数据](writing_data.md)里写入的数据点。

```
curl -G 'http://localhost:8086/query?pretty=true' --data-urlencode "db=mydb" --data-urlencode "q=SELECT \"value\" FROM \"cpu_load_short\" WHERE \"region\"='us-west'"
```

InfluxDB返回一个json值，你查询的结果在`result`列表中，如果有错误发送，InfluxDB会在`error`这个key里解释错误发生的原因。

```
{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "cpu_load_short",
                    "columns": [
                        "time",
                        "value"
                    ],
                    "values": [
                        [
                            "2015-01-29T21:55:43.702900257Z",
                            2
                        ],
                        [
                            "2015-01-29T21:55:43.702900257Z",
                            0.55
                        ],
                        [
                            "2015-06-11T20:46:02Z",
                            0.64
                        ]
                    ]
                }
            ]
        }
    ]
}
```

>说明：添加`pretty=ture`参数在URL里面，是为了让返回的json格式化。这在调试或者是直接用`curl`的时候很有用，但在生产上不建议使用，因为这样会消耗不必要的网络带宽。

### 多个查询
在一次API调用中发送多个InfluxDB的查询语句，可以简单地使用分号分隔每个查询，例如：

```
curl -G 'http://localhost:8086/query?pretty=true' --data-urlencode "db=mydb" --data-urlencode "q=SELECT \"value\" FROM \"cpu_load_short\" WHERE \"region\"='us-west';SELECT count(\"value\") FROM \"cpu_load_short\" WHERE \"region\"='us-west'"
```

返回：

```
{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "cpu_load_short",
                    "columns": [
                        "time",
                        "value"
                    ],
                    "values": [
                        [
                            "2015-01-29T21:55:43.702900257Z",
                            2
                        ],
                        [
                            "2015-01-29T21:55:43.702900257Z",
                            0.55
                        ],
                        [
                            "2015-06-11T20:46:02Z",
                            0.64
                        ]
                    ]
                }
            ]
        },
        {
            "statement_id": 1,
            "series": [
                {
                    "name": "cpu_load_short",
                    "columns": [
                        "time",
                        "count"
                    ],
                    "values": [
                        [
                            "1970-01-01T00:00:00Z",
                            3
                        ]
                    ]
                }
            ]
        }
    ]
}
```

### 查询数据时的其他可选参数
#### 时间戳格式
在InfluxDB中的所有数据都是存的UTC时间，时间戳默认返回RFC3339格式的纳米级的UTC时间，例如`2015-08-04T19:05:14.318570484Z`，如果你想要返回Unix格式的时间，可以在请求参数里设置`epoch`参数，其中epoch可以是`[h,m,s,ms,u,ns]`之一。例如返回一个秒级的epoch：

```
curl -G 'http://localhost:8086/query' --data-urlencode "db=mydb" --data-urlencode "epoch=s" --data-urlencode "q=SELECT \"value\" FROM \"cpu_load_short\" WHERE \"region\"='us-west'"
```

#### 认证
InfluxDB里面的认证默认是关闭的，查看[认证和鉴权]()章节了解如何开启认证。

#### 最大行限制
可选参数`max-row-limit`允许使用者限制返回结果的数目，以保护InfluxDB不会在聚合结果的时候导致的内存耗尽。

在1.2.0和1.2.1版本中，InfluxDB默认会把返回的数目截断为10000条，如果有超过10000条返回，那么返回体里面会包含一个`"partial":true`的标记。该默认设置可能会导致Grafana面板出现意外行为，如果返回值大于10000时，这个面板就会看到[截断/部分数据](https://github.com/influxdata/influxdb/issues/8050)。

在1.2.2版本中，`max-row-limit`参数默认被设置为了0，这表示说对于返回值没有限制。

这个最大行的限制仅仅作用于非分块(non-chunked)的请求中，分块(chunked)的请求还是返回无限制的数据。

#### 分块(chunking)
可以设置参数`chunked=true`开启分块，使返回的数据是流式的batch，而不是单个的返回。返回结果可以按10000数据点被分块，为了改变这个返回最大的分块的大小，可以在查询的时候加上`chunk_size`参数，例如返回数据点是每20000为一个批次。

```
curl -G 'http://localhost:8086/query' --data-urlencode "db=deluge" --data-urlencode "chunked=true" --data-urlencode "chunk_size=20000" --data-urlencode "q=SELECT * FROM liters"
```

#### InfluxQL
现在你已经知道了如何查询数据，查看[数据探索页面]()可以熟悉InfluxQL的用法。想要获取更多关于数据查询的HTTP接口的用法，情况[API参考文档]()。


