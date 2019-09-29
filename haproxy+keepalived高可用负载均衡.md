

# haproxy+keepalived高可用负载均衡

## 1.主机规划

| 主机  | ip                           | 角色      |
| ----- | ---------------------------- | --------- |
| node1 | 192.168.2.4(vip:192.168.2.8) | haproxy主 |
| node2 | 192.168.2.5(vip:192.168.2.8) | haproxy备 |
| node3 | 192.168.2.6                  | web1      |
| node4 | 192.168.2.7                  | web2      |

本次实验基于haproxy七层负载均衡模式，设置了一个VIP（Virtual IP）为192.168.2.8，用户只需要访问这个IP地址即可获得网页服务。其中，负载均衡主机为192.168.2.4，备机为192.168.2.5。Web服务器1为192.168.2.6，Web服务器2为192.168.2.7

## 2.安装haproxy

这里选择编译安装haproxy

安装环境

```
yum install -y openssl openssl-devel pcre pcre-devel zlib zlib-devel gcc-c++
```

解压并编译

```
tar xf haproxy-1.8.14.tar.gz &&  cd haproxy-1.8.14/
```

```
make TARGET=linux2628 PREFIX=/usr/local/haproxy USE_OPENSSL=1 USE_PCRE=1 USE_ZLIB=1
```

```
make install PREFIX=/usr/local/haproxy
```

## 3.配置haproxy

node1和node2相同

haproxy.cfg

```
#全局
global
    maxconn 100000
    user haproxy
    group haproxy
    daemon
    nbproc 1
    pidfile /usr/local/haproxy/haproxy.pid
    log 127.0.0.1 local6 info
#默认
defaults
    option http-keep-alive
    option forwardfor  #使后端服务能获取用户真实ip
    maxconn 100000
    mode http          #使用七层负载均衡
    timeout connect 3s
    timeout client 3s
    timeout server 3s
#监听
listen stats
    mode http
    bind 0.0.0.0:9999  #设置监控页面监听地址和端口
    stats enable
    log global
    stats uri /haproxy-status  #监控页面的url
    stats auth shz:199747      #监控页面的用户和密码
#前端服务器
frontend web_port
    bind 0.0.0.0:80   #绑定VIP
    mode http
    option httplog
    log global
    option forwardfor
    default_backend server_www
#后端服务器
backend server_www
    mode http
    option forwardfor header X-REAL-IP
    option httpchk HEAD / HTTP/1.0
    balance roundrobin  #轮询
    server web1 192.168.2.6:80 check inter 2000 rise 3 fall 2
    server web2 192.168.2.7:80 check inter 2000 rise 3 fall 2  
```

haproxy的负载均衡算法

- roundrobin：轮询
- static-rr：加权轮询
- source：ip哈希
- leastconn：最小连接数
- uri：url哈希
- uri_param：根据url路径中的参数转发
- hdr(<name>)：根据http头部转发

server后的参数

check表示对后端服务器进行健康检查；inter后加检查时间间隔，单位为毫秒；rise表示由故障转换为正常要进行成功检查的次数；fall表示服务器变成不可用状态要检查的次数；weight表示权重；backup表示后端的真实备份服务器，仅在后端真实服务器都不可用才启用

## 4.配置keepalived

node1

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
```
## 5. 准备测试页面

```
echo "this is node3" >/usr/share/nginx/html/index.html
```

```
echo "this is node4" >/usr/share/nginx/html/index.html
```

并启动所节点的nginx服务，并启动node1和node2的keepalived服务

## 6.测试

浏览器访问192.168.2.8，不断刷新，测试页面将在两个来回变换

关闭node1或者停止node1的keepalived的服务，浏览器继续刷新页面仍然正常变换说明测试成功

## 7.通过haproxy的ACL规则配置转发

要实现acl规则转发，必须位于7层负载均衡模式下

常用的acl方法有，hdr_reg(host)、hdr_dom(host)、url_sub、url_dir、path_beg、path_end

以下配置一个基于acl规则的负载均衡集群，实现当用户访问www.a.com或者a.com时转发到node3，当访问bbs.a.com时转发到node4

### ①haproxy配置文件

```
#全局
global
    maxconn 100000
    user haproxy
    group haproxy
    daemon
    nbproc 1
    pidfile /usr/local/haproxy/haproxy.pid
    log 127.0.0.1 local6 info
#默认
defaults
    option http-keep-alive
    option forwardfor  #使后端服务能获取用户真实ip
    maxconn 100000
    mode http          #使用七层负载均衡
    timeout connect 3s
    timeout client 3s
    timeout server 3s
#监听
listen stats
    mode http
    bind 0.0.0.0:9999  #设置监控页面监听地址和端口
    stats enable
    log global
    stats uri /haproxy-status  #监控页面的url
    stats auth shz:199747      #监控页面的用户和密码
#前端服务器
frontend web_port
    bind 0.0.0.0:80   #绑定VIP
    mode http
    option httplog
    log global
    option forwardfor
    acl host_www hdr_reg(host) -i ^(www.a.com|a.com)
    acl host_bbs hdr_beg(host) -i bbs.
    use_backend server_www if host_www
    use_backend server_bbs if host_bbs
backend server_www
    mode http
    option forwardfor header X-REAL-IP
    option httpchk HEAD / HTTP/1.0
    balance roundrobin
    server web1 192.168.2.6:80
backend server_bbs
    mode http
    option forwardfor header X-REAL-IP
    option httpchk HEAD / HTTP/1.0
    balance roundrobin
    server web1 192.168.2.7:80
```

### ②准备测试页面

```
echo "www" >/usr/share/nginx/html/index.html
```

```
echo "bbs" >/usr/share/nginx/html/index.html
```

### ③启动haproxy

```
/usr/local/haproxy/sbin/haproxy -f /usr/local/haproxy/sbin/haproxy.cfg
```

### ④在windows上配置好hosts

以管理员权限打开notepad++编辑`C:\Windows\System32\drivers\etc\hosts`，加入

```
192.168.2.4  www.a.com
192.168.2.4  a.com
192.168.2.4  bbs.a.com
```

### ⑤打开浏览器测试

分别访问www.a.com，a.com，bbs.a.com看是否出现正确的测试页面





