# GlusterFS的部署

## 1.概述

- GlusterFS适合存储大文件，小文件性能较差。生产上可以使用GlusterFS+Cinder为Openstack提供块存储，存储虚拟机镜像。
- GlusterFS是Scale-Out存储解决方案Gluster的核心，它是一个开源的分布式文件系统，具有强大的横向扩展能力，通过扩展能够支持数PB存储容量和处理数千客户端。GlusterFS借助TCP/IP或InfiniBand RDMA网络将物理分布的存储资源聚集在一起，使用单一全局命名空间来管理数据。GlusterFS基于可堆叠的用户空间设计，可为各种不同的数据负载提供优异的性能。

## 2.主机环境

| 主机名 | IP地址      | 角色           |
| ------ | ----------- | -------------- |
| node1  | 192.168.2.4 | server、client |
| node2  | 192.168.2.5 | server、client |

## 3.安装并启动

node1、node2都要执行

```
yum install centos-release-gluster -y
```

```
yum install glusterfs-server -y
```

启动

```
systemctl start glusterd
```

## 4.Trusted Storage Pools创建

在开始创建ClusterFS卷之前，需要创建一个称之为Trusted Storage的池，是一个可信的网络存储服务器，可以理解为集群。为卷提供bricks。

把node1当做集群节点，本机不需要添加，把要加入的节点添加即可

- 创建

  ```
  gluster peer probe 192.168.2.5
  ```

- 查看

  ```
  [root@node1 ~]# gluster peer status
  Number of Peers: 1
  
  Hostname: 192.168.2.5
  Uuid: 46b8f1f5-e9dd-4b85-ac96-51d1be263d2f
  State: Peer in Cluster (Connected)
  ```
node2

- 查看

  ```
  [root@node2 data]# gluster peer status
  Number of Peers: 1
  
  Hostname: 192.168.2.4
  Uuid: 69289b57-0a55-4fca-a70e-4b1087b51d50
  State: Peer in Cluster (Connected)
  
  ```

## 5.GlusterFS Volumes创建

### 每个节点各一个volume

GlusterFS的卷共有三种基本类型，可以组合共7种类型，其中Distributed、Replicated、Strped、为基本类型。Distributed Striped、Distributed Replicated、Distributed Striped Replicated、Striped Replicated。为组合类型。

本文为了测试7种卷类型，需要在各个服务器分别创建7个目录。

```
mkdir -p /data/type{1,2,3,4,5,6,7}/exp{1,2,3,4,5,6,7}
```

- 创建volume

```shell
gluster volume create test1-volume 192.168.2.4:/data/type1/exp1/ 192.168.2.5:/data/type1/exp1/ force  #force为强制，强制在根分区创建volume
```

- 查看volume

node1:

```
[root@node1 ~]# gluster volume info test1-volume
 
Volume Name: test1-volume
Type: Distribute
Volume ID: ff107631-c36a-47d5-97a1-42e5cb13c1e0
Status: Started
Snapshot Count: 0
Number of Bricks: 2
Transport-type: tcp
Bricks:
Brick1: 192.168.2.4:/data/type1/exp1
Brick2: 192.168.2.5:/data/type1/exp1
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
```

node2:

```
[root@node2 data]# gluster volume info test1-volume
 
Volume Name: test1-volume
Type: Distribute
Volume ID: ff107631-c36a-47d5-97a1-42e5cb13c1e0
Status: Started
Snapshot Count: 0
Number of Bricks: 2
Transport-type: tcp
Bricks:
Brick1: 192.168.2.4:/data/type1/exp1
Brick2: 192.168.2.5:/data/type1/exp1
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
```

- 启动volume

```
gluster volume start test1-volume
```

-  挂载volume

  安装client(默认已经安装)

  ```
  yum install -y glusterfs-client
  ```

  挂载

  ```
  mkdir -p /mnt/glusterfs/type1 
  ```

  ```
  mount.glusterfs 192.168.2.4:/test1-volume /mnt/glusterfs/type1
  ```

  查看

  ```
  [node1 ~]# df -h
  文件系统                   容量  已用  可用 已用% 挂载点
  /dev/sda3                   19G  1.7G   18G    9% /
  devtmpfs                   2.0G     0  2.0G    0% /dev
  tmpfs                      2.0G     0  2.0G    0% /dev/shm
  tmpfs                      2.0G   12M  2.0G    1% /run
  tmpfs                      2.0G     0  2.0G    0% /sys/fs/cgroup
  /dev/sda1                  197M  102M   96M   52% /boot
  tmpfs                      394M     0  394M    0% /run/user/0
  192.168.2.4:/test1-volume   38G  3.7G   34G   10% /mnt/glusterfs/type1
  ```
- 测试

  在挂载点下touch 8个文件

  ```
  touch /mnt/glusterfs/type1/{1,2,3,4,5,6,7,8}
  ```

  查看在各个节点下volume的文件结构

  node1:

  ```
  [root@node1 type1]# tree /data/type1/exp1
  exp1/
  ├── 1
  ├── 5
  ├── 7
  └── 8
  ```

  node2:

  ```
  [root@node2 ~]# tree /data/type1/exp1
  /data/type1/exp1
  ├── 2
  ├── 3
  ├── 4
  └── 6
  ```
### 每个节点各两个volume

- 创建volume

  ```
  gluster volume create test2-volume 192.168.2.4:/data/type1/exp2 192.168.2.4:/data/type1/exp3 192.168.2.5:/data/type1/exp2 192.168.2.5:/data/type1/exp3 force
  ```

- 查看

  ```
  [root@node2 ~]# gluster volume info test2-volume
   
  Volume Name: test2-volume
  Type: Distribute
  Volume ID: 5207253d-1968-4b6c-9aea-938fcd5f0e1c
  Status: Started
  Snapshot Count: 0
  Number of Bricks: 4
  Transport-type: tcp
  Bricks:
  Brick1: 192.168.2.4:/data/type1/exp2
  Brick2: 192.168.2.4:/data/type1/exp3
  Brick3: 192.168.2.5:/data/type1/exp2
  Brick4: 192.168.2.5:/data/type1/exp3
  Options Reconfigured:
  transport.address-family: inet
  nfs.disable: on
  ```

- 启动volume

  ```
  gluster volume start test2-volume
  ```

- 挂载

  ```
  mkdir -p /mnt/glusterfs/type2
  ```

  ```
  mount.glusterfs 192.168.2.4:/test2-volume /mnt/glusterfs/type2
  ```

- 测试

   同样touch 8个文件

  ```
  touch /mnt/glusterfs/type2/{1,2,3,4,5,6,7,8}
  ```

  查看个节点vloume目录结构

  node1:

  ```
  [root@node1 ~]# tree /data/type1/
  ├── exp2
  │   ├── 1
  │   ├── 5
  │   └── 8
  ├── exp3
  │   └── 7
  ```

  node2:

  ```
  [root@node2 ~]# tree /data/type1/
  ├── exp2
  │   └── 4
  ├── exp3
  │   ├── 2
  │   ├── 3
  │   └── 6
  ```
### 创建一个复制卷

- 创建volume

```shell
gluster volume create test3-volume replica 2 192.168.2.4:/data/type2/exp1  192.168.2.4:/data/type2/exp2 192.168.2.5:/data/type2/exp1 192.168.2.5:/data/type2/exp2 force #replica是说明复制卷，后面2是复制两份,分别在4个brack上分布，如果是4，则每个brick各复制一份
```

- 查看

  ```
  [root@node1 ~]# gluster volume info test3-volume
   
  Volume Name: test3-volume
  Type: Distributed-Replicate
  Volume ID: a54a5956-cd6d-43e7-9ad2-47cfba5485c1
  Status: Created
  Snapshot Count: 0
  Number of Bricks: 2 x 2 = 4
  Transport-type: tcp
  Bricks:
  Brick1: 192.168.2.4:/data/type2/exp1
  Brick2: 192.168.2.4:/data/type2/exp2
  Brick3: 192.168.2.5:/data/type2/exp1
  Brick4: 192.168.2.5:/data/type2/exp2
  Options Reconfigured:
  transport.address-family: inet
  nfs.disable: on
  performance.client-io-threads: off
  ```

- 启动

  ```
  gluster volume start test3-volume
  ```

- 挂载

  ```
  mkdir -p /mnt/glusterfs/type3
  ```

  ```
  mount.glusterfs 192.168.2.4:/test3-volume /mnt/glusterfs/type3/
  ```

- 查看

  ```
  [root@node1 ~]# df -h
  文件系统                   容量  已用  可用 已用% 挂载点
  /dev/sda3                   19G  1.7G   18G    9% /
  devtmpfs                   2.0G     0  2.0G    0% /dev
  tmpfs                      2.0G     0  2.0G    0% /dev/shm
  tmpfs                      2.0G   12M  2.0G    1% /run
  tmpfs                      2.0G     0  2.0G    0% /sys/fs/cgroup
  /dev/sda1                  197M  102M   96M   52% /boot
  tmpfs                      394M     0  394M    0% /run/user/0
  192.168.2.4:/test1-volume   38G  3.7G   34G   10% /mnt/glusterfs/type1
  192.168.2.4:/test2-volume   38G  3.7G   34G   10% /mnt/glusterfs/type2
  192.168.2.4:/test3-volume   19G  1.9G   17G   10% /mnt/glusterfs/type3
  ```

- 测试

  挂载点下touch 4个文件

  ```
  touch /mnt/glusterfs/type3/{1,2,3,4}
  ```

  查看各节点volume的目录结构

  node1:

  ```
  [root@node1 type3]# tree /data/type2/
  /data/type2/
  ├── exp1
  │   └── 1
  ├── exp2
  │   └── 1
  ```

  node2:

  ```
  [root@node2 ~]# tree /data/type2/
  /data/type2/
  ├── exp1
  │   ├── 2
  │   ├── 3
  │   └── 4
  ├── exp2
  │   ├── 2
  │   ├── 3
  │   └── 4
  ```
