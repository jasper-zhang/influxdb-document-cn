# 认证和授权

## 认证
InfluxDB的HTTP API和命令行界面（CLI），包括简单的基于用户凭据的内置认证。当开启认证时，InfluxDB只会执行发送中带有有效证书的HTTP请求。

>注意：认证只发生在HTTP请求范围内。插件目前不具备认证请求的能力，（例如Graphite、collectd等）是没有认证的。

### 创建认证
#### 1. 至少创建一个admin用户

>如果你开启了认证但是没有用户，那么InfluxDB将不会开启认证，而且只有在创建了一个admin用户之后才会接受外部请求。

当创建一个admin用户后，InfluxDB才能开启认证。

#### 2. 在配置文件中，认证默认是不开启的
将`[http]`区域的配置`auth-enabled`设为`true`，可以开启认证：

```
[http]  
  enabled = true  
  bind-address = ":8086"  
  auth-enabled = true # ✨
  log-enabled = true  
  write-tracing = false  
  pprof-enabled = false  
  https-enabled = false  
  https-certificate = "/etc/ssl/influxdb.pem"  
```

#### 3. 重启进程
现在InfluxDB会核对每个请求中的用户信息，只会处理已有用户而认证通过的请求。

### 认证请求
#### HTTP API中的认证
有两个HTTP API进行验证的方式。

如果使用基本身份验证和URL查询参数进行身份验证，则以查询参数中指定的用户优先。下面的示例中的查询假定用户是admin用户。

##### 用[RFC 2617](http://tools.ietf.org/html/rfc2617)中所描述的基本身份验证

这是提供用户的首选方法。例:

```
curl -G http://localhost:8086/query -u todd:influxdb4ever --data-urlencode "q=SHOW DATABASES"
```

##### 在URL的参数或是请求体里面提供认证
设置`u`和`p`参数， 例：

```
curl -G "http://localhost:8086/query?u=todd&p=influxdb4ever" --data-urlencode "q=SHOW DATABASES"
```

认证在请求体中的例子：

```
curl -G http://localhost:8086/query --data-urlencode "u=todd" --data-urlencode "p=influxdb4ever" --data-urlencode "q=SHOW DATABASES"
```

#### CLI中的认证
在CLI中有三种认证方式。

##### 设置`INFLUX_USERNAME`和`INFLUX_PASSWORD`环境变量
例如：

```
export INFLUX_USERNAME todd
export INFLUX_PASSWORD influxdb4ever
echo $INFLUX_USERNAME $INFLUX_PASSWORD
todd influxdb4ever

influx
Connected to http://localhost:8086 version 1.3.x
InfluxDB shell 1.3.x
```

##### 开启CLI时设置`username`和`password`
例如：

```
influx -username todd -password influxdb4ever
Connected to http://localhost:8086 version 1.3.x
InfluxDB shell 1.3.x
```

##### 开启CLI后使用`auth <username> <password>`
例如：

```
influx
Connected to http://localhost:8086 version 1.3.x
InfluxDB shell 1.3.x
> auth
username: todd
password:
>
```

## 授权
