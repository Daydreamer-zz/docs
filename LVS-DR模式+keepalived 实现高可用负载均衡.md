# LVS-DR模式+keepalived 实现高可用负载均衡

## 1.  主机和ip规划

| 主机  | ip                           | 角色  |
| ----- | ---------------------------- | ----- |
| node1 | 192.168.2.4(vip:192.168.2.8) | lvs主 |
| node2 | 192.168.2.5                  | lvs备 |
| node3 | 192.168.2.6                  | web1  |
| node4 | 192.168.2.7                  | web2  |

- 本次基于VMware Workstation14搭建一个四台Linux（CentOS 7.4）系统所构成的一个服务器集群，其中两台负载均衡服务器（一台为主机，另一台为备机），另外两台作为真实的Web服务器（向外部提供http服务，这里仅仅使用了nginx作为web服务，4台主机均关闭iptables/firewalld，selinux
- 本次实验基于DR负载均衡模式，设置了一个VIP（Virtual IP）为192.168.2.8，用户只需要访问这个IP地址即可获得网页服务。其中，负载均衡主机为192.168.2.4，备机为192.168.2.5。Web服务器1为192.168.2.6，Web服务器2为192.168.2.7

## 2.安装软件

node1和node2上安装keepalived，lvs是基于linux内核的无需安装

这里选择编译安装keepaived

### 安装环境

```
yum -y install gcc-c++ libnl libnl-devel libnfnetlink-devel openssl openssl-devel
```

### 解压并编译

```
tar xf /root/keepalived-2.0.10.tar.gz && cd /root/keepalived-2.0.10 && ./configure --prefix=/usr/local/keepalived && make && make install
```

node3和node4安装nginx，这里直接选择yum安装

```
yum install -y nginx
```

并准备和测试用页面

```
echo "this is node3" >/usr/share/nginx/html/index.html
```

```
echo "this is node4" >/usr/share/nginx/html/index.html
```

## 3.配置主备负载均衡器

node1

```
vim /usr/local/keepalived/keeplived.conf
```

```
! Configuration File for keepalived

global_defs {
   notification_email {
     shz1997@hotmail.com
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.2.4
   smtp_connect_timeout 30
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.2.8/24 dev eth0
    }
}

virtual_server 192.168.2.8 80 {
    delay_loop 6
    lb_algo wrr
    lb_kind DR 
    #persistence_timeout 1
    protocol TCP
    real_server 192.168.2.6 80 {
        weight 1
        TCP_CHECK {
        connect_timeout 8
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80
        }
    }
    real_server 192.168.2.7 80 {
        weight 1
        TCP_CHECK {
        connect_timeout 8
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80
        }
    }
}
```

node2

```
! Configuration File for keepalived

global_defs {
   notification_email {
     shz1997@hotmail.com
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.2.4
   smtp_connect_timeout 30
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 50
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.2.8/24 dev eth0
    }
}

virtual_server 192.168.2.8 80 {
    delay_loop 6
    lb_algo wrr
    lb_kind DR 
    #persistence_timeout 1
    protocol TCP
    real_server 192.168.2.6 80 {
        weight 1
        TCP_CHECK {
        connect_timeout 8
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80
        }
    }
    real_server 192.168.2.7 80 {
        weight 1
        TCP_CHECK {
        connect_timeout 8
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80
        }
    }
}
```

## 4.realserver做arp抑制并绑定vip

由于lvs-dr模式下调度器和realserver在一个网段，为了只让调度器相应arp广播，所以需要在realserver上做arp抑制

node3和node4上

- 绑定vip至lo:0

```
ifconfig lo:0 192.168.2.8 netmask 255.255.255.255 up
```

或者执行

```
ip addr add 192.168.2.8/32 dev lo lable lo:0
```

- arp抑制

```
echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
```

如果要永久生效

 在`/etc/sysctl.conf`下写入以下内容

```
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
```

```
sysctl -p
```

- 提供一个脚本实现以上俩个功能

realserver_vip.sh

```shell
#!/bin/bash
#auther:shz
#lvs-dr模式下为realserver绑定vip和arp抑制
VIP=192.168.2.8
start(){
    ifconfig lo:0 $VIP netmask 255.255.255.255 up
    echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
    echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
    echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
    echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
}
stop(){
    ifconfig lo:0 down
    echo "0" >/proc/sys/net/ipv4/conf/lo/arp_ignore
    echo "0" >/proc/sys/net/ipv4/conf/lo/arp_announce
    echo "0" >/proc/sys/net/ipv4/conf/all/arp_ignore
    echo "0" >/proc/sys/net/ipv4/conf/all/arp_announce
}
case $1 in
  start)
    start;
  ;;
  stop)
    stop;
  ;;
  *)
    echo "Usage: $0 {start|stop}"
    exit 1
esac
exit 0     
```

## 5.启动keepalived并打开浏览器测试

```
/usr/local/keepalived/sbin/keepalived -f /usr/local/keepalived/etc/keepalived/keepalived.conf
```

浏览器访问192.168.2.8，不断Ctrl+F5刷新可以看见页面在`this is node3`和`this is node4`来回变换

将node1主调度器关机或者停掉node1的keepalived服务，继续刷新，vip将飘到node2上，继续Ctrl+F5刷新页面仍然能够自由变换说明测试成功