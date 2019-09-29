# cobbler 自动安装

## 1.安装包

```
yum install -y cobbler cobbler-web dhcp xinetd pykickstart httpd
```

## 2.配置

修改以下配置文件

/etc/cobbler/settings

```
vim /etc/cobbler/settings
```

```
next_server: 192.168.2.4
server: 192.168.2.4
manage_dhcp: 1
```

/etc/xinetd.d/tftp

```
vim /etc/xinetd.d/tftp
```

```
disable                 = no
```

/etc/cobbler/dhcp.template

```
vim /etc/cobbler/dhcp.template
```

```
subnet 192.168.2.0 netmask 255.255.255.0 {
     option routers             192.168.2.1;
     option domain-name-servers 8.8.8.8;
     option subnet-mask         255.255.255.0;
     range dynamic-bootp        192.168.2.100 192.168.2.254;
```

## 3.启动并解决报错

```
systemctl start httpd cobblerd
```

```
cobbler check
```

解决报错

- ```
  cobbler get-loaders
  ```

- 替换/etc/cobbler/settings的`default_password_crypted`部分为以下命令结果

  ```
  openssl passwd -1 -salt 'root' '199747'
  ```

- ```
  systemctl enable rsyncd && systemctl start rsyncd
  ```
再执行`cobbler check`

## 4.导入镜像

- 挂载iso文件

```
mount -o loop CentOS-7-x86_64-DVD-1804.iso /iso
```

- 导入cobbler

```
cobbler import --path=/iso/ --name=CentOS7-full --arch=x86_64
```

- 指定kickstart文件

```
cobbler profile edit  --name=CentOS7-full-x86_64 --kickstart=/var/lib/cobbler/kickstarts/centos7.ks
```

这里提供一个可用的kickstart 文件

centos7.ks

指定完后要执行`cobbler sync`不然不会生效

- 修改内核参数，让网卡命名为`eth0`

  ```
  cobbler profile edit  --name=CentOS7-full-x86_64 --kopts='net.ifnames=0 biosdevname=0'
  ```
