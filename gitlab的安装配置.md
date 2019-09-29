# gitlab的安装配置

## 1. 安装

解决依赖

```
yum install -y curl policycoreutils openssh-server openssh-client postfix
```

```
systemctl start sshd postfix
```

下载rpm包

```shell
wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-10.0.0-ce.0.el7.x86_64.rpm
```

## 2.配置

修改

```
vim /etc/gitlab/gitlab.rb


external_url http://39.105.62.80:9999
```

生效 

```
gitlab-ctl reconfigure
```

## 3.常用命令

```
gitlab-ctl status
gitlab-ctl start
gitlab-ctl stop
gitlab-ctl restart
gitlab-ctl tail  #查看某个组件的日志
```

## 4.主要目录

/var/opt/git;ab/git-data/repositories/root   #库默认保存位置

/var/opt/gitlab       #执行gitlab-ctl reconfigure命令后编译后的应用数据和配置文件

/etc/gitlab                #gitlab配置文件目录

/var/log/gitlab          #gitlab各个组件生成的日志

/var/opt/gitlab/backups   #备份文件生成的目录

## 5.gitlab的备份和恢复 

配置

```
vim /etc/gitlab/gitlab.rb

gitlab_rails['backup_path']='/data/backup/gitlab'  #备份路径
gitlab_rails['backup_archive_permissions'] = 0644  #生成备份文件的权限
gitlab_rails['backup_keep_time']=60480             #保留7天
```

生效

```
gitlab-ctl reconfigure
```

创建备份目录和修改权限

```
mkdir -p /data/backup/gitlab
chown -R git:git /data/backup/gitlab
```

备份命令 

```
gitlab-rake gitlab:backup:create
```

恢复

```
gitlab-ctl stop unicorn
gitlab-ctl stop sidekiq #停止相关数据连接服务
gitlab-rake gitlab:backup:restore BACKUP=XXXXX  #xxxx可到备份目录查看，有具体时间戳
```

最后再次启动Gitlab

```
gitlab-ctl start
```

恢复命令完成后，可以check检查一下恢复情况

```
gitlab-rake gitlab:check SANITIZE=true
```

