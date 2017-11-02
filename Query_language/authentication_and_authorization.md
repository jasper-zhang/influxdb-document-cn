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
当开启认证之后，授权也就开启了。默认情况下，所有的用户都有所有的权限。

### 用户类型和权限
#### admin用户
admin用户有所有数据库的读写权限，这些所有的权限包括如下：

数据库管理：

* `CREATE DATABASE`, `DROP DATABASE`
* `DROP SERIES`, `DROP MEASUREMENT`
* `CREATE RETENTION POLICY`,` ALTER RETENTION POLICY`, 和 `DROP RETENTION POLICY`
* `CREATE CONTINUOUS QUERY`和`DROP CONTINUOUS QUERY`

用户管理：

* admin用户管理：  
  `CREATE USER`, `GRANT ALL PRIVILEGES`, `REVOKE ALL PRIVILEGES`,和`SHOW USERS`

* 非admin用户管理：  
  `CREATE USER`, `GRANT [READ,WRITE,ALL]`, `REVOKE [READ,WRITE,ALL]`,和`SHOW GRANTS`
  
* 一般admin用户管理：  
  `SET PASSWORD`和`DROP USER`
 
#### 非admin用户
非admin用户对于每个数据库，有如下三个权限：

* `READ`
* `WRITE`
* `ALL`（包括`READ`和`WRITE`）

`READ`,`WRITE`和`ALL`控制到每个数据库每个用户上。一个新的非admin用户对任何数据库都没有权限，除非被admin用户指定一个数据库权限。非admin用户可以在他们有`READ`或/和`WRITE`权限的机器上运行`SHOW`命令。

### 用户管理命令
#### admin用户管理
当开启HTTP认证之后，在你操作系统之前，InfluxDB要求你至少创建一个admin用户。

```
CREATE USER admin WITH PASSWORD '<password>' WITH ALL PRIVILEGES
``` 

##### `CREATE`另一个admin用户

```
CREATE USER <username> WITH PASSWORD '<password>' WITH ALL PRIVILEGES
```

CLI例子：

```
> CREATE USER paul WITH PASSWORD 'timeseries4days' WITH ALL PRIVILEGES
>
```

>注意：重复创建确切的用户语句是幂等的。如果有值发生变化，数据库将返回一个重复的用户错误。
>CLI例子：
>
>```
> CREATE USER todd WITH PASSWORD '123456' WITH ALL PRIVILEGES
> CREATE USER todd WITH PASSWORD '123456' WITH ALL PRIVILEGES
> CREATE USER todd WITH PASSWORD '123' WITH ALL PRIVILEGES
ERR: user already exists
> CREATE USER todd WITH PASSWORD '123456'
ERR: user already exists
> CREATE USER todd WITH PASSWORD '123456' WITH ALL PRIVILEGES
>
>```

##### 给一个存在的用户`GRANT`权限

```
GRANT ALL PRIVILEGES TO <username>
```

CLI例子：

```
> GRANT ALL PRIVILEGES TO "todd"
>
```

##### 给一个admin用户`REVOKE`权限
```
REVOKE ALL PRIVILEGES FROM <username>
```

CLI例子：

```
> REVOKE ALL PRIVILEGES FROM "todd"
>
```

##### `SHOW`所有存在的用户以及其admin状态
```
SHOW USERS
```

CLI例子：

```
> SHOW USERS
user 	 admin
todd     false
paul     true
hermione false
dobby    false
```

#### 非admin用户管理
##### `CREATE`一个非admin用户：

```
CREATE USER <username> WITH PASSWORD '<password>'
``` 

CLI例子：

```
> CREATE USER todd WITH PASSWORD 'influxdb41yf3'
> CREATE USER alice WITH PASSWORD 'wonder\'land'
> CREATE USER "rachel_smith" WITH PASSWORD 'asdf1234!'
> CREATE USER "monitoring-robot" WITH PASSWORD 'XXXXX'
> CREATE USER "$savyadmin" WITH PASSWORD 'm3tr1cL0v3r'
>
```

>注意：
>
>* 用户名如果用一个数字的开始，或者是一个influxql关键字，又或者包含任何特殊字符，例如：`!@#$%^&*()-`, 则必须用双引号引起来
>* 密码字符串必须用单引号引起来。
>* 在验证请求时不包含单引号。
>
>如果密码包含单引号或换行符，当提交认证请求时，应该用反斜杠转义。

##### `GRANT READ, WRITE`或者`ALL`数据库权限给一个存在的用户

```
GRANT [READ,WRITE,ALL] ON <database_name> TO <username>
```

CLI例子： 给用户`todd``GRANT`数据库`NOAA_water_database`的`READ`的权限：

```
> GRANT READ ON "NOAA_water_database" TO "todd"
>
```

给用户`todd``GRANT`数据库`NOAA_water_database`的`ALL`的权限：

```
> GRANT ALL ON "NOAA_water_database" TO "todd"
>
```

##### `REVOKE READ, WRITE`或者`ALL` 数据库权限给一个存在的用户

```
REVOKE [READ,WRITE,ALL] ON <database_name> FROM <username>
```

CLI例子： 给用户`todd``REVOKE`数据库`NOAA_water_database`的`ALL`的权限：

```
> REVOKE ALL ON "NOAA_water_database" FROM "todd"
>
```

给用户`todd``REVOKE`数据库`NOAA_water_database`的`WRITE`的权限：

```
> REVOKE WRITE ON "NOAA_water_database" FROM "todd"
>
```

##### `SHOW`一个用户的数据库权限

```
SHOW GRANTS FOR <user_name>

```

CLI例子：

```
> SHOW GRANTS FOR "todd"
database		            privilege
NOAA_water_database	        WRITE
another_database_name	    READ
yet_another_database_name   ALL PRIVILEGES
```

#### 普通admin和非admin用户管理
##### 重新设置一个用户的密码

```
SET PASSWORD FOR <username> = '<password>'
```

CLI例子：

```
> SET PASSWORD FOR "todd" = 'influxdb4ever'
>
```

```
> **Note:** The password [string](/influxdb/v1.3/query_language/spec/#strings) must be wrapped in single quotes.
```

##### `DROP`一个用户
```
DROP USER <username>
```

CLI例子：

```
> DROP USER "todd"
>
```

## 认证和授权的HTTP错误
没有认证或者是带有不正确认证信息的请求发到InfluxDB，将会返回一个`HTTP 401 Unauthorized`的错误。

没有授权的用户的请求发到InfluxDB，将会返回一个`HTTP 403 Forbidden`的错误。