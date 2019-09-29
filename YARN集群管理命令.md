# YARN集群管理命令

 yarn命令有许多子命令，大体可分为用户命令和管理命令两类。直接运行yarn命令，可显示其简单使用语法及各子命令的简单介绍

```
[yarn@node1 ~]$ yarn
Usage: yarn [--config confdir] [COMMAND | CLASSNAME]
  CLASSNAME                             run the class named CLASSNAME
 or
  where COMMAND is one of:
  resourcemanager -format-state-store   deletes the RMStateStore
  resourcemanager                       run the ResourceManager
  nodemanager                           run a nodemanager on each slave
  timelineserver                        run the timeline server
  rmadmin                               admin tools
  sharedcachemanager                    run the SharedCacheManager daemon
  scmadmin                              SharedCacheManager admin tools
  version                               print the version
  jar <jar>                             run a jar file
  application                           prints application(s)
                                        report/kill application
  applicationattempt                    prints applicationattempt(s)
                                        report
  container                             prints container(s) report
  node                                  prints node report(s)
  queue                                 prints queue information
  logs                                  dump container logs
  classpath                             prints the class path needed to
                                        get the Hadoop jar and the
                                        required libraries
  cluster                               prints cluster information
  daemonlog                             get/set the log level for each
                                        daemon

Most commands print help when invoked w/o parameters.
```

这些命令中，jar、application、node、logs、classpath和version是常用的用户命令，而resourcemanager、nodemanager、proxyserver、rmadmin和daemonlog是较为常用的管理类命令

## 用户命令

用户命令为Hadoop-YARN客户端命令。这些客户端根据yarn-site.xml中配置的参数连接至YARN服务，并按需运行指定的命令

- jar

  jar命令通过YARN代码运行jar文件，即向RM提交YARN应用程序。运行时，其会启动RunJar类的主方法，检查参数列表并校验jar文件，而后展开jar文件并运行之。例如，前面章节中演示的运行mapreduce示例程序pi，用Monte   Carlo方法估算Pi（π）值

  ```
  yarn jar /bdapps/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.7.jar pi 16 1000
  ```

- application

  管理yarn中的各application，其 --help选项可获取命令的使用帮助

  -list：获取application列表，默认只显示活动的，有两个可选的子选项-appTypes和-appStates

  ```
  [yarn@node1 ~]$ yarn application -list -appStates=all
  19/04/02 20:44:54 INFO client.RMProxy: Connecting to ResourceManager at node1/192.168.2.4:8032
  Total number of applications (application-types: [] and states: [NEW, NEW_SAVING, SUBMITTED, ACCEPTED, RUNNING, FINISHED, FAILED, KILLED]):2
                  Application-Id	    Application-Name	    Application-Type	      User	     Queue	             State	       Final-State	       Progress	                       Tracking-URL
  application_1554206354749_0001	          word count	           MAPREDUCE	      hdfs	   default	          FINISHED	         SUCCEEDED	           100%	http://node2:19888/jobhistory/job/job_1554206354749_0001
  application_1554206354749_0002	     QuasiMonteCarlo	           MAPREDUCE	      hdfs	   default	          FINISHED	         SUCCEEDED	           100%	http://node2:19888/jobhistory/job/job_1554206354749_0002
  ```

  

  -status：获取指定application的状态信息

  ```
  [yarn@node1 ~]$ yarn application -status application_1554206354749_0002
  19/04/02 20:27:22 INFO client.RMProxy: Connecting to ResourceManager at node1/192.168.2.4:8032
  Application Report : 
  	Application-Id : application_1554206354749_0002
  	Application-Name : QuasiMonteCarlo
  	Application-Type : MAPREDUCE
  	User : hdfs
  	Queue : default
  	Start-Time : 1554207869086
  	Finish-Time : 1554207965528
  	Progress : 100%
  	State : FINISHED
  	Final-State : SUCCEEDED
  	Tracking-URL : http://node2:19888/jobhistory/job/job_1554206354749_0002
  	RPC Port : 43228
  	AM Host : node2
  	Aggregate Resource Allocation : 1572910 MB-seconds, 1420 vcore-seconds
  	Diagnostics : 
  ```

  -kill：停止已经提交的或运行的application

- node

  -list：yarn集群内的node列表，可结合-states子选项过滤节点，而-all子选项用于控制显示所有节点。常用的状态有NEW,    RUNNING,UNHEALTHY,DECOMMISSIONED,LOST和REBOOTED

  ```
  [yarn@node1 ~]$ yarn node -list
  19/04/02 20:33:50 INFO client.RMProxy: Connecting to ResourceManager at node1/192.168.2.4:8032
  Total Nodes:3
           Node-Id	     Node-State	Node-Http-Address	Number-of-Running-Containers
       node2:39435	        RUNNING	       node2:8042	                           0
       node4:35042	        RUNNING	       node4:8042	                           0
       node3:35984	        RUNNING	       node3:8042	                           0
  ```

  -status NodeID：以节点报告格式打印指定节点的信息。后跟node:id

  ```
  [yarn@node1 ~]$ yarn node -status node2:39435
  19/04/02 20:35:13 INFO client.RMProxy: Connecting to ResourceManager at node1/192.168.2.4:8032
  Node Report : 
  	Node-Id : node2:39435
  	Rack : /default-rack
  	Node-State : RUNNING
  	Node-Http-Address : node2:8042
  	Last-Health-Update : 星期二 02/四月/19 08:34:07:72CST
  	Health-Report : 
  	Containers : 0
  	Memory-Used : 0MB
  	Memory-Capacity : 8192MB
  	CPU-Used : 0 vcores
  	CPU-Capacity : 8 vcores
  	Node-Labels :
  ```

- logs

  用于从已经完成的YARN应用程序(即状态为FAILED、KILLED或FINISHED)上获取日志信息 。 。不过 ，如果需要通过命令行查看日志，需要为YARN集群启用log-aggregation属性 ，在yarn-site.xml配置文件中定义yarn.log-aggregation-enable属性的值为true即可 

  -applicationId applicationID：必备选项，用于从resourcemanager获取详细信息

  ```
  yarn logs -applicationId application_1554206354749_0002
  ```

- classpath

  用于显示YARN集群CLASSPATH的值

  ```
  [yarn@node1 ~]$ yarn classpath
  /bdapps/hadoop/etc/hadoop:/bdapps/hadoop/etc/hadoop:/bdapps/hadoop/etc/hadoop:/bdapps/hadoop/share/hadoop/common/lib/*:/bdapps/hadoop/share/hadoop/common/*:/bdapps/hadoop/share/hadoop/hdfs:/bdapps/hadoop/share/hadoop/hdfs/lib/*:/bdapps/hadoop/share/hadoop/hdfs/*:/bdapps/hadoop/share/hadoop/yarn/lib/*:/bdapps/hadoop/share/hadoop/yarn/*:/bdapps/hadoop/share/hadoop/mapreduce/lib/*:/bdapps/hadoop/share/hadoop/mapreduce/*:/contrib/capacity-scheduler/*.jar:/bdapps/hadoop/share/hadoop/yarn/*:/bdapps/hadoop/share/hadoop/yarn/lib/*
  ```

- version

  顾名思义不解释

## 管理命令

- rmadmin

  rmadmin是ResourceManager的客户端程序，可用于刷新访问控制策略、调度器队列及注册到RM上的节点等。刷新之后结果无需重启集群服务即可生效

  常用选项：

  -help：获取命令帮助；

  -refreshQueues：重载队列的acl、状态及调用器队列；它会根据配置文件中的配置信息重新初始化调用器；

  -refreshNodes：为RM刷新主机信息，它通过读取RM节点的include和exclude文件来更新集群需要包含或排除的节点列表；

  -refreshUserToGroupsMappings：根据配置的Hadoop安全组映射，通过刷新组缓存中的信息来更新用户和组之间的映射关系；

  -refreshSuperUserGroupsConfiguration：刷新超级用户代理组映射，以及更新代理主机和core-site.xml配置文件中hadoop.proxyuser属性定义的代理组；

  -refreshAdminAcls：根据yarn站点配置文件或默认配置文件中的yarn.admin.acl属性刷新RM的管理ACL

  -refreshServiceAcl：重载服务级别授权策略文件，而后RM将重载授权策略文件；它会检查Hadoop安全授权是否启用并为IPC  Server、ApplicationMaster、Client和Resourcetracker刷新ACL；

- DaemonLog