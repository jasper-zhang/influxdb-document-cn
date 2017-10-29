# 认证和授权

## 认证
InfluxDB的HTTP API和命令行界面（CLI），包括简单的基于用户凭据的内置认证。当开启认证时，InfluxDB只会执行发送中带有有效证书的HTTP请求。

>注意：认证只发生在HTTP请求范围内。插件目前不具备认证请求的能力，（例如Graphite、collectd等）是没有认证的。

### 创建认证