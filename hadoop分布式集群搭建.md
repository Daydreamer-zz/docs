# hadoop分布式集群搭建

安装多节点Hadoop-YARN集群与安装单节点（伪分布式模型）的方式类似。集群模型中，主节点的配置方式与单节点相同，除了需要将主机地址改为相应的节点地址；对于从节点来说，将主节点的安装目录直接复制到从节点，并配置好其所需要的目录和环境变量即可。配置完成后，可于主节点启动整个集群，也可手动启动各节点上的相应的服务

## 1.环境准备和主机规划

接下来的示例配置一个有一个主节点和三个从节点的Hadoop-YARN集群 。集群中所用的各节点必须有一个惟一的主机名和IP地址，并能够基于主机互相通信。如果没有配置合用的DNS服务 ，也可以通过/etc/hosts文件进行主机解析，本示例中所用到的hosts文件内容如下，其中node1为主节点，node2、node3和node4为从节点

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.2.4 node1
192.168.2.5 node2
192.168.2.6 node3
192.168.2.7 node4
```

| 主机名 | ip          | 角色   |
| ------ | ----------- | ------ |
| node1  | 192.168.2.4 | master |
| node2  | 192.168.2.5 | slave  |
| node3  | 192.168.2.6 | slave  |
| node4  | 192.168.2.7 | slave  |

## 2.配置主节点(master)

### 配置jdk

```
tar xf jdk-8u191-linux-x64.tar.gz -C /usr/local
```

```
vim /etc/profile
```

```
export JAVA_HOME=/usr/local/jdk1.8.0_191
export JRE_HOME=${JAVA_HOME}/jre  
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib  
export PATH=${JAVA_HOME}/bin:$PATH
```

```
source /etc/profile
```

### 创建目录并解压hadoop文件

```
mkdir -p /bdapps/
tar xf hadoop-2.7.7.tar.gz -C /bdapps/
ln -s /bdapps/hadoop-2.7.7  /bdapps/hadoop
```

### 配置hadoop环境变量

编辑/etc/profile加入

```
export   HADOOP_PREFIX="/bdapps/hadoop"
export   PATH=$PATH:$HADOOP_PREFIX/bin:$HADOOP_PREFIX/sbin
export   HADOOP_COMMON_HOME=${HADOOP_PREFIX}
export   HADOOP_HDFS_HOME=${HADOOP_PREFIX}
export   HADOOP_MAPRED_HOME=${HADOOP_PREFIX}
export   HADOOP_YARN_HOME=${HADOOP_PREFIX}
```

### 创建运行Hadoop进程的用户

```
 groupadd hadoop 
 useradd -g hadoop yarn 
 useradd -g hadoop hdfs
 useradd -g hadoop mapred
```
### 修改配置文件

- /bdapps/hadoop/etc/hadoop/core-site.xml

  ```xml
  <configuration>
    <property>
      <name>fs.defaultFS</name>
      <value>hdfs://node1:8020</value>
      <final>true</final>
    </property>
  </configuration>
  ```

- /bdapps/hadoop/etc/hadoop/yarn-site.xml

  ```xml
  <configuration>
    <property>
      <name>yarn.resourcemanager.address</name>
      <value>node1:8032</value>
    </property>
    <property>
      <name>yarn.resourcemanager.scheduler.address</name>
      <value>node1:8030</value>
    </property>
    <property>
      <name>yarn.resourcemanager.resource-tracker.address</name>
      <value>node1:8031</value>
    </property>
    <property>
      <name>yarn.resourcemanager.admin.address</name>
      <value>node1:8033</value>
    </property>
    <property>
      <name>yarn.resourcemanager.webapp.address</name>
      <value>node1:8088</value>
    </property>
    <property>
      <name>yarn.nodemanager.aux-services</name>
      <value>mapreduce_shuffle</value>
    </property>
    <property>
       <name>yarn.nodemanager.auxservices.mapreduce_shuffle.class</name>
      <value>org.apache.hadoop.mapred.ShuffleHandler</value>
    </property>
    <property>
      <name>yarn.resourcemanager.scheduler.class</name>
      <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
    </property>
  </configuration>
  ```

- /bdapps/hadoop/etc/hadoop/hdfs-site.xml

  修改dfs.replication属性的值为所需要冗余的数值，例如

  ```xml
  <configuration>
    <property>
      <name>dfs.replication</name>
      <value>2</value>
    </property>
    <property>
      <name>dfs.namenode.name.dir</name>
      <value>file:///data/hadoop/hdfs/nn</value>
    </property>
    <property>
      <name>dfs.datanode.data.dir</name>
      <value>file:///data/hadoop/hdfs/dn</value>
    </property>
    <property>
      <name>fs.checkpoint.dir</name>
      <value>file:///data/hadoop/hdfs/snn</value>
    </property>
    <property>
      <name>fs.checkpoint.edits.dir</name>
      <value>file:///data/hadoop/hdfs/snn</value>
    </property>
  </configuration>
  ```

- /bdapps/hadoop/etc/hadoop/mapred-site.xml

  ```xml
  <configuration>
    <property>
      <name>mapreduce.framework.name</name>
      <value>yarn</value>
    </property>
  </configuration>
  ```
- /bdapps/hadoop/etc/hadoop/slave

  ```
  node2
  node3
  node4
  ```
### 创建数据和日志目录

```
mkdir -p /data/hadoop/hdfs/{nn,sn,dn}
chown -R hdfs:hadoop /data/hadoop/hdfs/
mkdir -p /bdapps/hadoop/logs
chmod g+w /bdapps/hadoop/logs
chown -R yarn:hadoop /bdapps/hadoop/*
```
## 3.配置slave节点

slave节点的配置与master节点相同，只是启动的服务不同，因此 ，在各slave节点创建所需的用户  、组和目录、设置好权限并安装hadoop之后后  ，将master节点的各配置文件直接复制到各slave节点hadoop配置文件目录，再为其设置好各环境变量即可。所有slave节点的安装配置过程均相同。

以下步骤均在slave节点操作

### 安装jdk

master节点已安装，具体不做描述

### 创建并解压hadoop文件

```
mkdir -p /bdapps
```

```
tar xf hadoop-2.7.7.tar.gz -C /bdapps
```

```
ln -s /bdapps/hadoop-2.7.7 /bdapps/hadoop
```

### 配置hadoop环境变量

```
export   HADOOP_PREFIX="/bdapps/hadoop"
export   PATH=$PATH:$HADOOP_PREFIX/bin:$HADOOP_PREFIX/sbin
export   HADOOP_COMMON_HOME=${HADOOP_PREFIX}
export   HADOOP_HDFS_HOME=${HADOOP_PREFIX}
export   HADOOP_MAPRED_HOME=${HADOOP_PREFIX}
export   HADOOP_YARN_HOME=${HADOOP_PREFIX}
```

### 创建用户和组

```
groupadd hadoop 
useradd -g hadoop yarn 
useradd -g hadoop hdfs
useradd -g hadoop mapred
```
### 在master节点上把配置文件复制到各slave节点

```
scp /bdapps/hadoop/etc/hadoop/{core-site.xml,hdfs-site.xml,yarn-site.xml,mapred-site.xml} nodeX:bdapps/hadoop/etc/hadoop/
```
### 创建数据和日志目录

一般来说，从节点只用到dn目录

```
mkdir -p /data/hadoop/hdfs/{nn,snn,dn}
chown -R hdfs:hadoop /data/hadoop/hdfs/
mkdir -p /bdapps/hadoop/logs
chmod g+w /bdapps/hadoop/logs
chown -R yarn:hadoop /bdapps/hadoop/*
```

## 4.格式化hdfs

在HDFS集群的NN启动之前需要先初始化其用于存储数据的目录。如果hdfs-site.xml中dfs.namenode.name.dir属性指定的目录不存在，格式化命令会自动创建之；如果事先存在，请确保其权限设置正确，此时格式操作会清除其内部的所有数据并重新建立一个新的文件系统。需要以hdfs用户的身份在master节点执行如下命令

```
su - hdfs -c 'hdfs namenode -format'
```

## 5.启动各节点的服务

master节点需要启动HDFS的NameNode服务 ，以及YARN的ResourceManager服务

```
su - hdfs -c 'hadoop-daemon.sh start namenode'
```

```
su - yarn -c 'yarn-daemon.sh start resourcemanager'
```

各slave节点需要启动HDFS的DataNode服务，以及YARN的NodeManager服务

```
su - hdfs -c 'hadoop-daemon.sh start datanode'
```

```
su - yarn -c 'yarn-daemon.sh start nodemanager'
```

集群规模较大时，分别启动各节点的各服务过于繁琐和低效，为此，hadoop专门提供了start-dfs.sh和stop-dfs.sh来启动及停止整个hdfs集群 ，以及start-yarn.sh和stop-yarn.sh来启动及停止整个yarn集群。

```
 su - hdfs  -c 'start-dfs.s'
```

```
 su - yarn  -c 'start-yarn'
```

## 6.验证

集群启动完成后，可在各节点以jps命令等验正各进程是否正常运行，也可以通过Web  UI来检查集群的运行状态。

master

```
[root@node1 ~]# jps
1618 ResourceManager
1508 NameNode
1879 Jps
```

slave

```
[root@node4 ~]# jps 
1459 Jps
1288 NodeManager
1196 DataNode
```

webUI![](http://39.105.62.80/wp-content/uploads/2019/04/QQ截图20190402165855.png)

![](http://39.105.62.80/wp-content/uploads/2019/04/QQ截图20190402165945.png)