# mysql主主互备模式

node1 192.168.1.4

node2 192.168.1.5

node3 192.168.1.6

## 1. 修改mysql配置文件

做如下修改

node2

```
[mysqld]
server-id   = 1
log-bin     = /data/binlog/mysql-bin
relay-log   = /data/binlog/mysql-relay-bin
replicate-wild-ignore-table=mysql.%
replicate-wild-ignore-table=test.%
replicate-wild-ignore-table=information_schema.%
```

node3

```
[mysqld]
server-id   = 2
log-bin     = /data/binlog/mysql-bin
relay-log   = /data/binlog/mysql-relay-bin
replicate-wild-ignore-table=mysql.%
replicate-wild-ignore-table=test.%
replicate-wild-ignore-table=information_schema.%
```

## 2.重启数据库

```
mysqladmin -uroot -p shutdown -S /data/mysql.sock
```

```
mysqld_safe --defaults-files=/etc/my.cnf &
```

## 3.开启主从

登录node2的mysql数据库，添加复制用户

```
mysql> grant replication slave on *.* to repl@'192.168.1.6' identified by 'slave-pass';
```

查看node2的mysql binlog日志的文件名和pos

```
mysql> show master status'
```

然后登录node3的mysql数据库开启复制

```
mysql> change master to master_host='192.168.1.5',master_user='repl',master_password='slave-pass',master_log_file='mysql-bin.000001',master_log_pos=178;
```

```
mysql> start slave;
```

在node3上查看复制状态

```
mysql> show slave status\G;
```

同理在node2上也要做复制操作，完成后，node2和node3的mysql数据库互为主从

## 4.配置keepalived

keepalived的安装暂不详细描述

node2的keepalived配置文件

```
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.88/24
    }
```

node3的keepalived配置文件

```
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 50
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.88/24
    }
```

分别启动keepalived服务后，node2和node3共同守护192.168.1.88这个vip，当node2主keepalived服务关闭或者node2宕机，node3就会接管这个vip

## 5.测试

这里以部署一个wordpress为例，node1 192.168.1.4上已经部署好nginx,php,在初始化wordpress的时候连接node2和node3的mysql数据库

先随便登录一个库，添加wordpress库和用户

```
mysql> create database wordpress character set utf8;
mysql> grant all on wordpress.* to wordpress@'%' identified by 'wordpress';
```

浏览器访问wordpress，初始化数据库的时候连接192.168.1.88这个vip

![QQ截图20181212152313](C:\Users\shz\Desktop\QQ截图20181212152313.png)

关闭node2的keepalived服务或给node2断电，node3会接替192.168.1.88这个vip，且node3数据库和node2完全相同，node3数据库可以继续提供服务，然后再登录wordpress后台，没有出现异常说明测试成功