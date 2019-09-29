#  CentOS7下编译安装zabbix

使用yum安装的zabbix默认用的是lamp，本文档将示范LNMP环境下zabbix监控的部署，关于LNMP环境的安装暂不过多描述

## 1.安装zabbix所需依赖

```
yum install unixODBC-devel mysql-devel net-snmp-devel libxml2-devel libcurl-devel libevent-devel wget -y
```

## 2.创建zabbix用户和组并授权

```
useradd zabbix -s /sbin/nologin -M
```

```
mkdir -p /usr/local/zabbix/logs
```

```
chown -R zabbix:zabbix /usr/local/zabbix
```

## 3.下载源码包并解压

```
wget https://jaist.dl.sourceforge.net/project/zabbix/ZABBIX%20Latest%20Stable/3.0.26/zabbix-3.0.26.tar.gz
```

```
tar xf zabbix-3.4.10.tar.gz
```

## 4.在mysql中创建库并导入表(导入表的顺序必须按下面的)

```shell
mysql -uroot -p199747 -e 'create database zabbix charset utf8;'
```

```shell
mysql -uroot -p199747 zabbix < /root/zabbix-3.4.10/database/mysql/schema.sql
```

```shell
mysql -uroot -p199747 zabbix < /root/zabbix-3.4.10/database/mysql/images.sql
```

```shell
mysql -uroot -p199747 zabbix < /root/zabbix-3.4.10/database/mysql/data.sql
```

## 5.开始编译安装zabbix-server和zabbix-agent

编译安装的mysql要指定`mysql_config`

```
./configure --prefix=/usr/local/zabbix --enable-server --enable-agent --with-libcurl --with-libxml2 --with-mysql=/usr/local/mysql/bin/mysql_config && make && make install
```

## 6.配置zabbix-server和zabbix_agentd

zabbix_server

```
LogFile=/usr/local/zabbix/logs/zabbix_server.log
PidFile=/usrl/local/zabbix/zabbix_server.pid
DBHost=127.0.0.1
DBUser=root
DBPassword=199747
DBSocket=/data/mysql.sock
Timeout=4
LogSlowQueries=3000
Include=/usr/local/zabbix/etc/zabbix_server.conf.d/*.conf
```

zabbix_agent

```
Server=127.0.0.1
ServerActive=127.0.0.1
Hostname=Zabbix server
Include=/usr/local/zabbix/etc/zabbix_agentd.conf.d/*.conf
```

## 7.启动zabbix_server和zabbix_agentd

```
/usr/local/zabbix/sbin/zabbix_server
```

```
/usr/local/zabbix/sbin/zabbix_agentd
```



启动后查看端口10051和10050是否监听，监听则说明zabbix_server启动成功

## 8. 拷贝zabbix web文件

```
cp -a /root/zabbix-3.4.10/frontends/php/* /usr/local/nginx/html/
```

## 9.配置php.ini

```
post_max_size = 16M
max_execution_time = 300
max_input_time = 300
date.timezone = Asia/Shanghai
always_populate_raw_post_data = -1
```

