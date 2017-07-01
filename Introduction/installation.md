# 安装

这篇将会介绍怎么安装、运行和配置InfluxDB。

## 准备
安装InfluxDB包需要`root`或是有管理员权限才可以。

### 网络
InfluxDB默认使用下面的网络端口：

* TCP端口`8086`用作InfluxDB的客户端和服务端的http api通信
* TCP端口`8088`给备份和恢复数据的RPC服务使用

另外，InfluxDB也提供了多个可能需要自定义端口的插件，所以的端口映射都可以通过配置文件修改，对于默认安装的InfluxDB，这个配置文件位于`/etc/influxdb/influxdb.conf`。

### NTP
InfluxDB使用服务器本地时间给数据加时间戳，而且是UTC时区的。并使用NTP来同步服务器之间的时间，如果服务器的时钟没有通过NTP同步，那么写入InfluxDB的数据的时间戳就可能不准确。

## 安装
对于不想安装的用户，可以使用inluxdata公司提供的云产品（帮忙给他们打个广告吧，毕竟开源不易）。
### Debain & Ubuntu
Debian和Ubuntu用户可以直接用`apt-get`包管理来安装最新版本的InfluxDB。

对于Ubuntu用户，可以用下面的命令添加InfluxDB的仓库

```
curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -
source /etc/lsb-release
echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
```

Debian用户用下面的命令：

```
curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -
source /etc/os-release
test $VERSION_ID = "7" && echo "deb https://repos.influxdata.com/debian wheezy stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
test $VERSION_ID = "8" && echo "deb https://repos.influxdata.com/debian jessie stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
```

然后安装、运行InfluxDB服务：

```
sudo apt-get update && sudo apt-get install influxdb
sudo service influxdb start
```

如果你的系统可以使用Systemd(比如Ubuntu 15.04+, Debian 8+），也可以这样启动：

```
sudo apt-get update && sudo apt-get install influxdb
sudo systemctl start influxdb
```

### RedHat & CentOS
RedHat和CentOS用户可以直接用`yum`包管理来安装最新版本的InfluxDB。

```
cat <<EOF | sudo tee /etc/yum.repos.d/influxdb.repo
[influxdb]
name = InfluxDB Repository - RHEL \$releasever
baseurl = https://repos.influxdata.com/rhel/\$releasever/\$basearch/stable
enabled = 1
gpgcheck = 1
gpgkey = https://repos.influxdata.com/influxdb.key
EOF
```
一旦加到了`yum`源里面，就可以运行下面的命令来安装和启动InfluxDB服务：

```
sudo yum install influxdb
sudo service influxdb start
```
如果你的系统可以使用Systemd(比如CentOS 7+, RHEL 7+），也可以这样启动：

```
sudo yum install influxdb
sudo systemctl start influxdb
```

### MAC OS X
OS X 10.8或者更高版本的用户，可以使用Homebrew来安装InfluxDB; 一旦`brew`安装了，可以用下面的命令来安装InfluxDB：

```
brew update
brew install influxdb
```

登陆后在用`launchd`开始运行InfluxDB之前，先跑：

```
ln -sfv /usr/local/opt/influxdb/*.plist ~/Library/LaunchAgents
```

然后运行InfluxDB：

```
launchctl load ~/Library/LaunchAgents/homebrew.mxcl.influxdb.plist
```
如果你不想用或是不需要launchctl，你可以直接在terminal里运行下面命令来启动InfluxDB：

```
influxd -config /usr/local/etc/influxdb.conf
```

## 配置
安装好之后，每个配置文件都有了默认的配置，你可以通过命令`influxd config`来查看这些默认配置。

在配置文件`/etc/influxdb/influxdb.conf`之中的大部分配置都被注释掉了，所有这些被注释掉的配置都是由内部默认值决定的。配置文件里任意没有注释的配置都可以用来覆盖内部默认值，需要注意的是，本地配置文件不需要包括每一项配置。

有两种方法可以用自定义的配置文件来运行InfluxDB：

* 运行的时候通过可选参数`-config`来指定：

```
influxd -config /etc/influxdb/influxdb.conf
```

* 设置环境变量`INFLUXDB_CONFIG_PATH`来指定，例如：

```
echo $INFLUXDB_CONFIG_PATH
/etc/influxdb/influxdb.conf


influxd
```

其中`-config`的优先级高于环境变量。

想看更详细的配置解释，请移步[配置]()文档。

## 托管在AWS上
### 硬件
我们建议使用两块SSD卷，一个是为了`influxdb/wal`，一个是为了`influxdb/data`；根据您的负载量，每个卷应具有大约1k-3k的IOPS。`influxdb/data`卷应该有更多的磁盘空间和较低的IOPS，而`influxdb/wal`卷则相反有较少的磁盘空间但是较高的IOPS。

每台机器应该有不少于8G的内存。

在AWS上的R4型号的机器上的我们看到了最好的性能，因为这种型号的机器提供的内存比C3/C4和M4型号机器大得多。

### 配置aws实例
这个例子假定你正在使用两个SSD卷，并且已正确安装它们。此示例还假定每个卷都安装在`/mnt/influx`和`/mnt/db`上。如何安装请看亚马逊提供的文档[给你的aws实例添加卷](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-attaching-volume.html)。

### 配置文件
你需要修改每一个InfluxDB实例的配置文件：

```
...

[meta]
  dir = "/mnt/db/meta"
  ...

...

[data]
  dir = "/mnt/db/data"
  ...
wal-dir = "/mnt/influx/wal"
  ...

...

[hinted-handoff]
    ...
dir = "/mnt/db/hh"
    ...
```

### 权限
如果InfluxDB没有使用标准的数据和配置文件的文件夹的话，你需要确定文件系统的权限是正确的：

```
chown influxdb:influxdb /mnt/influx
chown influxdb:influxdb /mnt/db
```