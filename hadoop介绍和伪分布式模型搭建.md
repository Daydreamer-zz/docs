# hadoop

## 简介

hadoop= HDFS+MapReduce 即：底层的分布式存储平台+能够结合分布式存储平台完成分布式程序的运行和数据分布式处理的框架。之所以叫hadoop是因为作者儿子的玩具小象hadoop

HBase :  hadoop的database

缺陷：MapReduce是批处理程序

HDFS分布式文件存储是有中心节点的分布式文件系统，用户要访问完整数据必须通过中心节点

HDFS包括：NN：NameNode(名称节点、中心节点)

​                     DN：DateNode(数据节点)

MRv1(hadoop1)---->MRv2(hadoop2)

MRv1：Cluser resource manager ,Data processing

MRv2：

​	YARN：Cluster resource manager

​	MRv2：Data processing

​		    MR：batch(批处理)

​		    Tez：execution engine



​		     RM：resource manager

​		     NM：node manager

​		     AM：application manager

​		     container：mr任务 

client提交一个任务，请求到resource manager，resource manager询问每个节点 的node manager是否有空闲容器来运行程序，有的话找一个节点来启动程序主控进程叫application master，程序需要在node上启动几个mapper或reducer由application master向resource manager申请(resource request)，resouce manager根据application master的请求在节点上创建所请求的数量的容器，所以application master就可以使用这些container来运行作业。

每个container在运行过程中不断向application master反馈作业任务，一旦任务完成，application master也会报告给resource manager

## hadoop伪分布式集群搭建

hadoop伪分布式，就是所有节点都位于一个主机上，仅做为测试用和学习，不能用于真实生产环境

### 1.安装hadoop并设置其所需的环境变量

- 设置好JAVA_HOME变量，而后解压安装包至指定目录下 

```
mkdir -p /bdapps/
```
```
tar xf hadoop-2.7.7.tar.gz -C /bdapps/
```
```
ln -s /bdapps/hadoop-2.7.7  /bdapps/hadoop
```

- 编辑环境配置文件/etc/profile.d/hadoop.sh，定义类似如下环境环境变量，设定Hadoop的运行环境

```
export   HADOOP_PREFIX="/bdapps/hadoop"
export   PATH=$PATH:$HADOOP_PREFIX/bin:$HADOOP_PREFIX/sbin
export   HADOOP_COMMON_HOME=${HADOOP_PREFIX}
export   HADOOP_HDFS_HOME=${HADOOP_PREFIX}
export   HADOOP_MAPRED_HOME=${HADOOP_PREFIX}
export   HADOOP_YARN_HOME=${HADOOP_PREFIX}
```

### 2.创建运行Hadoop进程的用户和相关目录

- 创建用户和组：出于安全等目的，通常需要用特定的用户来运行hadoop不同的守护进程，例如 ，以hadoo   p为组，分别用三个用户yarn、hdfs和mapred来运行相应的进程。

```
 groupadd hadoop 
 useradd -g hadoop yarn 
 useradd -g hadoop hdfs
 useradd -g hadoop mapred
```

- 创建数据和日志目录:

  Hadoop需要不同权限的数据和日志目录，这里以/data/hadoop/hdfs为hdfs数据存储目录。

  ```
  mkdir -p /data/hadoop/hdfs/{nn,sn,dn}
  chown -R hdfs:hadoop /data/hadoop/hdfs/
  mkdir -p /var/log/hadoop/yarn
  ```

  而后，在hadoop的安装目录中创建logs目录，并修改hadoop所有文件的属主和属组

  ```
  cd /bdapps/hadoop/
  mkdir -p logs
  chmod g+w logs
  chown -R yarn:hadoop ./*
  ```

### 3.配置hadoop

- /bdapps/hadoop/etc/hadoop/core-site.xml

  core-site.xml文件包含了NameNode主机地址以及其监听RPC端口等信息，对于伪分布式模型的安装来说，其主机地址为localhost。NameNode默认使用的RPC端口为8020。其简要的配置内容如下所示

  ```xml
  <configuration>
    <property>
      <name>fs.defaultFS</name>
      <value>hdfs://localhost:8020</value>
      <final>true</final>
    </property>
  </configuration>
  ```

- /bdapps/hadoop/etc/hadoop/hdfs-site.xml

  hdfs-site.xml主要用于配置HDFS相关的属性，例如复制因子（即数据块的副本数）、NN和DN用于存储数据的目录等。数据块的副本数对于伪分布式的Hadoop应该为1，而NN和DN用于存储的数据的目录为前面的步骤中专门为其创建的路径。另外 ，前面的步骤中也为SNN创建了相关的目录，这里也一并配置其为启用状态。

  ```xml
  <configuration>
    <property>
      <name>dfs.replication</name>
      <value>1</value>
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

  注意，如果需要其它用户对hdfs有写入权限，还需要在hdfs-site.xml添加一项属性定义。

  ```xml
  <property>
    <name>dfs.permissions</name>
    <value>false</value>
  </property>
  ```

- /bdapps/hadoop/etc/hadoop/mapred-site.xml

  mapred-site.xml文件用于配置集群的MapReduceframework，此处应该指定使用yarn，另外的 可 用 值 还 有local和classic。mapred-site.xml默 认 不 存 在 ， 但 有 模 块 文件mapred-site.xml.template，只需要将其复制mapred-site.xml即可。

  ```xml
  <configuration>
    <property>
      <name>mapreduce.framework.name</name>
      <value>yarn</value>
    </property>
  </configuration>
  ```

- /bdapps/hadoop/etc/hadoop/yarn-site.xml

  yarn-site.xml用于配置YARN进程及YARN的相关属性。首先需要指定ResourceManager守护进程的主机和监听的端口，对于伪分布式模型来讲，其主机为localhost，默认的端口为8032；其次需要指定ResourceManager使用的scheduler，以及NodeManager的辅助服务。一个简要的配置示例如下所示。
```xml
<configuration>
  <property>
    <name>yarn.resourcemanager.address</name>
    <value>localhost:8032</value>
  </property>
  <property>
    <name>yarn.resourcemanager.scheduler.address</name>
    <value>localhost:8030</value>
  </property>
  <property>
    <name>yarn.resourcemanager.resource-tracker.address</name>
    <value>localhost:8031</value>
  </property>
  <property>
    <name>yarn.resourcemanager.admin.address</name>
    <value>localhost:8033</value>
  </property>
  <property>
    <name>yarn.resourcemanager.webapp.address</name>
    <value>localhost:8088</value>
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

- etc/hadoop/hadoop-env.sh和etc/hadoop/yarn-env.sh

  Hadoop的各守护进程依赖于JAVA_HOME环境变量，如果有类似于前面步骤中通过/etc/profile.d/java.sh全局配置定义的JAVA_HOME变量即可正常使用。不过，如果想为Hadoop定义依赖到的特定JAVA环境，也可以编辑这两个脚本文件，为其JAVA_HOME取消注释并配置合适的值即可。此外 ，Hadoop大多数守护进程默认使用的堆大小为1GB，但现实应用中，可能需要对其各类进程的堆内存大小做出调整，这只需要编辑此两者文件中的相关环境变量值即可，例如HADOOP_HEAPSIZE、HADOOP_JOB_HISTORY_HEAPSIZE、JAVA_HEAP_SIZE和YARN_HEAP_SIZE等。

- slaves文件

  slaves文件存储于了当前集群的所有slave节点的列表，对于伪分布式模型，其文件内容仅应该为localhost，这也的确是这个文件的默认值。因此 ，伪分布式模型中，此文件的内容保持默认即可。

###    4.格式化HDFS

在HDFS的NN启动之前需要先初始化其用于存储数据的目录。如果hdfs-site.xml中dfs.namenode.name.dir属性指定的目录不存在，格式化命令会自动创建之；如果事先存在，请确保其权限设置正确，此时格式操作会清除其内部的所有数据并重新建立一个新的文件系统。需要以hdfs用户的身份执行如下命令

```
su hdfs
```

```
 hdfs namenode –format
```

出现`INFO common.Storage: Storage directory /data/hadoop/hdfs/nn has been successfully formatted.`说明成功格式化

### 5.启动hadoop

Hadoop2的启动等操作可通过其位于sbin路径下的专用脚本进行：

NameNode：hadoop-daemon.sh (start|stop) namenode

DataNode：hadoop-daemon.sh (start|stop) datanode 

SecondaryNameNode：hadoop-daemon.sh (start|stop) secondarynamenode

ResourceManager：yarn-daemon.sh (start|stop) resourcemanager

NodeManager：yarn-daemon.sh (start|stop) nodemanage

- 启动HDFS服务

  ```
  [hdfs@node1 ~]$ hadoop-daemon.sh start namenode
  starting namenode, logging to /bdapps/hadoop/logs/hadoop-hdfs-namenode-node1.out
  ```

  ```
  [hdfs@node1 ~]$ hadoop-daemon.sh start  secondarynamenode
  starting secondarynamenode, logging to /bdapps/hadoop/logs/hadoop-hdfs-secondarynamenode-node1.out
  ```

  ```
  [hdfs@node1 ~]$ hadoop-daemon.sh start datanode
  starting datanode, logging to /bdapps/hadoop/logs/hadoop-hdfs-datanode-node1.out
  ```

  上述三个命令均在执行完成后给出了一个日志信息保存指向的信息，但是 ，实际用于保存日志的文件是以“.log”为后缀的文件，而非以“.out”结尾。可通过日志文件中的信息来判断进程启动是否正常完成。如果所有进程正常启动，可通过jdk提供的jps命令来查看相关的java进程状态

  ```
  [hdfs@node1 ~]$ jps 
  1745 NameNode
  1895 DataNode
  1991 Jps
  1837 SecondaryNameNode
  ```

- 启动YARN服务

  YARN有两个守护进程：resoucemanager和nodemanager，它们都可通过yarn-daemon.sh脚本启动或停止。以yarn用户执行相关的命令即可，如下所示

  ```
  su yarn
  ```

  ```
  [yarn@node1 ~]$ yarn-daemon.sh start resourcemanager
  starting resourcemanager, logging to /bdapps/hadoop/logs/yarn-yarn-resourcemanager-node1.out
  ```

  ```
  [yarn@node1 ~]$ yarn-daemon.sh start nodemanager
  starting nodemanager, logging to /bdapps/hadoop/logs/yarn-yarn-nodemanager-node1.out
  ```

### 6.WebUI预览 

HDFS和YARNResourceManager各自提供了一个Web接口 ，通过这些接口可检查HDFS集群以及YARN集群的相关状态信息。它们的访问接口分别为如下所求，具体使用中，需要将NameNodeHost和RresouceManagerHost分别改为其相应的主机地址。 HDFS-NameNode http://192.168.2.4:50070/ ![](http://39.105.62.80/wp-content/uploads/2019/04/QQ截图20190401233851.png)YARN-ResourceManager http://192.168.2.4:8088/，这里由于resourcemanager监听在127.0.0.1:8088，无法从外部访问，可以通过本机访问(前提是安装了图形界面)，或者nginx反向代理到本机8088端口![](http://39.105.62.80/wp-content/uploads/2019/04/QQ截图20190401235035.png)

### 7.运行测试

先往hdfs上传一个测试文件

整个测试要切换到hdfs用户

```
su hdfs
```

```
hdfs dfs -put /etc/fstab /fatab
```

```
yarn jar /bdapps/hadoop/share/hadoop/mapreducehadoop-mapreduce-examples-2.7.7.jar wordcount /fstab /fstab.out
```

输出效果如下

```
19/04/02 00:17:57 INFO client.RMProxy: Connecting to ResourceManager at localhost/127.0.0.1:8032
19/04/02 00:17:58 INFO input.FileInputFormat: Total input paths to process : 1
19/04/02 00:17:59 INFO mapreduce.JobSubmitter: number of splits:1
19/04/02 00:17:59 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1554131441520_0003
19/04/02 00:18:00 INFO impl.YarnClientImpl: Submitted application application_1554131441520_0003
19/04/02 00:18:00 INFO mapreduce.Job: The url to track the job: http://node1:8088/proxy/application_1554131441520_0003/
19/04/02 00:18:00 INFO mapreduce.Job: Running job: job_1554131441520_0003
19/04/02 00:18:11 INFO mapreduce.Job: Job job_1554131441520_0003 running in uber mode : false
19/04/02 00:18:11 INFO mapreduce.Job:  map 0% reduce 0%
19/04/02 00:18:19 INFO mapreduce.Job:  map 100% reduce 0%
19/04/02 00:18:26 INFO mapreduce.Job:  map 100% reduce 100%
19/04/02 00:18:27 INFO mapreduce.Job: Job job_1554131441520_0003 completed successfully
19/04/02 00:18:27 INFO mapreduce.Job: Counters: 49
	File System Counters
		FILE: Number of bytes read=591
		FILE: Number of bytes written=246535
		FILE: Number of read operations=0
		FILE: Number of large read operations=0
		FILE: Number of write operations=0
		HDFS: Number of bytes read=593
		HDFS: Number of bytes written=433
		HDFS: Number of read operations=6
		HDFS: Number of large read operations=0
		HDFS: Number of write operations=2
	Job Counters 
		Launched map tasks=1
		Launched reduce tasks=1
		Data-local map tasks=1
		Total time spent by all maps in occupied slots (ms)=5109
		Total time spent by all reduces in occupied slots (ms)=3997
		Total time spent by all map tasks (ms)=5109
		Total time spent by all reduce tasks (ms)=3997
		Total vcore-milliseconds taken by all map tasks=5109
		Total vcore-milliseconds taken by all reduce tasks=3997
		Total megabyte-milliseconds taken by all map tasks=5231616
		Total megabyte-milliseconds taken by all reduce tasks=4092928
	Map-Reduce Framework
		Map input records=11
		Map output records=54
		Map output bytes=625
		Map output materialized bytes=591
		Input split bytes=92
		Combine input records=54
		Combine output records=38
		Reduce input groups=38
		Reduce shuffle bytes=591
		Reduce input records=38
		Reduce output records=38
		Spilled Records=76
		Shuffled Maps =1
		Failed Shuffles=0
		Merged Map outputs=1
		GC time elapsed (ms)=275
		CPU time spent (ms)=2440
		Physical memory (bytes) snapshot=444149760
		Virtual memory (bytes) snapshot=4207095808
		Total committed heap usage (bytes)=288882688
	Shuffle Errors
		BAD_ID=0
		CONNECTION=0
		IO_ERROR=0
		WRONG_LENGTH=0
		WRONG_MAP=0
		WRONG_REDUCE=0
	File Input Format Counters 
		Bytes Read=501
	File Output Format Counters 
		Bytes Written=433
```





