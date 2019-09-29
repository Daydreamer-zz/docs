## docker
### 1.什么是docker
- 三大理念：构建，运输，运行
- 容器
- 运行环境和代码都带
### 2.组成(c/s架构)
- docker client
- docker server
### 3.组件
- 镜像(image)
- 容器(container)
- 仓库(repository)
### 4.docker能干什么

- 面向产品：产品交付
- 面向开发：简化环境配置
- 面向测试：多版本测试
- 面向运维：环境一致性
- 面向架构：自动化扩容（微服务）

### 5. 安装启动docker
```shell
yum install -y docker
systemctl start docker
```
安装docker-ce

```
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

```
yum install -y docker-ce
```

### 6.docker实际操作

```
docker pull centos #获取名为centos的镜像
docker search centos  #搜索镜像
docker images     #查看已经装好的镜像
docker rmi "nameID"   #删除某个镜像
docker load < nginx.tar  #从本地文件中导入镜像
```
##### 启动doker容器
```
docker run centos /bin/bash 'hello world'
```

##### docker运行完就退出了，要查看所有docker容器状态

```
docker ps -a
```

##### 启动一个自定义名称的容器

```
docker run --name mydocker -t -i centos /bin/bash
```

<!--启动一个name为mydocker的容器，"-t"让docker分配伪终端，"-i"打开docker标准输入输出,centos为镜像名称-->

##### 启动刚才退出的容器mydocker

```
docker start mydocker
```

- ###### 进入刚才启动的容器
```
docker attache mydocker #此方法退出后容器还是会退出，不实用
```
- ###### 进入刚才的容器法2
```shell
docker inspect  -f "{{ .State.Pid}}" mydocker   #获取mydocker容器的pid，例2505
nsenter -t 2505 -m -u -i -n -p    #此方法进入后退出后mysocker容器还会运行
```
此方法可以写成脚本如下：docker_in.sh
```shell
#!/bin/bash
#use nsenter to access docker
docker_in(){
   NAME_ID=$1
   PID=`docker inspect  -f "{{ .State.Pid}}" $NAME_ID`
   nsenter -t $PID -m -u -i -n -p
}
docker_in $1
```
- ###### 不进入已经后台启动的docker容器(mydocker)执行命令,如`whoami`
```
docker exec mydocker whoami 
docker exec mydocker -i -t /bin/bash  #这样也可以进入容器，退出也不会导致容器退出
```
##### 删除容器
```
docker rm mydocker
docker rm -f mydocker    #强制删除一个正在运行的容器
docker run --rm centos /bin/echo "haha"   #直接运行一个容器，获取命令结果停止后直接删除不除，不会在docker ps -a 历史中存在
```
### 7.docker网络访问(以nginx镜像为例子)

随机端口映射

```
docker run -d -P nginx   #"-P“随机映射某个端口，“-d"后台运行，
```

```
423f0f38052a        nginx               "nginx -g 'daemon ..."   21 seconds ago      Up 20 seconds               0.0.0.0:32768->80/tcp   vibrant_wiles     #将本地的32768端口映射到nginx容器的80端口，直接访问物理机的32768端口即可打开nignx默认首页
```

```
iptables -t nat -vnL      #查看nat表的映射
```

```
docker logs 423f0f38052a   # 查看日志，423f0f38052a是本例nginx容器的id
```

指定端口映射

```
docker run -d -p 192.168.1.4:81:80 --name mynginx nginx #将本机的81端口映射到nginx容器的80端口
```

```
docker port mynginx     #单独查看端口映射
```

### 8.docker数据管理

将本地的目录挂载到容器中，容器的数据就会写入到本地的目录

- 不指定本机的目录挂载

```
docker run -d -P --name nginx-volume-test1 -v /data nginx  #启动容器并将一个本地/data目录挂载
docker inspect -f {{.Mounts}} nginx-volume-test1 #查看容器挂载的本地目录实际路径

```

```
[{volume 0e34a2e2759b6548ea5edb768dbbf41a76f8adbe0637440771c60dc626104020 /var/lib/docker/volumes/0e34a2e2759b6548ea5edb768dbbf41a76f8adbe0637440771c60dc626104020/_data /data local  true }] #返回结果为
```

即容器中/data目录实际在主机的/var/lib/docker/volumes/0e34a2e2759b6548ea5edb768dbbf41a76f8adbe0637440771c60dc626104020/_data 中

- 指定本地的一个目录挂载

```
mkdir -p /data/docker-volume-nginx  #创建一个测试目录
docker run -d --name nginx-volume-test2  -v /data/docker-volume-nginx/:/data nginx    # “-v”指定 本机源:容器目标
```

### 9.docker镜像构建(nginx为例)

```
docker kill $(docker ps -a -q) #关闭所有运行的容器
docker rm $(docker ps -a -q)   #删除所有容器
```

##### 手动构建docker镜像

- 启动容器并安装相关app

```
docker run --name mynginx -it centos
[root@99497e43fd33 /]#rpm -ivh https://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm            #进入centos容器后配置epel源
[root@99497e43fd33 /]# install -y nginx         #然后手动安装nignx
```

想让该容器退出后不停止，必须把应用设置为前台运行，所以修改nginx配置文件设置为前台启动

```
[root@99497e43fd33 /]# vim /etc/nginx/nginx.conf
```

```
daemon off;     #加入该项目
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
.........
```

退出容器 

```
[root@99497e43fd33 /]# exit
```

- 提交修改

```
docker commit -m "my nginx" 99497e43fd33  shz/mynginx:v1  #提交修改到本地，99497e43fd33是容器的id,"shz"是仓库名称,"mynginx是镜像名称"，“V1”是标签
```
查看刚才的提交镜像`docker images`结果为

```
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
shz/mynginx         v1                  050be9aaf69c        7 seconds ago       847 MB
docker.io/nginx     latest              62f816a209e6        6 days ago          109 MB
docker.io/centos    latest              75835a67d134        4 weeks ago         200 MB
```

- 启动刚做的镜像

```
docker run --name mynginxv1 -d -p 81:80 shz/mynginx:v1  nginx #必须带镜像库名/镜像名:标签,最后面运行的命令
```

##### 用Dockerfile做镜像
```
mkdir -p /opt/dockerfile/nginx   #新建一个测试目录
```

- 编写Dockerfile
```
vim /opt/dockerfile/nginx/Dockerfile
```

```shell
#This Dockerfile

#Base images
FROM centos

#Maintainer
MAINTAINER XXXX

#Commands
RUN rpm -ivh https://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
RUN yum install -y nginx $$ yum clean all
RUN echo "daemon off;" >>/etc/nginx/nginx.conf
ADD index.html /usr/share/nginx/html/index.html
EXPOSE 80 
CMD ["nginx"]
```
- 执行
```
docker build -t mynginx:v2 .     #/opt/dockerfile/nginx下执行，会自动执行Dockerfile
```

- 查看
```
docker images
```
```
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
mynginxv2           latest              da4ffd4ac801        36 seconds ago      437 MB
shz/mynginx         v1                  050be9aaf69c        30 minutes ago      847 MB
docker.io/nginx     latest              62f816a209e6        6 days ago          109 MB
docker.io/centos    latest              75835a67d134        4 weeks ago         200 MB
```

- 启动刚创建的镜像
```
docker run --name mynginxv2 -d -p 82:80 mynginxv2  #后面不加执行命令的原因是做镜像的过程已经加进去了，也可以说Dockerfile中的CMD项已经覆盖了该命令
```
### 10.生产实际Dockfile设计
##### 目录结构设计
```
/docker/
├── app
│   ├── xxx-admin
│   └── xxx-api
├── runtime
│   ├── java
│   ├── php
│   └── python
└── system
    ├── centos
    ├── centos-ssh
    └── ubuntu
```

##### 用Dockerfile制作基础系统docker镜像

```
vim /docker/system/centos/Dockerfile
```

```shell
# Docker for CentOS

#Base image
FROM centos

#Who
MAINTAINER shz shz1997@hotmail.com

#EPEL
RUN  rpm -ivh https://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm

#Base pkg
RUN yum install -y wget mysql-devel supervisor git redis tree net-tools sudo psmisc && yum clean all

```
开始制作
```
docker build -t shz/centos:base /docker/system/centos
```

##### 制作ssh登录的docker系统镜像

```
vim /docker/system/centos-ssh/Dockerfile
```

```shell
# Docker for CentOS

#Base image
FROM centos

#Who
MAINTAINER Jason.Zhao xxx@gmail.com

#EPEL
RUN rpm -ivh https://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm

#Base pkg
RUN yum install -y openssh-clients openssl-devel openssh-server wget mysql-devel supervisor git redis tree net-tools sudo psmisc && yum clean all

# For SSHD
RUN ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key
RUN ssh-keygen -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key
RUN echo "root:199747" | chpasswd
EXPOSE 22
```

开始制作

```
docker build -t shz/centos-ssh /docker/system/centos-ssh
```

启动镜像测试ssh登录

- 启动该容器

```
docker run --name centos-ssh-test -d -p 8022:22 shz/centos-ssh /usr/sbin/sshd -D
```

- 查看端口
```
netstat -tulnp

tcp6  0   0 :::8022    :::*   LISTEN   6944/docker-proxy-c  #出现此列说明成功
```

- 连接 (xshell也可以直接连接192.168.1.5的8022端口)
```
ssh 127.0.0.1:8022
```