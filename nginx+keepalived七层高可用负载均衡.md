# nginx+keepalived七层高可用负载均衡

## 1.介绍

nginx的负载均衡功能是用过upstream命令实现的，因此它的负载均衡机制的实现比较简单，这是一个基于内容和应用的七层交换负载均衡的实现。nginx负载均衡默认对后端服务器有健康监测能力，但是监测能力较弱，仅限制于端口检测，在后端服务器较少的情况下负载均衡能力表现突出。而对于有大量后端节点的负载应用，由于所有访问请求都从一台服务器进出，容易发生请求阻塞而引起连接失败，因此无法充分发乎后端服务器的性能

## 2.负载均衡算法

- 轮询(默认)
- weight(权重)
- ip_hash：同一个ip请求分配到一个后端服务器上，确保会话保持
- url_hash

在upstream模块中，可通过server命令指定后端服务器的ip和端口，还可以指定后端服务器的状态，有以下状态

- down：表示当前的sever不参与负载均衡
- backup：预留的备份机器，当其他非backup机器宕机，会调度到这个服务器上
- max_fails：允许请求失败的次数，默认为1。当超过最大次数时，返回proxy_next_upstream模块定义的错误
- fail_timeout：在经历了max_fails多次失败后，暂停服务的时间

## 3.主机规划

| 主机  | Ip                           | 角色       |
| ----- | ---------------------------- | ---------- |
| node1 | 192.168.2.4(vip:192.168.2.8) | nginx_lb主 |
| node2 | 192.168.2.5(vip:192.168.2.8) | nginx_lb备 |
| node3 | 192.168.2.6                  | web1       |
| node4 | 192.168.2.7                  | web2       |

## 4.nginx负载均衡配置

node1和node2上配置相同

nginx.conf

```
http
{
    upstream web-servers {
        server 192.168.2.6:80 weight=1 max_fails=3 fail_timeout=20s;
        server 192.168.2.7:80 weight=1 max_fails=3 fail_timeout=20s;
    }
}
server
{
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html
    location / {
        proxy_pass http://web-servers;
        proxy_next_upstream http_500 http_502 http_503 error timeout invalid_header; 
    }
}
```

其中`proxy_next_upstream`段定义了故障专义策略，当后端服务器返回500 502 503 504和超时错误时，自动将请求转移到upstream负载均衡组的另一台服务器上，从而实现故障转移。

## 5.keepalived的配置

`vim keepalived.conf`

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

## 6. 准备测试页面

```
echo "this is node3" >/usr/share/nginx/html/index.html
```

```
echo "this is node4" >/usr/share/nginx/html/index.html
```

并启动所节点的nginx服务，并启动node1和node2的keepalived服务

## 7.测试

浏览器访问192.168.2.8，不断刷新，测试页面将在两个来回变换

关闭node1或者停止node1的keepalived的服务，浏览器继续刷新页面仍然正常变换说明测试成功

## 8.根据客户端设备(UA)转发

nginx.conf

```
http {
    upstream upload_pools {
        server 192.168.2.5:80;
    }
    upstream default_pools {
        server 192.168.2.6:80;
    }
    upstream static_pools {
        server 192.168.2.7:80;
    }
}
server {
    listen 80;
    server_name localhost;
    localtion / {
        if ($http_user_agent ~* "MSIE")
        {
            proxy_pass http://static_pools;
        }
        if ($http_user_agent ~* "Chrome")
        {
            proxy_pass http://upload_pools;
        }
        proxy_pass http://default_pools;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote-addr;
    }
}
```

