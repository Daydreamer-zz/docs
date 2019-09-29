# sFastDFS集群配置搭建

## 1.概念

FastDFS 是余庆老师开发的一个开源的高性能分布式文件系统（DFS）。 它的主要功能包括：文件存储，文件同步和文件访问，以及高容量和负载平衡。

FastDFS 系统有三个角色：跟踪服务器(Tracker Server)、存储服务器(Storage Server)和客户端(Client)。

- Tracker Server: 跟踪服务器，主要做调度工作，起到均衡的作用；负责管理所有的storage server和group，每个storage在启动后会连接 Tracker，告知自己所属 group 等信息，并保持周期性心跳。多个Tracker之间是对等关系，不存在单点故障。
- Storage Server: 存储服务器，主要提供容量和备份服务；以 group 为单位，每个 group 内可以有多台 storage server，组内的storage server上的数据互为备份。
- Client:客户端，上传下载数据的服务器

## 2.实例主机规划

本实例集群规划了6台主机，其中node1和node2为tracker，互为主备，node3到node6为storage，其中node3和node4为group1，node和node6为group2

| 主机名 |       角色        |   IP地址    |                     安装应用                     |
| :----: | :---------------: | :---------: | :----------------------------------------------: |
| node1  |     tracker01     | 192.168.2.4 |              FastDFS,libfastcommon               |
| node2  |     tracker02     | 192.168.2.5 |              FastDFS,libfastcommon               |
| node3  | storage01(group1) | 192.168.2.6 | FastDFS,libfastcommon,nginx,fastdfs-nginx-module |
| node4  | storage02(group1) | 192.168.2.7 | FastDFS,libfastcommon,nginx,fastdfs-nginx-module |
| node5  | storage03(group2) | 192.168.2.8 | FastDFS,libfastcommon,nginx,fastdfs-nginx-module |
| node6  | storage04(group2) | 192.168.2.9 | FastDFS,libfastcommon,nginx,fastdfs-nginx-module |

## 3.安装配置tracker主机

### 下载所需包和解决依赖(每个主机上需要执行)

```shell
yum -y install gcc-c++ pcre pcre-devel openssl openssl-devel
```

```shell
wget https://github.com/happyfish100/fastdfs/archive/V5.11.tar.gz
wget https://github.com/happyfish100/libfastcommon/archive/V1.0.39.tar.gz
wget https://github.com/happyfish100/fastdfs-nginx-module/archive/V1.20.tar.gz
```

### 集群所有主机分别创建目录结构

```shell
mkdir -p /fastdfs/{file,tracker,storage}
```

### 在node1和node2 tracker主机上执行

```shell
tar xf libfastcommon-1.0.39.tar.gz && cd libfastcommon-1.0.39/ && ./make.sh && ./make.sh make install
```

```shell
tar xf fastdfs-5.11.tar.gz && cd fastdfs-5.11/ && ./make.sh && ./make.sh install
```

默认安装到/usr/bin下，配置文件在/etc/fdfs下

```shell
[root@node6 ~]# ll /etc/fdfs/
total 24
-rw-r--r-- 1 root root 1461 Sep  8 19:13 client.conf.sample
-rw-r--r-- 1 root root 7945 Sep  8 20:18 storage.conf
-rw-r--r-- 1 root root  105 Sep  8 19:13 storage_ids.conf.sample
-rw-r--r-- 1 root root 7389 Sep  8 19:13 tracker.conf.sample
```

### 配置tracker

**在node1和node2上都要执行**

```shell
cp /etc/fdfs/tracker.conf.sample /etc/fdfs/tracker.conf
```

```shell
vim /etc/fdfs/tracker.conf
```

主要修改如下部分

```
port=22122                  #tracker服务器端口（默认22122,一般不修改）
base_path=/fastdfs/tracker  #fdfs_storaged服务存储日志和数据的根目录
store_lookup=0             #设置为0上传数据在group1和group2轮询(默认即可，只是这个地方需要特别注意)
store_group=group2          #只有当store_lookup=1是这项才会生效，本例不修改，默认即可
```

### 分别启动node1和node2的tracker

```shell
/etc/init.d/fdfs_trackerd start
```

### 查看端口

```shell
[root@node1 ~]# netstat -tunlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:22122           0.0.0.0:*               LISTEN  6192/fdfs_trackerd  
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      754/sshd            
tcp6       0      0 :::22                   :::*                    LISTEN      754/sshd 
```

## 4.安装配置storage主机

### 编译安装

在node3,node4,node5,node6上执行

```shell
tar xf libfastcommon-1.0.39.tar.gz && cd libfastcommon-1.0.39/ && ./make.sh && ./make.sh make install
```

```shell
tar xf fastdfs-5.11.tar.gz && cd fastdfs-5.11/ && ./make.sh && ./make.sh install
```

###   配置group1即node3,node4

```shell
cp /etc/fdfs/storage.conf.sample /etc/fdfs/storage.conf
```

```
vim /etc/fdfs/storage.conf
```

主要修改如下部分

```
group_name=group1
port=23000
base_path=/fastdfs/storage
store_path0=/fastdfs/file
tracker_server=192.168.2.4:22122
tracker_server=192.168.2.5:22122
http.server_port=80                #配置nginx模块的时候访问的端口
```

### 配置group2即node5,node6

和node3-4的配置大部分相同，只需要改`group_name=group2`即可

### 启动node3-6的fdfs_storaged服务

```shell
/etc/init.d/fdfs_storaged start
```

## 5.查看集群信息

随便找一台storage主机执行，看到每个storage标签有个ACTIVE即为成功

```
fdfs_monitor /etc/fdfs/storage.conf
```

```
[2019-09-08 21:32:17] DEBUG - base_path=/fastdfs/storage, connect_timeout=30, network_timeout=60, tracker_server_count=2, anti_steal_token=0, anti_steal_secret_key length=0, use_connection_pool=0, g_connection_pool_max_idle_time=3600s, use_storage_id=0, storage server id count: 0

server_count=2, server_index=0

tracker server is 192.168.2.4:22122

group count: 2

Group 1:
group name = group1
disk total space = 50971 MB
disk free space = 48421 MB
trunk free space = 0 MB
storage server count = 2
active server count = 2
storage server port = 23000
storage HTTP port = 8888
store path count = 1
subdir count per path = 256
current write server index = 1
current trunk file id = 0

	Storage 1:
		id = 192.168.2.6
		ip_addr = 192.168.2.6 (node3)  ACTIVE
		http domain = 
		version = 5.11
		join time = 2019-09-08 20:18:20
		up time = 2019-09-08 20:18:20
		total storage = 50971 MB
		free storage = 48421 MB
		upload priority = 10
		store_path_count = 1
		subdir_count_per_path = 256
		storage_port = 23000
		storage_http_port = 8888
		current_write_path = 0
		source storage id = 
		if_trunk_server = 0
		connection.alloc_count = 256
		connection.current_count = 1
		connection.max_count = 2
		total_upload_count = 1
		success_upload_count = 1
		total_append_count = 0
		success_append_count = 0
		total_modify_count = 0
		success_modify_count = 0
		total_truncate_count = 0
		success_truncate_count = 0
		total_set_meta_count = 0
		success_set_meta_count = 0
		total_delete_count = 0
		success_delete_count = 0
		total_download_count = 0
		success_download_count = 0
		total_get_meta_count = 0
		success_get_meta_count = 0
		total_create_link_count = 0
		success_create_link_count = 0
		total_delete_link_count = 0
		success_delete_link_count = 0
		total_upload_bytes = 1024694
		success_upload_bytes = 1024694
		total_append_bytes = 0
		success_append_bytes = 0
		total_modify_bytes = 0
		success_modify_bytes = 0
		stotal_download_bytes = 0
		success_download_bytes = 0
		total_sync_in_bytes = 0
		success_sync_in_bytes = 0
		total_sync_out_bytes = 0
		success_sync_out_bytes = 0
		total_file_open_count = 1
		success_file_open_count = 1
		total_file_read_count = 0
		success_file_read_count = 0
		total_file_write_count = 4
		success_file_write_count = 4
		last_heart_beat_time = 2019-09-08 21:31:55
		last_source_update = 2019-09-08 20:39:19
		last_sync_update = 1970-01-01 08:00:00
		last_synced_timestamp = 1970-01-01 08:00:00 
	Storage 2:
		id = 192.168.2.7
		ip_addr = 192.168.2.7  ACTIVE
		http domain = 
		version = 5.11
		join time = 2019-09-08 20:18:30
		up time = 2019-09-08 20:18:30
		total storage = 50971 MB
		free storage = 48421 MB
		upload priority = 10
		store_path_count = 1
		subdir_count_per_path = 256
		storage_port = 23000
		storage_http_port = 8888
		current_write_path = 0
		source storage id = 192.168.2.6
		if_trunk_server = 0
		connection.alloc_count = 256
		connection.current_count = 1
		connection.max_count = 1
		total_upload_count = 0
		success_upload_count = 0
		total_append_count = 0
		success_append_count = 0
		total_modify_count = 0
		success_modify_count = 0
		total_truncate_count = 0
		success_truncate_count = 0
		total_set_meta_count = 0
		success_set_meta_count = 0
		total_delete_count = 0
		success_delete_count = 0
		total_download_count = 0
		success_download_count = 0
		total_get_meta_count = 0
		success_get_meta_count = 0
		total_create_link_count = 0
		success_create_link_count = 0
		total_delete_link_count = 0
		success_delete_link_count = 0
		total_upload_bytes = 0
		success_upload_bytes = 0
		total_append_bytes = 0
		success_append_bytes = 0
		total_modify_bytes = 0
		success_modify_bytes = 0
		stotal_download_bytes = 0
		success_download_bytes = 0
		total_sync_in_bytes = 1024694
		success_sync_in_bytes = 1024694
		total_sync_out_bytes = 0
		success_sync_out_bytes = 0
		total_file_open_count = 1
		success_file_open_count = 1
		total_file_read_count = 0
		success_file_read_count = 0
		total_file_write_count = 4
		success_file_write_count = 4
		last_heart_beat_time = 2019-09-08 21:32:07
		last_source_update = 1970-01-01 08:00:00
		last_sync_update = 2019-09-08 20:39:24
		last_synced_timestamp = 2019-09-08 20:39:20 (-1s delay)

Group 2:
group name = group2
disk total space = 50971 MB
disk free space = 48422 MB
trunk free space = 0 MB
storage server count = 2
active server count = 2
storage server port = 23000
storage HTTP port = 8888
store path count = 1
subdir count per path = 256
current write server index = 1
current trunk file id = 0

	Storage 1:
		id = 192.168.2.8
		ip_addr = 192.168.2.8  ACTIVE
		http domain = 
		version = 5.11
		join time = 2019-09-08 20:18:39
		up time = 2019-09-08 20:18:39
		total storage = 50971 MB
		free storage = 48422 MB
		upload priority = 10
		store_path_count = 1
		subdir_count_per_path = 256
		storage_port = 23000
		storage_http_port = 8888
		current_write_path = 0
		source storage id = 
		if_trunk_server = 0
		connection.alloc_count = 256
		connection.current_count = 1
		connection.max_count = 2
		total_upload_count = 1
		success_upload_count = 1
		total_append_count = 0
		success_append_count = 0
		total_modify_count = 0
		success_modify_count = 0
		total_truncate_count = 0
		success_truncate_count = 0
		total_set_meta_count = 0
		success_set_meta_count = 0
		total_delete_count = 0
		success_delete_count = 0
		total_download_count = 0
		success_download_count = 0
		total_get_meta_count = 0
		success_get_meta_count = 0
		total_create_link_count = 0
		success_create_link_count = 0
		total_delete_link_count = 0
		success_delete_link_count = 0
		total_upload_bytes = 29184
		success_upload_bytes = 29184
		total_append_bytes = 0
		success_append_bytes = 0
		total_modify_bytes = 0
		success_modify_bytes = 0
		stotal_download_bytes = 0
		success_download_bytes = 0
		total_sync_in_bytes = 0
		success_sync_in_bytes = 0
		total_sync_out_bytes = 0
		success_sync_out_bytes = 0
		total_file_open_count = 1
		success_file_open_count = 1
		total_file_read_count = 0
		success_file_read_count = 0
		total_file_write_count = 1
		success_file_write_count = 1
		last_heart_beat_time = 2019-09-08 21:32:16
		last_source_update = 2019-09-08 20:44:47
		last_sync_update = 1970-01-01 08:00:00
		last_synced_timestamp = 1970-01-01 08:00:00 
	Storage 2:
		id = 192.168.2.9
		ip_addr = 192.168.2.9  ACTIVE
		http domain = 
		version = 5.11
		join time = 2019-09-08 20:18:52
		up time = 2019-09-08 20:18:52
		total storage = 50971 MB
		free storage = 48422 MB
		upload priority = 10
		store_path_count = 1
		subdir_count_per_path = 256
		storage_port = 23000
		storage_http_port = 8888
		current_write_path = 0
		source storage id = 
		if_trunk_server = 0
		connection.alloc_count = 256
		connection.current_count = 1
		connection.max_count = 1
		total_upload_count = 0
		success_upload_count = 0
		total_append_count = 0
		success_append_count = 0
		total_modify_count = 0
		success_modify_count = 0
		total_truncate_count = 0
		success_truncate_count = 0
		total_set_meta_count = 0
		success_set_meta_count = 0
		total_delete_count = 0
		success_delete_count = 0
		total_download_count = 0
		success_download_count = 0
		total_get_meta_count = 0
		success_get_meta_count = 0
		total_create_link_count = 0
		success_create_link_count = 0
		total_delete_link_count = 0
		success_delete_link_count = 0
		total_upload_bytes = 0
		success_upload_bytes = 0
		total_append_bytes = 0
		success_append_bytes = 0
		total_modify_bytes = 0
		success_modify_bytes = 0
		stotal_download_bytes = 0
		success_download_bytes = 0
		total_sync_in_bytes = 29184
		success_sync_in_bytes = 29184
		total_sync_out_bytes = 0
		success_sync_out_bytes = 0
		total_file_open_count = 1
		success_file_open_count = 1
		total_file_read_count = 0
		success_file_read_count = 0
		total_file_write_count = 1
		success_file_write_count = 1
		last_heart_beat_time = 2019-09-08 21:31:58
		last_source_update = 1970-01-01 08:00:00
		last_sync_update = 2019-09-08 20:44:52
		last_synced_timestamp = 2019-09-08 20:44:47 (0s delay)
```

## 6.上传文件测试

配置client.conf

```shell
cp /etc/fdfs/client.conf.sample /etc/fdfs/client.conf
```

```shell
vim /etc/fdfs/client.conf
```

主要修改入下部分

```
base_path=/fastdfs/tracker
tracker_server=192.168.2.4:22122
tracker_server=192.168.2.5:22122
http.tracker_server_port=80
```

上传测试文件

```shell
[root@node1 ~]# fdfs_upload_file /etc/fdfs/client.conf test.txt 
group1/M00/00/00/wKgCB111Bn2APEKoAAAAAAAAAAA002.txt
```

这样说明文件上传到了group1的路径下，查看

```shell
[root@node3 ~]# ll  /fastdfs/file/data/00/00/
total 0
-rw-r--r-- 1 root root 0 Sep  8 21:47 wKgCB111Bn2APEKoAAAAAAAAAAA002.txt
```

## 7.storage主机安装fastdfs-nginx-module和nignx

### 解压并编译安装

**node3,node4,node5,node6都要装**

```shell
tar xf fastdfs-nginx-module-1.20.tar.gz && tar xf nginx-1.15.4.tar.gz
```

```shell
cd nginx-1.15.4/ && ./configure --prefix=/usr/local/nginx --add-module=/root/fastdfs-nginx-module-1.20/src/ && make && make install
```

**关于make报错的处理**

不明原因如下报错

```shell
In file included from /root/fastdfs-nginx-module-1.20/src//common.c:26:0,
                 from /root/fastdfs-nginx-module-1.20/src//ngx_http_fastdfs_module.c:6:
/usr/include/fastdfs/fdfs_define.h:15:27: fatal error: common_define.h: No such file or directory
```

查阅得知如下解决

```
修改fastdfs-nginx-module-1.20/src/config文件，修改如下： 
ngx_module_incs="/usr/include/fastdfs /usr/include/fastcommon/"   第6行
CORE_INCS="$CORE_INCS /usr/include/fastdfs /usr/include/fastcommon/"  第15行
然后重新./configure && make，就可以了
```

### 拷贝配置文件

```shell
cd fastdfs-nginx-module-1.20/src && cp mod_fastdfs.conf /etc/fdfs/
```

```shell
cd fastdfs-5.11/conf && cp anti-steal.jpg http.conf mime.types storage_ids.conf /etc/fdfs/
```

### 修改配置文件

- group1即node3和node4主要修改如下配置

  ```shell
  vim /etc/fdfs/mod_fastdfs.conf
  ```

  ```
  connect_timeout=10
  load_fdfs_parameters_from_tracker=true
  tracker_server=192.168.2.4:22122
  tracker_server=192.168.2.5:22122
  group_name=group1     #区分group1还是group2重要配置
  store_path_count=1  
  store_path0=/fastdfs/file
  group_count = 2
  
  
  #以下直接添加即可
  [group1]
  group_name=group1
  storage_server_port=23000
  store_path_count=1
  store_path0=/fastdfs/file
  [group2]
  group_name=group2
  storage_server_port=23000
  store_path_count=1
  store_path0=/fastdfs/file
  ```

- group2即node5和node6主要修改如下配置

  ```
  vim /etc/fdfs/mod_fastdfs.conf
  ```

  ```
  connect_timeout=10
  load_fdfs_parameters_from_tracker=true
  tracker_server=192.168.2.4:22122
  tracker_server=192.168.2.5:22122
  group_name=group2     #区分group1还是group2重要配置
  store_path_count=1  
  store_path0=/fastdfs/file
  group_count = 2
  
  
  #以下直接添加即可
  [group1]
  group_name=group1
  storage_server_port=23000
  store_path_count=1
  store_path0=/fastdfs/file
  [group2]
  group_name=group2
  storage_server_port=23000
  store_path_count=1
  store_path0=/fastdfs/file
  ```

- 所有storage节点nginx配置

  ```shell
  vim /usr/local/nginx/conf/nginx.conf
  ```

  ```nginx
  server {
          listen       80;
          server_name  localhost;
          location ~/group([0-9])/M00 {
              ngx_fastdfs_module;
          }
  
      }
  ```

### 所有strorage做软连接

  ```shell
  ln -s /fastdfs/file/data/ /fastdfs/file/data/M00
  ```

## 8.启动所有storage节点的nginx服务

注意出现**ngx_http_fastdfs_set**说明配置成功

```shell
[root@node3 ~]# /usr/local/nginx/sbin/nginx 
ngx_http_fastdfs_set pid=1490
```

## 9.上传文件测试

```shell
[root@node1 ~]# fdfs_upload_file /etc/fdfs/client.conf thedivision2_screenshot1.jpg 
group1/M00/00/00/wKgCBl11mNyAbtKqAFIf2OTXcw0393.jpg
```

在浏览器访问任意storage节点

http://192.168.2.6/group1/M00/00/00/wKgCBl11mNyAbtKqAFIf2OTXcw0393.jpg

![](http://39.105.62.80/wp-content/uploads/2019/09/test.jpg)

能访问当刚才上传的图片说明整个FastDFS集群配置完成