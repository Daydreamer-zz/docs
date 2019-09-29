# tomcat优化配置参数

## 1.内存优化

主要是对/usr/local/tomcat/bin/catalina.sh进行修改，在catalina.sh中添加

```
JAVA_OPTS='-server -Xms1024m -Xmx2048m'
```

- -server：启用jdk的server版本
- -Xms：虚拟机初始化时的最小堆内存
- -Xmx: 虚拟机可使用的最大堆内存

## 2.tomcat并发优化

对/usr/local/tomcat/conf/server.xml的优化

```xml
<Connector port="80" protocol="HTTP/1.1"
               maxThreads="1000"
               minProcessors="100"
               maxProcessors="1000"
               minSpareThreads="100"
               maxSpareThreads="1000"
               enableLookups="false"
               URIEncoding="utf-8"
               acceptCount="1000"
               connectionTimeout="20000"
               disableUploadTimeout="ture"
               redirectPort="8443"
               compression="on"
               compressionMinSize="2048"
   compressableMimeType="text/html,text/xml,text/javascript,text/css,text/plain" />
```

- maxThreads：客户请求的最大线程数
- minProcessors: 最小空闲连接进程数，用于提高系统处理性能，默认为10
- maxProcessors: 最大连接进程数，即并发处理的最大请求，默认为75
- minSpareThreads: Tomcat初始化时创建的 socket 线程数
- maxSpareThreads: Tomcat连接器的最大空闲 socket 线程数
- enableLookups： 是否允许域名解析
- URIEncoding:  URL统一编码
- acceptCount: 监听端口队列最大数，满了之后客户请求会被拒绝(不能小于maxSpareThreads )
- connectionTimeout: 连接超时时间
- compression: 开启压缩
- compressionMinSize: 最小压缩大小，单位KB
- compressableMimeType:  启用压缩的文件类型

## 3.tomcat开启apr

### ①安装apr

```shell
[root@node1 ~]# tar xf apr-1.7.0.tar.gz 
[root@node1 ~]# cd apr-1.7.0/
[root@node1 apr-1.7.0]# ./configure --prefix=/usr/local/apr
[root@node1 apr-1.7.0]# make -j4 && make install
```

### ②安装apr-iconv

```shell
[root@node1 ~]# tar xf apr-iconv-1.2.2
[root@node1 apr-iconv-1.2.2]# ./configure --prefix=/usr/local/apr-iconv --with-apr=/usr/local/apr
[root@node1 apr-iconv-1.2.2]# make -j4 && make install
```

## ③安装apr-util

```shell
[root@node1 ~]# tar xf apr-util-1.6.1.tar.gz
[root@node1 apr-iconv-1.2.2]# ./configure --prefix=/usr/local/apr-util --with-apr=/usr/local/apr --with-apr-iconv=/usr/local/apr-iconv/bin/apriconv
[root@node1 apr-iconv-1.2.2]# make -j4 && make install
```

### ④安装tomcat-native

```shell
[root@node1 ~]# tar xf tomcat-native-1.2.21-src.tar.gz
[root@node1 ~]# cd tomcat-native-1.2.21-src/native/
[root@node1 ~]# ./configure --with-apr=/usr/local/apr --with-java-home=/usr/local/jdk1.8.0_191
[root@node1 ~]# make -j4 && make install
```

### ⑤添加环境变量

```shell
[root@node1 ~]# vim /etc/profile

#apr
export LD_LIBRARY_PATH=/usr/local/apr/lib
```

### ⑥修改server.xml

```shell
[root@node1 ~]# vim /usr/local/tomcat/conf/server.xml

protocol="org.apache.coyote.http11.Http11AprProtocol" #修改其中一个connector
<Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="off" /> #修改SSLEngine为off
```

