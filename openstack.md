# Openstack

本例主机规划

node1 192.168.1.4

node2 192.168.1.5

## 一、openstack基础环境

### 1.openstack包

- 安装epel源

```
rpm -ivh https://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
```

- 安装openstack仓库

```
yum install -y centos-release-openstack-ocata.noarch
```

- 安装openstack客户端

```
yum install -y python-openstackclient
```

- 安装openstack SELinux管理包

```
yum install -y openstack-selinux
```

### 2.mysql数据库的安装
- 安装软件包
```
yum install -y mariadb-server mariadb python2-PyMySQL
```
- 配置mysql数据库

```
vim /etc/my.cnf.d/openstack.cnf
```
```
[mysqld]
bind-address = 192.168.1.4
default-storage-engine = innodb
innodb_file_per_table
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```
- 启动数据库并设置为开机自启动
```
systemctl enable mariadb.service
systemctl start mariadb.service
```
- 初始化mysql
```
mysql_secure_installation
```
  分别建立cinder,glance,keystone,neutron,nova,nova_api ,nova_cell0库，并完成授权，为localhost和'%' 

```
create database cinder;
grant all on cinder.* to cinder@localhost identified by '199747';
grant all on cinder.* to cinder@'%' identified by '199747';

create database glance;
grant all on glance.* to glance@localhost identified by '199747';
grant all on glance.* to glance@'%' identified by '199747';

create database keystone;
grant all on keystone.* to keystone@localhost identified by '199747';
grant all on keystone.* to keystone@'%' identified by '199747';

create database neutron;
grant all on neutron.* to neutron@localhost identified by '199747';
grant all on neutron.* to neutron@'%' identified by '199747';

create database nova;
grant all on nova.* to nova@localhost identified by '199747';
grant all on nova.* to nova@'%' identified by '199747';

create database nova_api;
grant all on nova_api.* to nova@localhost identified by '199747';
grant all on nova_api.* to nova@'%' identified by '199747';

CREATE DATABASE nova_cell0;
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY '199747';;
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY '199747';
```

### 3.消息队列(rabbitmq)

- 安装包

```
yum install rabbitmq-server -y
```
- 启动rabbitmq并设置为开机自启动
```
systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service
```
- 添加 openstack 用户

```
rabbitmqctl add_user openstack openstack
```
- 给openstack用户配置写和读权限：
```
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```
### 4.memcached服务

- 安装软件包
```
yum install memcached python-memcached -y
```
- 配置memcached
```
vim /etc/sysconfig/memcached
```
```
OPTIONS="-l 192.168.1.4,::1"
```
- 启动Memcached服务，并且配置它随机启动
```
systemctl enable memcached.service
systemctl start memcached.service
```



## 二、keystone验证服务和用户管理服务部署
### 1.安装包

```
yum install openstack-keystone httpd mod_wsgi -y
```

### 2.配置keystone服务

#### 配置文件修改

```
vim /etc/keystone/keystone.conf
```

```
[database]
connection = mysql+pymysql://keystone:199747@192.168.1.4/keystone
[memcache]
servers = 192.168.1.4:11211
[token]
provider = fernet
driver = memcache
```
#### 初始化身份认证服务的数据库
```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

查看是否创建成功

```
mysql -ukeystone -p199747  -h 192.168.1.4 -e 'use keystone;show tables;'
```
#### 初始化Fernet keys

```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
```
```
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```
#### 初始化admin用户

```
keystone-manage bootstrap --bootstrap-password admin \
  --bootstrap-admin-url http://192.168.1.4:35357/v3/ \
  --bootstrap-internal-url http://192.168.1.4:5000/v3/ \
  --bootstrap-public-url http://192.168.1.4:5000/v3/ \
  --bootstrap-region-id RegionOne
```

### 3.配置并启动apache服务器(相当于启动keystone)
- 配置apache
```
vim /etc/httpd/conf/httpd.conf

ServerName 192.168.1.4:80
```
```
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```
- 启动 Apache HTTP 服务并配置其随系统启动
```
systemctl enable httpd.service
systemctl start httpd.service
```

### 4.配置admin账户

```
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://192.168.1.4:35357/v3
export OS_IDENTITY_API_VERSION=3
```

### 5.创建域、项目、用户和角色

#### 创建service项目

```
openstack project create --domain default --description "Service Project" service
```

#### 创建 demo项目和用户

- 创建demo项目：
```
openstack project create --domain default --description "Demo Project" demo
```

- 创建demo用户
```
openstack user create --domain default --password-prompt demo
```

- 创建 user 角色
```
openstack role create user
```
- 把demo用户添加到demo项目,并赋予user的角色
```
openstack role add --project demo --user demo user
```
#### 创建以下服务需要用到的用户

分别添加到service项目(上面已经创建)，并授予admin角色

- glance服务
```
openstack user create --domain default --password-prompt glance
openstack role add --project service --user glance admin
```

- nova服务
```
openstack user create --domain default --password-prompt nova
openstack role add --project service --user nova admin
```
- neutron服务
```
openstack user create --domain default --password-prompt neutron
openstack role add --project service --user neutron admin
```
- cinder服务 
```
openstack user create --domain default --password-prompt cinder
openstack role add --project service --user cinder admin
```

### 6.验证操作

#### 因为安全性的原因，关闭临时认证令牌机制
编辑 /etc/keystone/keystone-paste.ini 文件，从[pipeline:public_api]，[pipeline:admin_api]和[pipeline:api_v3]部分删除admin_token_auth

#### 撤销临时环境变量OS_AUTH_URL和OS_PASSWORD

```
unset OS_AUTH_URL OS_PASSWORD
```

#### 作为admin用户，请求认证令牌

```
openstack --os-auth-url http://192.168.1.4:35357/v3 \
  --os-project-domain-name default --os-user-domain-name default \
  --os-project-name admin --os-username admin token issue
```

#### 作为demo用户，请求认证令牌

```
openstack --os-auth-url http://192.168.1.4:5000/v3 \
  --os-project-domain-name default --os-user-domain-name default \
  --os-project-name demo --os-username demo token issue
```

### 7.创建OpenStack客户端环境脚本

#### admin用户

```
vim admin-openrc
```

```
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_AUTH_URL=http://192.168.1.4:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

```
chmod +x admin-openrc
```

#### demo用户

```
vim demo-openrc
```

```
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=demo
export OS_AUTH_URL=http://192.168.1.4:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

```
chmod +x demo-openrc
```

#### 使用脚本

```
. admin-openrc
```

```
openstack token issue   #这样测试成功，能获得令牌
```

## 三、glance镜像服务

### 1.安装软件包

```
yum install openstack-glance -y
```

### 2.配置数据库和赋予角色以及所属哪个项目，以上已经做了，所以只需要创建glance实体服务和api端点

```
. admin-openrc
```

- glance实体服务

```
openstack service create --name glance --description "OpenStack Image" image
```

- 镜像服务api端点
```
openstack endpoint create --region RegionOne image public http://192.168.1.4:9292
```

```
openstack endpoint create --region RegionOne image internal http://192.168.1.4:9292
```

```
openstack endpoint create --region RegionOne image admin http://192.168.1.4:9292
```

### 3.修改glance配置文件

#### /etc/glance/glance-api.conf

  修改以下部分

```
[database]
connection = mysql+pymysql://glance:199747@192.168.1.4/glance

[keystone_authtoken]
auth_uri = http://192.168.1.4:5000
auth_url = http://192.168.1.4:35357
memcached_servers = 192.168.1.4:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = glance

[paste_deploy]
flavor = keystone

[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

#### /etc/glance/glance-registry.conf

修改以下部分

```
[database]
connection = mysql+pymysql://glance:199747@192.168.1.4/glance

[keystone_authtoken]
auth_uri = http://192.168.1.4:5000
auth_url = http://192.168.1.4:35357
memcached_servers = 192.168.1.4:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = glance

[paste_deploy]
flavor = keystone
```

### 4. 写入镜像服务数据库

实际就是往mysql中glance库建表

```
su -s /bin/sh -c "glance-manage db_sync" glance
```

### 5.启动glane镜像服务并将其配置为随机启动

```
systemctl enable openstack-glance-api.service openstack-glance-registry.service
```

```
systemctl start openstack-glance-api.service openstack-glance-registry.service
```

### 6.验证操作

#### 获得 admin凭证来获取只有管理员能执行的命令的访问权限
```
. admin-openrc
```

#### 下载源镜像
```
wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img
```

#### 使用 QCOW2 磁盘格式,bare 容器格式上传镜像到镜像服务并设置公共可见，这样所有的项目都可以访问它

```
openstack image create "cirros" --file cirros-0.3.5-x86_64-disk.img --disk-format qcow2 --container-format bare --public
```

#### 确认镜像的上传并验证属性
```
openstack image list
```

## 四、nova计算服务

### ①控制节点

#### 1.安装包

```
yum install openstack-nova-api openstack-nova-conductor openstack-nova-console openstack-nova-novncproxy openstack-nova-scheduler openstack-nova-placement-api -y
```

#### 2.创建数据库和用户，及所属项目和赋予admin角色第二部已经操作，这里只需要创建nova服务实体和api端点

- 获得 admin凭证来获取只有管理员能执行的命令的访问权限

```
. admin-openrc
```

- 创建 nova服务实体

```
openstack service create --name nova --description "OpenStack Compute" compute
```

-  创建api端点

```
openstack endpoint create --region RegionOne compute public http://192.168.1.4:8774/v2.1
```

```
openstack endpoint create --region RegionOne compute internal http://192.168.1.4:8774/v2.1
```

```
openstack endpoint create --region RegionOne compute admin http://192.168.1.4:8774/v2.1
```

- 额外需要一个创建placement用户 ，也是以上的操作

```
openstack user create --domain default --password-prompt placement
```

```
openstack role add --project service --user placement admin
```

创建placement服务实体

```
openstack service create --name placement --description "Placement API" placement
```

api端点

```
openstack endpoint create --region RegionOne placement public http://192.168.1.4:8778
```

```
openstack endpoint create --region RegionOne placement internal http://192.168.1.4:8778
```

```
openstack endpoint create --region RegionOne placement admin http://192.168.1.4:8778
```

 #### 3.修改配置文件

##### /etc/nova/nova.conf

需要修改的部分

```
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://guest:guest@192.168.1.4
my_ip = 192.168.1.4
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api_database]
connection = mysql+pymysql://nova:199747@192.168.1.4/nova_api

[database]
connection = mysql+pymysql://nova:199747@192.168.1.4/nova

[api]
auth_strategy = keystone

[keystone_authtoken]
auth_uri = http://192.168.1.4:5000
auth_url = http://192.168.1.4:35357
memcached_servers = 192.168.1.4:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = nova

[vnc]
enabled = true
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip

[glance]
api_servers = http://192.168.1.4:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://192.168.1.4:35357/v3
username = placement
password = placement
```

##### `/etc/httpd/conf.d/00-nova-placement-api.conf`

加入以下内容

```
<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>
```

#### 4.重启apache(keystone)

```
systemctl restart httpd
```

#### 5.同步数据库


##### nova_api

```
su -s /bin/sh -c "nova-manage api_db sync" nova
```

##### cell0

```
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
```

##### cell1

```
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
```
##### nova

```
su -s /bin/sh -c "nova-manage db sync" nova
```


##### 查看是否成功注册

```
nova-manage cell_v2 list_cells
```

#### 6.启动 Compute 服务并将其设置为随系统启动

```
systemctl enable openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
```

```
systemctl start openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
```

### ②计算节点

#### 1.安装包

```
yum install openstack-nova-compute -y 
```

#### 2.修改配置文件

/etc/nova/nova.conf

修改如下部分

```
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://guest:guest@192.168.1.4
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api]
auth_strategy = keystone

[keystone_authtoken]
auth_uri = http://192.168.1.4:5000
auth_url = http://192.168.1.4:35357
memcached_servers = 192.168.1.4:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = nova

[vnc]
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = 192.168.1.5
novncproxy_base_url = http://192.168.1.4:6080/vnc_auto.html

[glance]
api_servers = http://192.168.1.4:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://192.168.1.4:35357/v3
username = placement
password = placement
```

#### 3.启动并设置开机自启动

```
systemctl enable libvirtd.service openstack-nova-compute.service
```

```
systemctl start libvirtd.service openstack-nova-compute.service
```

#### 4.验证操作(在node1控制节点验证)

```
. admin-openrc
```

```
openstack host list
```

或者

```
nova service-list
```

出现node2节点说明计算节点创建成功

## 五、Neutron网络服务

### 控制节点网络服务(node1)

#### 1.安装组件

```
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y
```

#### 2.修改配置文件

网络服务器组件的配置包括数据库、认证机制、消息队列、拓扑变化通知和插件

##### /etc/neutron/neutron.conf

修改以下内容

```
[DEFAULT]
core_plugin = ml2
service_plugins =
transport_url = rabbit://guest:guest@192.168.1.4
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[database]
connection = mysql+pymysql://neutron:199747@192.168.1.4/neutron

[keystone_authtoken]
auth_uri = http://192.168.1.4:5000
auth_url = http://192.168.1.4:35357
memcached_servers = 192.168.1.4:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron

[nova]
auth_url = http://192.168.1.4:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = nova

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

##### /etc/neutron/plugins/ml2/ml2_conf.ini

```
[ml2]
type_drivers = flat,vlan
tenant_network_types =
mechanism_drivers = linuxbridge
extension_drivers = port_security

[ml2_type_flat]
flat_networks = provider

[securitygroup]
enable_ipset = true
```

##### /etc/neutron/plugins/ml2/linuxbridge_agent.ini

```
[linux_bridge]
physical_interface_mappings = provider:ens33

[vxlan]
enable_vxlan = false

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

##### /etc/neutron/dhcp_agent.ini

```
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

##### /etc/neutron/metadata_agent.ini

修改以下部分

```
[DEFAULT]
nova_metadata_ip = 192.168.1.4
metadata_proxy_shared_secret = shz
```

##### /etc/nova/nova.conf

修改以下部分

```
[neutron]
url = http://192.168.1.4:9696
auth_url = http://192.168.1.4:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
service_metadata_proxy = true
metadata_proxy_shared_secret = shz
```

#### 4.同步数据库

网络服务初始化脚本需要一个超链接 /etc/neutron/plugin.ini指向ML2插件配置文件/etc/neutron/plugins/ml2/ml2_conf.ini

```
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

同步数据库

```
 su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

#### 5.重启计算API 服务

```
systemctl restart openstack-nova-api.service
```

#### 6.启动并设置开机自启动

```
systemctl enable neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
```

```
systemctl start neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
```

#### 7.创建服务实体和api端点

```
. admin-openrc
```

- 创建neutron服务实体

```
openstack service create --name neutron --description "OpenStack Networking" network
```

- API端点

```
openstack endpoint create --region RegionOne network public http://192.168.1.4:9696
```

```
openstack endpoint create --region RegionOne network internal http://192.168.1.4:9696
```

```
openstack endpoint create --region RegionOne network admin http://192.168.1.4:9696
```

#### 8.验证

```
neutron agent-list
```

分别看到Metadata agent ，DHCP agent  ， Linux bridge agent，且后面跟笑脸:-)，说明成功

### 计算节点网络服务(node2)

#### 1.安装包

```
yum install openstack-neutron-linuxbridge ebtables ipset -y
```

#### 2.修改配置文件

##### /etc/neutron/neutron.conf

修改如下部分

```
[DEFAULT]
transport_url = rabbit://guest:guest@192.168.1.4
auth_strategy = keystone

[keystone_authtoken]
auth_uri = http://192.168.1.4:5000
auth_url = http://192.168.1.4:35357
memcached_servers = 192.168.1.4:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

##### /etc/neutron/plugins/ml2/linuxbridge_agent.ini

修改以下部分

```
[linux_bridge]
physical_interface_mappings = provider:ens33

[vxlan]
enable_vxlan = false

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

##### /etc/nova/nova.conf

修改以下部分

```
[neutron]
url = http://192.168.1.4:9696
auth_url = http://192.168.1.4:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
```

#### 3.重启计算服务

```
systemctl restart openstack-nova-compute.service
```

#### 4.启动Linuxbridge代理并配置它开机自启动

```
systemctl enable neutron-linuxbridge-agent.service
```

```
systemctl start neutron-linuxbridge-agent.service
```

#### 5.验证（在控制节点node1操作）

```
neutron agent-list
```

出现 node2 后面有笑脸:-) 说明成功

## 六、创建一个云主机测试(控制节点node1操作)

### 1.万事俱备

- mysql：为各个服务提供数据存储
- rabbitmq：为各个服务之间通信提供交通枢纽
- keystone：为各个服务之间通信提供认证和服务注册
- glance：为虚拟机提供镜像管理
- nova：为虚拟机提供计算资源
- neutron：为虚拟机提供网络资源

### 2.创建提供者网络(通过命令)

#### ①在控制节点上，加载 admin凭证来获取管理员能执行的命令访问权限

```
. admin-openrc
```

#### ②创建网络(需要和配置文件中匹配)

```
openstack network create  --share --external --provider-physical-network provider --provider-network-type flat provider
```

查看

```
neutron net-list
```

#### ③创建子网provider-subnet

```
openstack subnet create --network provider --allocation-pool start=192.168.1.100,end=192.168.1.200 --dns-nameserver 223.5.5.5 --gateway 192.168.1.1 --subnet-range 192.168.1.0/24 provider-subnet
```

查看

```
neutron subnet-list
```

### 3.创建m1.nano类型

默认的最小规格的主机需要512 MB内存。对于环境中计算节点内存不足4 GB的，我们推荐创建只需要64 MB的m1.nano规格的主机。若单纯为了测试的目的，请使用m1.nano规格的主机来加载CirrOS镜像

```
openstack flavor create --id 0 --vcpus 1 --ram 64 --disk 1 m1.nano
```

### 4.生成一个键值对

大部分云镜像支持 :term而不是传统的密码登陆。在启动实例前，你必须添加一个公共密钥到计算服务

#### ①导入demo项目凭证

```
. demo-openrc
```

#### ②生产一个ssh-key，并添加秘钥对

```
ssh-keygen -q -N ""
```

```
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
```

会把创建的公钥上传到openstack，并起名mykey

以后创建完虚拟机就ssh登录就不需要输密码，直接就能连上

#### ③验证公钥的添加

```
openstack keypair list
```

### 5.增加安全组规则

添加规则到 default安全组

- 允许icmp协议(ping)

```
openstack security group rule create --proto icmp default
```

- 允许安全 shell (SSH) 的访问

```
openstack security group rule create --proto tcp --dst-port 22 default
```

### 6.启动一个实例

由于之前选的网络选项1，只能在共有网络上创建实例
#### 确定实例选项
- 取得凭证 

```
. demo-openrc
```

- 列出可用类型

```
openstack flavor list
```
- 列出可用镜像

```
openstack image list
```
- 列出可用网络

```
openstack network list
```
- 列出可用的安全组

```
openstack security group list
```
#### 启动云主机

- 启动实例(provider-instance)

```
openstack server create --flavor m1.nano --image cirros --nic net-id=33cae2a9-8c3f-4c23-b889-f501d7aaf6b4 --security-group default --key-name mykey provider-instance
```

- 启动实例状态可能是error解决办法
具体在dashboard的界面是：主机compute没有映射到任何单元

```
解决：添加计算节点到cell数据库：
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```

- 检查实例的状态

```
openstack server list
```

#### 使用虚拟控制台访问实例

- 获取url

```
openstack console url show provider-instance
```

## 七、dashboard仪表盘的部署

### 1.安装包

```
yum install openstack-dashboard -y
```

### 2. 修改配置文件

/etc/openstack-dashboard/local_settings

修改如下部分

```
OPENSTACK_HOST = "192.168.1.4"

ALLOWED_HOSTS = ['*',]

OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True


OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"


OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"


OPENSTACK_NEUTRON_NETWORK = {
    ...
    'enable_router': False,
    'enable_quotas': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    'enable_fip_topology_check': False,
}



TIME_ZONE = "Asia/Shanghai"

```

/etc/httpd/conf.d/openstack-dashboard.conf

```
WSGIApplicationGroup %{GLOBAL}
```

### 3.重启httpd

```
systemctl restart httpd
```

### 4.浏览器登录192.168.1.4/dashboard

### 5.创建自定义镜像

#### 1.用kvm新建镜像

```
qemu-img create -f qcow2 /tmp/centos.qcow2 10G
```

#### 2.创建虚拟机

```
virt-install --virt-type kvm --name centos --ram 1024 --disk /tmp/centos.qcow2,format=qcow2 --network network=default --graphics vnc,listen=0.0.0.0 --noautoconsole --os-type=linux --os-variant=rhel7 --location=/tmp/CentOS-7-x86_64-Minimal-1804.iso
```

#### 3.vnc连接上进行安装系统

#### 4.启动这个kvm虚拟机

- 安装默认需要的软件

 ```
yum install net-tools tree screen wget git vim salt-minion zabbix-agent
 ```

- 编写/tmp/init.sh

```
cp /etc/rc.d/rc.local /tmp
chmod +x /etc/rc.d/rc.local
```

- 修改/etc/rc.d/rc.local 增加

```
/bin/bash /tmp/init.sh
```

### 6.上传镜像到glance

```
openstack image create "CentOS-7.5-x86_64" --file /root/centos.qcow2 --disk-format qcow2 --container-format bare --public
```

### 7.创建相关实例即可

## 八、cinder块存储服务

### 1.安装并配置控制节点

 #### 1.安装包

```
yum install openstack-cinder -y
```

#### 2.修改配置文件

/etc/cinder/cinder.conf

修改如下部分

```
[DEFAULT]
transport_url = rabbit://guest:guest@192.168.1.4
auth_strategy = keystone
my_ip = 192.168.1.4

[database]
connection = mysql+pymysql://cinder:199747@192.168.1.4/cinder

[keystone_authtoken]
auth_uri = http://192.168.1.4:5000
auth_url = http://192.168.1.4:35357
memcached_servers = 192.168.1.4:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = cinder

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```

/etc/nova/nova.conf

```
[cinder]
os_region_name = RegionOne
```

#### 3.创建cinder实体服务和api端点

```
openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
```

```
openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
```

api 端点

```
openstack endpoint create --region RegionOne volumev2 public http://192.168.1.4:8776/v2/%\(project_id\)s
```

```
openstack endpoint create --region RegionOne volumev2 internal http://192.168.1.4:8776/v2/%\(project_id\)s
```

```
openstack endpoint create --region RegionOne volumev2 admin http://192.168.1.4:8776/v2/%\(project_id\)s
```

```
openstack endpoint create --region RegionOne volumev3 public http://192.168.1.4:8776/v3/%\(project_id\)s
```

```
openstack endpoint create --region RegionOne volumev3 internal http://192.168.1.4:8776/v3/%\(project_id\)s
```

```
openstack endpoint create --region RegionOne volumev3 admin http://192.168.1.4:8776/v3/%\(project_id\)s
```

#### 4.同步到数据库

```
su -s /bin/sh -c "cinder-manage db sync" cinder
```

#### 5.完成安装

重启计算API 服务：

```
systemctl restart openstack-nova-api.service
```

启动块设备存储服务，并将其配置为开机自启：

```
systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service
systemctl start openstack-cinder-api.service openstack-cinder-scheduler.service
```

### 2.安装并配置一个存储节点(ISCSI)

#### 1.安装

安装 LVM 包

```
yum install lvm2 -y 
```

启动LVM的metadata服务并且设置该服务随系统启动

```
 systemctl enable lvm2-lvmetad.service
 systemctl start lvm2-lvmetad.service
```

#### 2.创建LVM 物理卷 /dev/sdb(虚拟机环境下需要到vmware添加硬盘)

```
pvcreate /dev/sdb
```

#### 3.创建 LVM 卷组 cinder-volumes

```
vgcreate cinder-volumes /dev/sdb
```

#### 4.修改配置文件

/etc/lvm/lvm.conf

加入如下部分

```
devices {
...
filter = [ "a/sdb/", "r/.*/"]
```

#### 5.安装cinder组件

```
yum install openstack-cinder targetcli python-keystone -y
```

#### 6.修改cinder配置文件

/etc/cinder/cinder.conf

```
[DEFAULT]
transport_url = rabbit://guest:guest@192.168.1.4
auth_strategy = keystone
my_ip = 192.168.1.5
enabled_backends = lvm
glance_api_servers = http://192.168.1.4:9292


[database]
connection = mysql+pymysql://cinder:199747@192.168.1.4/cinder

[keystone_authtoken]
auth_uri = http://192.168.1.4:5000
auth_url = http://192.168.1.4:35357
memcached_servers = 192.168.1.4:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = cinder

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp


[lvm]        #这一部分需要不存在，需要添加
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
iscsi_protocol = iscsi
iscsi_helper = lioadm
```

#### 7.启动并设置开机自启

```
systemctl enable openstack-cinder-volume.service target.service
systemctl start openstack-cinder-volume.service target.service
```

### 3.验证 

```
. admin-openrc
```

```
openstack volume service list
```

### 4.创建一块云硬盘

#### 1.加载demo证书，作为非管理员项目执行下面的步骤

```
. demo-openrc
```

#### 2.创建一个 1 GB 的卷

```
openstack volume create --size 1 volume1
```

#### 3.查看刚创建的卷

```
openstack volume list
```

#### 4.附加卷到实例中（以openstack提供的cirros镜像为例）

这个步骤也可以在dashboard中操作，更简单

```
openstack server add volume INSTANCE_NAME VOLUME_NAME
```

#### 5.ssh或者控制台登录实例，挂载这个硬盘

初始化该云硬盘

```
fdisk /dev/vdb     #剩下按提示操作
```

格式化

```
mke2fs -t ext4 /dev/vdb1   #这里默认序列号是1
```

挂载到文件夹/new

```
mount /dev/vdb1 /new
```

### 5.cinder使用nfs作为后端存储

在node3 192.168.1.6操作

#### 1. 安装包

```
yum install openstack-cinder python-keystone -y
```

```
yum install -y nfs-utils rpcbind
```

#### 2.配置并启动nfs

/etc/exports

```
/data/nfs *(rw,no_root_squash)
```

```
systemctl enable rpcbind nfs
systemctl start rpcbind nfs
```

#### 3.配置cinder使用nfs 

复制node2的cinder配置文件

```
scp /etc/cinder/cinder.conf 192.168.1.6:/etc/cinder/
```

修改配置文件

```
[lvm]项目删除

[DEFAULT]
enabled_backends = nfs



[nfs]
volume_driver = cinder.volume.driver.nfs.NfsDriver
nfs_shares_config = /etc/cinder/nfs_shares
nfs_mount_point_base = $state_path/mnt
```

```
vim /etc/cinder/nfs_shares

192.168.1.6:/data/nfs
```

```
chown root:cinder /etc/cinder/nfs_shares
chmod 640 /etc/cinder/nfs_shares
```

#### 4.启动并设置开机自启动

```
systemctl enable openstack-cinder-volume
systemctl start openstack-cinder-volume
```

#### 5.验证

在node1管理节点上，执行如下命令结果有node3@nfs说明成功

```
openstack volume service list
```

#### 6.创建云盘类型

vim /etc/cinder/cinder.conf

在[lvm]标签中最下添加 node2节点

```
volume_backend_name = ISCSI-Storage
```

```
systemctl restart openstack-cinder-volume
```

vim /etc/cinder/cinder.conf  node3节点

在[nfs]标签中最下添加

```
volume_backend_name = NFS-Storage
```

```
systemctl restart openstack-cinder-volume
```



```
cinder type-create NFS
cinder type-create ISCSI
cinder type-key NFS set volume_backend_name=NFS-Storage
cinder type-key ISCSI set volume_backend_name=ISCSI-Storage
```

#### 7.在dashboard中添加卷选择NFS类型

