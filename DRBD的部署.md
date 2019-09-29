# DRBD的部署 

DRBD(Distributed Relicated Block Device 分布式复制块设备)， 可以解决磁盘单点故障。一般情况下只支持2个节点。

node1 192.168.2.4 主要节点

ndoe2 192.168.2.5 

事先准备一个分区，不要格式化

## 1.安装

```
rpm -ivh https://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
```

```
yum install -y drbd84-utils kmod-drbd84
```

把加载命令放入开机执行命令

```
chmod +x /etc/rc.local && echo "modprobe drbd" >> /etc/rc.local
```

 重启

```
reboot
```

## 2.配置

```
vim /etc/drbd.d/r0.res
```

```
resource r0 { 
on node1 {
  device     /dev/drbd0;
  disk       /dev/sdb1;
  address    192.168.2.4:7898;
  meta-disk  internal;
 }
on node2 {
  device     /dev/drbd0;
  disk       /dev/sdb1;
  address    192.168.2.5:7898;
  meta-disk  internal;
 }
}
```

## 3.在两台机器上分别创建数据块

```
drbdadm create-md r0
```

## 4.两台机器启动drbd服务

```
systemctl start drbd.service
```

## 5.设置主用节点

node1

```
drbdadm -- --overwrite-data-of-peer primary all
```

```
drbdadm primary r0
```

## 6.查看设备

node1

```
fdisk -l
```

出现如下说明成功

```
磁盘 /dev/drbd0：5367 MB, 5367459840 字节，10483320 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
```

## 7.格式化drbd设备并挂载

node1

```
mkfs.ext4 /dev/drbd0
```

```
mkdir -p /drbd
```

```
mount /dev/drbd0 /drbd/
```

查看

```
df -h
```

```
文件系统        容量  已用  可用 已用% 挂载点
/dev/sda3        19G  1.7G   18G    9% /
devtmpfs        476M     0  476M    0% /dev
tmpfs           487M     0  487M    0% /dev/shm
tmpfs           487M  7.6M  479M    2% /run
tmpfs           487M     0  487M    0% /sys/fs/cgroup
/dev/sda1       197M  130M   68M   66% /boot
tmpfs            98M     0   98M    0% /run/user/0
/dev/drbd0      4.8G   20M  4.6G    1% /drbd
```

## 8.测试切换

- node1操作

准备200m测试文件

```
dd if=/dev/zero of=/drbd/testdrbd.tmp bs=10M count=20
```

 卸载磁盘

```
umount /dev/drbd0
```

切换为备节点

```
drbdadm secondary r0
```

- node2操作

设为主节点
```
drbdadm primary r0
```

挂载磁盘

```
mount /dev/drbd0 /mnt
```

然后进入/mnt 查看node1创建的测试文件是否存在



