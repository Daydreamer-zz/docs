# 使用frp部署内网穿透

当客户端没有固定公网ip，而且处于内网，在不使用端口映射的情况下，只需要一个有固定公网ip的服务器端

这里以本人阿里轻量应用服务器(39.105.62.80)为例，配置服务端，注意打开防火墙端口

## 1.下载包

```
wget https://github.com/fatedier/frp/releases/download/v0.27.0/frp_0.27.0_linux_amd64.tar.gz
```

## 2.安装并配置服务端

 ```
[root@shzll ~]# tar xf frp_0.27.0_linux_amd64.tar.gz && cd frp_0.27.0_linux_amd64
 ```

```
[root@shzll frp_0.27.0_linux_amd64]# vim frps.ini

[common]
bind_addr = 0.0.0.0
bind_port = 7000        #客户端与服务端进行通信的端口，即frp服务端口，需与客户端server_port一致
```

## 3. 安装配置客户端

```
[root@test ~]# tar xf frp_0.27.0_linux_amd64.tar.gz && cd frp_0.27.0_linux_amd64
```

```
[root@test frp_0.27.0_linux_amd64]# vim frpc.ini

[common]
server_addr = 39.105.62.80   #远程frp服务端的ip地址
server_port = 7000           #远程frp服务端的端口

[tomcat]
type = tcp
local_ip = 127.0.0.1         #要映射到内网机器的目标ip
local_port = 8080            #映射到本地的目标端口
remote_port = 8888           #在远程服务端启动的端口，即访问服务端的8888端口就等于访问本地8080端口服务
```

## 4.分别启动客户端和服务端

服务端

```
[root@shzll ~]# /root/frp_0.27.0_linux_amd64/frps -c /root/frp_0.27.0_linux_amd64/frps.ini &
```

客户端

```
[root@test ~]# /root/frp_0.27.0_linux_amd64/frpc -c /root/frp_0.27.0_linux_amd64/frpc.ini &
```

