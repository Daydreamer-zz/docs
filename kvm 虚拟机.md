## 一、安装kvm虚拟机

- 安装相关工具:
```
yum install -y qemu-kvm qemu-kvm-tools libvirt virt-install
```

- 启动并设置开机自启动:
```
systemctl start libvirtd
systemctl enable libvirtd
```
- 创建虚拟机“磁盘”:
```
qemu-img create -f raw /opt/CentOS-7-x86_64.raw 10g
```
- 开始安装该虚拟机（包含参数）:
```
virt-install --virt-type kvm --name CentOS-7-x86_64 --ram 2048 --cdrom=/opt/CentOS-7-x86_64-DVD-1804.iso --disk path=/opt/CentOS-7-x86_64.raw --network network=default --graphics vnc,listen=0.0.0.0 --noautoconsole
```
- 使用vnc工具连接上 `192.168.1.6` 默认vnc端口是`5600`，开始安装向导，分区项目应只分根分区"/" 

## 二、管理kvm虚拟机
- 列出当前虚拟机:
```
virsh list --all
```
- 启动该虚拟机:
```
virsh start CentOS-7-x86_64
```
- 强行停止该虚拟机 :
```
virsh destroy CentOS-7-x86_64
```
- 删除该虚拟机:
```
virsh undefine CentOS-7-x86_64
```
## 三、修改kvm虚拟机相关配置(cpu，内存，磁盘，网络等) 
- **修改默认vml配置文件**:
```
virsh edit CentOS-7-x86_64
```
### 1.修改cpu
- 修改虚拟机配置文件vcpu项目(默认cpu项目是固定的，必须修改才能热添加cpu等):
将xml文件中vcpu列修改为如下
```
<vcpu placement='auto' current="1">4</vcpu>
```
"目前是1个最大4个"


- 重启kvm虚拟机使其读取xml配置文件
```
virsh shutdown CentOS-7-x86_64
virsh start CentOS-7-x86_64
```
- 更改kvm虚拟机cpu数量(热更改):
```
virsh setvcpus CentOS-7-x86_64 2 --live
```
###2.修改内存
- 由于xml文件中限制了最大内存，所以本例将实现ram的缩小(如要增加可编辑xml文件增加即可)
```
virsh qemu-monitor-command CentOS-7-x86_64 --hmp --cmd balloon 1024
```
*热修改内存大小*
- 查看 
```
virsh qemu-monitor-command CentOS-7-x86_64 --hmp info balloon
```
 ！！！注意：虚拟机硬盘最好不要扩充，可以挂载个新的，扩充可能会出现数丢失
### 3.修改磁盘
- qemu 使用的镜像文件格式有:qcow2和raw,raw不能做快照，qcow2可以
- 对虚拟机来说，IO是qemu来模拟实现的，cpu和内存是kvm来模拟实现的
- qcow2是用多少占用多少的
- 查看镜像信息命令:
```
qemu-img info CentOS-7-x86_64
```
(显示镜像的格式，实际使用了多少)
- raw转为qcow2:
```
qemu-img convert -f raw -O qcow2 CentOS-7-x86_64.raw test.qcow2
```
### 4.网络的修改
- 默认kvm虚拟机是用nat转换来访问网络的,必须依靠iptables,性能有瓶颈，将其改为桥接模式会提升性能
##### 宿主机的操作如下:
- 创建桥接网卡:
```
brctl addbr br0
```
- 查看刚创建的网卡: 
```
brctl show
```
- 将网卡桥接到ens33:
```
brctl addif br0 ens33
```
（此时机器网络会断开 xshell也会断开）
- 将ens33 ip地址删除并配到br0上: 
```
ip addr del dev ens33 192.168.1.4/24
ifconfig br0 192.168.1.4/24 up
```
- 添加网关:
```
route add default gw 192.168.1.1
```
此时ens33是没有ip的，br0有ip，即为正常状态
- 编辑虚拟机的配置文件,使其链接网络
```
virsh edit CentOS-7-x86_64
```
如下修改:
```<interface type='bridge'>
  <mac address='52:54:00:cb:89:6f'/>
   <source bridge='br0'/>
   <model type='virtio'/>
   <address type='pci' d
```
- 重启kvm虚拟机: 
```
virsh shutdown CentOS-7-x86_64
virsh start CentOS-7-x86_64
```
- vnc链接上修改ip地址(宿主机ip为192.168.1.4)
```
BOOTPROTO=static
..........
IPADDR=192.168.1.88
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
```
- 重启kvm虚拟机的网络服务:
```
systemctl restart network
```
此时xshell可以直接连接该kvm虚拟机(192.168.1.88)
### 5.kvm虚拟机的优化
- cpu的优化：kvm在宿主机上是一个进程，会受到cpu的调度，对多核心的机器会把该进程调度到不同的核心，可能会出现cache miss，我们可以把kvm进程绑定到指定的cpu核心
```
taskset -cp 0 6573
```
(qemu进程的pid)  "让kvm进程只运行在cpu0核心上"
- 内存的优化：开启ept技术（bios中开启）
使用大页内存，加快寻址
	​			
	​		    
	​		
	​		