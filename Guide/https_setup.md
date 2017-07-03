# HTTPS设置

这篇描述了怎样在InfluxDB中开启HTTPS。设置HTTPS可以保护客户端和InfluxDB服务器之间的通信，在某些情况下，HTTPS可以用来验证InfluxDB服务器对客户端的真实性。

如果你计划通过网络来发送请求到InfluxDB，我们强烈建议你开启HTTPS。

## 准备
为了给InfluxDB上设置HTTPS，你需要一个已有的或是新建的InfluxDB实例，还需要TLS证书也可以说SSL证书。InfluxDB支持三种类型的TLS/SSL证书：

* **由证书颁发机构签名的单域证书**  
 这些证书为HTTPS请求提供加密安全性，并允许客户端验证InfluxDB服务器的身份。 如果使用此证书，每个InfluxDB实例都需要一个唯一的单域证书。
 
* **证书颁发机构签发的通配证书**  
这些证书为HTTPS请求提供加密安全性，并允许客户端验证InfluxDB服务器的身份。 可以在不同服务器上的多个InfluxDB实例中使用通配符证书。

* **自签证书**  
自签名证书不由CA签名，您可以在自己的机器上生成。 与CA签署的证书不同，自签名证书仅为HTTPS请求提供加密安全性。 他们不允许客户端验证InfluxDB服务器的身份。 如果您无法获得CA签发的证书，我们建议使用自签名证书。 如果使用此证书，每个InfluxDB实例都需要一个唯一的自签名证书。

无论您的证书类型如何，InfluxDB都支持由私钥文件（.key）和签名证书文件（.crt）文件对组成的证书，以及将私钥文件和签名的证书文件组合成一个捆绑的证书文件（.pem）。

以下两部分将介绍在Ubuntu 16.04上如何使用CA签发的证书和自签名证书给InfluxDB设置HTTPS。其他操作系统的具体步骤可能不同。

## 使用CA签发的证书设置HTTPS

### 第一步：安装SSL / TLS证书
将私钥文件（.key）和签名的证书文件（.crt）或单个捆绑文件（.pem）放在`/etc/ssl`目录中。

### 第二步：确保文件权限
证书文件需要root用户的读写权限。通过运行以下命令确保您具有正确的文件权限：

```
sudo chown root:root /etc/ssl/<CA-certificate-file>
sudo chmod 644 /etc/ssl/<CA-certificate-file>
sudo chmod 600 /etc/ssl/<private-key-file>
```

### 第三步：在InfluxDB的配置文件中开启HTTPS
默认HTTPS是关闭的，在InfluxDB的配置文件`/etc/influxdb/influxdb.conf`的`[http]`部分通过如下设置开启HTTPS：

* `https-enabled`设为`true`
* `http-certificate`设为`/etc/ssl/<signed-certificate-file>.crt`(或者`/etc/ssl/<bundled-certificate-file>.pem`)
* `http-private-key`设为`/etc/ssl/<private-key-file>.key`(或者`/etc/ssl/<bundled-certificate-file>.pem`)

```
[http]

  [...]

  # Determines whether HTTPS is enabled.
  https-enabled = true

  [...]

  # The SSL certificate to use when HTTPS is enabled.
  https-certificate = "<bundled-certificate-file>.pem"

  # Use a separate private key location.
  https-private-key = "<bundled-certificate-file>.pem"
```

### 第四步：重启InfluxDB
重启InfluxDB使配置生效：

```
sudo systemctl restart influxdb
```

### 第五步：验证HTTPS安装
可以通过InfluxDB的CLI来验证HTTPS是否工作：

```
influx -ssl -host <domain_name>.com
```
如果连接成功会返回：

```
Connected to https://<domain_name>.com:8086 version 1.x.x
InfluxDB shell version: 1.x.x
>
```

这样你就成功开启了InfluxDB的HTTPS了。

## 使用自签名证书设置HTTPS
### 第一步：生成自签名证书
以下命令生成私有密钥文件（.key）和自签名证书文件（.crt），该文件对于指定`NUMBER_OF_DAYS`情况下仍然有效。 它将这些文件输出到InfluxDB的默认证书文件路径，并向他们提供所需的权限。

```
sudo openssl req -x509 -nodes -newkey rsa:2048 -keyout /etc/ssl/influxdb-selfsigned.key -out /etc/ssl/influxdb-selfsigned.crt -days <NUMBER_OF_DAYS>
```

当执行该命令时，将提示您提供更多信息。 您可以选择填写该信息或将其留空; 这两个操作都会生成有效的证书文件。

### 第二步：在InfluxDB的配置文件中开启HTTPS

默认HTTPS是关闭的，在InfluxDB的配置文件`/etc/influxdb/influxdb.conf`的`[http]`部分通过如下设置开启HTTPS：

* `https-enabled`设为`true`
* `http-certificate`设为`/etc/ssl/influxdb-selfsigned.crt`
* `http-private-key`设为`/etc/ssl/influxdb-selfsigned.key`

```
[http]

  [...]

  # Determines whether HTTPS is enabled.
  https-enabled = true

  [...]

  # The SSL certificate to use when HTTPS is enabled.
  https-certificate = "/etc/ssl/influxdb-selfsigned.crt"

  # Use a separate private key location.
  https-private-key = "/etc/ssl/influxdb-selfsigned.key"
```

### 第三步：重启InfluxDB
重启InfluxDB使配置生效：

```
sudo systemctl restart influxdb
```

### 第四步：验证HTTPS安装
可以通过InfluxDB的CLI来验证HTTPS是否工作：

```
influx -ssl -unsafeSsl -host <domain_name>.com
```
如果连接成功会返回：

```
Connected to https://<domain_name>.com:8086 version 1.x.x
InfluxDB shell version: 1.x.x
>
```

>#### 将Telegraf连接到一个安全的InfluxDB实例
将Telegraf连接到使用HTTPS的InfluxDB实例需要一些额外的步骤。 
>
在Telegraf的配置文件（`/etc/telegraf/telegraf.conf`）中，编辑`urls`设置以指定`https`而不是`http`，并将`localhost`更改为相关域名。 如果您使用自签名证书，请取消`insecure_skip_verify`的注释设置并将其设置为true。

```
###############################################################################
#                            OUTPUT PLUGINS                                   #
###############################################################################

# Configuration for influxdb server to send metrics to
[[outputs.influxdb]]
  ## The full HTTP or UDP endpoint URL for your InfluxDB instance.
  ## Multiple urls can be specified as part of the same cluster,
  ## this means that only ONE of the urls will be written to each interval.
  # urls = ["udp://localhost:8089"] # UDP endpoint example
  urls = ["https://<domain_name>.com:8086"]

[...]

  ## Optional SSL Config
  [...]
  insecure_skip_verify = true # <-- Update only if you're using a self-signed certificate
```
>然后重启Telegraf就可以啦！