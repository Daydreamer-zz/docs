# DNS配置解析一个正向区域

以shz.com为例

## (1).定义区域

在主配置文件中或者主配置辅助配置文件中实现

```
zone "shz.com" IN {

		type master;

		file "shz.com.zone";

	};	
```

## (2)建立区域数据文件

在/var/named目录下建立数据文件

```
vim /var/named/shz.com.zone
```

```bash
$TTL 3600
@       IN      SOA     ns1 nsadmin (
                                2018022801
                                1H
                                10M
                                3D
                                1D )
        IN      NS      ns1
        IN      MX   10 mx1
        IN      MX   20 mx2
        IN      NS      ns2


ns1     IN      A       192.168.2.4
ns2     IN      A       192.168.2.5
mx1     IN      A       192.168.2.7
mx2     IN      A       192.168.2.8
www     IN      A       192.168.2.4
bbs     IN      A       192.168.2.5
blog    IN      CNAME   www

```



## (3)让服务器重载配置文件和区域文件

1.检查zone

````
named-checkzone shz.com /var/named/shz.com.zone
````

2.重载

```
rndc reload
```



