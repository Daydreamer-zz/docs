# inotify+rsync实现文件实时同步备份

## 1.rsync

### 工作端口：873

### 特性

1.支持实时拷贝特殊文件

2.有排除指定文件或目录的功能

3.保持源文件或目录的权限，时间，软硬链接，属主，属组均不改变

4.实现增量同步，即只同步变化的数据，因此传输效率很高

5.可使用rcp,rsh,ssh等方式传输文件

6.可通过socket方式传输文件和数据

7.支持匿名或认证的进程模式传输

### rsync工作方式 

单个主机间的数据传输

```shell
rsync -vzitopg  --delete /etc/hosts /backup   #类似cp，-vzitopg是保持属性，--delete让目标和源目录数据一致(危险操作)
```

通过rcp和ssh通道传输数据

```shell
raync -avz /etc/hosts -e 'ssh -p 22' root@192.168.2.4:/backup
```

以守护进程方式传输(rsync自身的重要功能)

### rsync服务端配置

配置文件实例`/etc/rsyncd.conf`

```
uid = root
gid = root
use chroot = no
max connections = 200
pid file = /var/run/rsyncd.pid
transfer logging = yes
timeout = 300
lock file = /var/run/rsync.lock
log file = /var/log/rsync.log
host allow = 192.168.2.0/24
[backup]
path = /backup
ignore error                          #忽略错误
read only = false                     #可写
list = false                          #不能列表
auth_user = rsync_backup              #指定同步时的虚拟用户
secrets file = /etc/rsync.passwd      #指定虚拟用户的密码文件
```

密码文件的实例/etc/rsync.passwd

```
echo 'rsync_backup:199747' >/etc/rsync.passwd
```

```
chmod 600 /etc/rsync.passwd
```

创建备份实例目录

```
mkdir -pv /backup
chown -R rsync:rsync /backup
```

启动

```
systemctl start rsync
```

### rsync客户端配置

创建密码文件

```
echo "199747" >/etc/rsync.passwd
chmod 600 /etc/rsync.passwd
```

将本地目录推送到服务器

```shell
rsync -avz /backup rsync_backup@192.168.2.4::backup --password-file=/etc/rsync.passwd
```

## 2.inotify

可以配合备份工具实现实时备份的监控目录变化的工具，通过监控某个目录文件变动再执行rsync命令

### 安装

```shell
yum install -y inotify-tools
```

### 功能介绍

inotifywait：在被监控的文件或者目录上等待文件系统事件(open,close,delete等)发生，执行后处于阻塞状态，适合shell脚本

inotifywatch：收集被监控的文件系统使用度统计数据

### inotifywait使用

```shell
inotifywait -mrq --timefmt '%d/%m/%y %H:%M' --format '%T%w%f' -e close_write,delete /backup
```

解释

-r 递归查询目录

-q 仅打监控事件的信息

-m 始终保持事件监听状态

--excludei 排除文件或者目录时，不区分大小写

--timefmt 指定事件输出的格式

--format 打印使用指定的输出格式的字符串

-e   指定要监控的事件，如access,modify,attrib,close,open,move_to,move,create,delete,unmount

### 配合rsync写成脚本完成实时备份

```shell
#!/bin/bash
/usr/bin/inotifywait -mrq --format '%w%f' -e close_write,delete /backup\
|while read file
  do
    rsync -az /backup --delete rsync_backup@192.168.2.4::backup --password-file=/etc/rsync.passwd
  done
```

### inotify调优

```shell
echo "50000000" >/proc/sys/fs/inotify/max_user_watches
echo "50000000" >/proc/sys/fs/inotify/max_queued_events
```











