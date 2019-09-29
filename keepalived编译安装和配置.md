# keepalived编译安装和配置

## 1.安装前环境

```
yum -y install gcc-c++ libnl libnl-devel libnfnetlink-devel openssl openssl-devel
```

## 2.编译安装

```
tar xf /root/keepalived-2.0.10.tar.gz && cd /root/keepalived-2.0.10 && ./configure ./configure --prefix=/application/keepalived && make -j4 && make install
```

## 3.配置

启动脚本

```
cp /root/keepalived-2.0.10/keepalived/etc/init.d/keepalived /etc/init.d/
```

拷贝配置文件

```
mkdir -p /etc/keepalived && cp /application/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
```

其他

```
cp /root/keepalived-2.0.10/keepalived/etc/sysconfig/keepalived /etc/sysconfig/keepalived
```



```
yum -y install gcc-c++ libnl libnl-devel libnfnetlink-devel openssl openssl-devel && tar xf /root/keepalived-2.0.10.tar.gz && cd /root/keepalived-2.0.10 && ./configure ./configure --prefix=/application/keepalived && make -j4 && make install && cp /root/keepalived-2.0.10/keepalived/etc/init.d/keepalived /etc/init.d/ && mkdir -p /etc/keepalived && cp /application/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/ && mkdir -p /etc/keepalived && cp /application/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/ && cd /root
```

