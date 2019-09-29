# 自动化运维工具ansible

## 1.安装

```
yum install -y ansible cowsay
```

## 2.配置

```
vim /etc/ansible/hosts
```

将要管理的主机加入配置文件，前提要做好ssh-key秘钥登录，这里不做描述

```
[nodes]
192.168.2.4
192.168.2.5
192.168.2.6
```

## 3.常用模块

`ping`模块：检测被控主机是否能ping通

```shell
ansible nodes -m ping  #ansible后加主机可以是all、在hosts文件配置的标签如nodes、或者ip，-m指定模块
```

`command`模块：执行命令，只能执行简单的命令，无法解释特殊符号，如管道|,统配*等等，不指定模块默认是执行此模块

```shell
ansible all -m command -a "ifconfig"  #-a后指定动作
```

`shell`模块：

```shell
ansible all -m shell -a "hostname >/tmp/hostname.txt"
```

`copy`模块：把管理机的文件复制到被控主机

```shell
ansibe all -m copy -a "src=/scripts/lnmp.sh dest=/root"
```

```shell
ansibe all -m copy -a "src=/scripts/lnmp.sh dest=/root/LNMP.sh owner=nobody group=nobody mode=700" 
```

`script`模块：相当于结合`shell`模块和`copy`模块，先把脚本传到服务器上再执行

```shell
ansible all -m script -a "/scripts/lnmp.sh"
```

`file`模块：修改文件用户，组，权限，路径，创建目录或文件

要指定path，state(directory|touch|link)

```shell
ansible all -m file -a "path=/www state=directory"
```

`yum`模块：指定包名，版本state有：present,latest

```shell
ansible all -m yum -a "name=nginx state=present"
```

`cron`模块：定时任务，相当于`vi /var/spool/cron/root`

```shell
ansible all -m cron -a 'name="backup etc" minute=00 hour=00 job="tar zcf /tmp/backup.tar.gz /backup/* >/dev/null 2>&1" state=present'
```

删除某个定时任务，指定state为adsent即可

```shell
ansible all -m cron -a 'name="backup etc" state=absent'
```

## 3.playbook剧本

```
vim /etc/ansible/xxx.yml
```

```yaml
---
  - hosts: all
    task:
      - name: show hostname
        command: hostname
```

执行

```shell
ansible-playbook -C /etc/ansible/xxx.yml  #检测playbook语法是否正确
```

```shell
ansible-playbook /etc/ansible/xxx.yml
```

添加定时任务，如cron.yml

```yaml
---
  - hosts: all
    tasks:
      - name: add cron
        cron: 
          name: "backup etc" 
          minute: 00 
          hour: 00 
          job: "tar zcf /tmp/backup.tar.gz /backup/* >/dev/null 2>&1" 
          state: present
```



## 4.absible注册变量
在playbook里使用变量，使用vars定义好后，用连个花括号表示引用`{{}}`

```yaml
---
  - hosts: all
    vars:
      file: shz.txt
      dir: /root/
    tasks:
      - name: touch file
        file: path={{dir}}/{{file}} state=touch
```
使用系统命令作为变量

```yaml
---
  - hosts: all
    tasks:
      - name: get ip address
        shell: hostname -I
        register: ip
      - name: print ip var to file
        shell: echo {{ip.stdout}} >/tmp/ip.txt
```

如下实例一个打包备份配置文件的playbook

```yaml
---
  - hosts: all
    tasks:
      - name: get ip
        shell: hostname -I
        register: ip
      - name: get date
        shell: date +%F
        register: date
      - name: mkdir 
        file: path=/backup/{{ip.stdout}} state=directory
      - name: tar
        shell: tar zcf /backup/etc-{{ip.stdout}}-{{date.stdout}}.tar.gz /etc/*
```

如何调试变量

`debug`模块：msg={{xxx}}

```yaml
---
  - hosts: all
    tasks:
      - name: get ip
        shell: hostname -I
        register: ip
      - name: debug test
        debug: msg={{ip}}
```

然后直接执行即可，不需要`-C`检查错误

## 5.ansible循环和判断

循环

```yaml
---
  - hosts: all
    tasks:
      - name: show ip
        shell: echo 192.168.2.{{item}} >/tmp/test1.txt
        with_items:
          - 4
          - 5
          - 6
```

条件，when指定主机名 `ansible_hostname`叫做ansible内置变量

```yaml
---
  - hosts: all
    tasks:
      - name: install nfs
        yum: name=nfs-utils,rpcbind state=present
        when: ( ansible_hostname == "node3" )
```

查看ansible所有内置变量

```
ansible 192.168.2.5 -m setup
```





