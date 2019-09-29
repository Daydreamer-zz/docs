k8s镜像默认地址仓库无法访问，可手动pull下来打tag

关于版本可以这样查看

```
kubeadm config images list
```



```
docker pull mirrorgooglecontainers/kube-apiserver-amd64:v1.13.4
docker pull mirrorgooglecontainers/kube-controller-manager-amd64:v1.13.4
docker pull mirrorgooglecontainers/kube-scheduler-amd64:v1.13.4
docker pull mirrorgooglecontainers/kube-proxy-amd64:v1.13.4
docker pull mirrorgooglecontainers/pause:3.1
docker pull mirrorgooglecontainers/etcd-amd64:3.2.24
docker pull coredns/coredns:1.2.6
```
```
docker tag docker.io/mirrorgooglecontainers/kube-proxy-amd64:v1.13.4 k8s.gcr.io/kube-proxy:v1.13.4
docker tag docker.io/mirrorgooglecontainers/kube-scheduler-amd64:v1.13.4 k8s.gcr.io/kube-scheduler:v1.13.4
docker tag docker.io/mirrorgooglecontainers/kube-apiserver-amd64:v1.13.4 k8s.gcr.io/kube-apiserver:v1.13.4
docker tag docker.io/mirrorgooglecontainers/kube-controller-manager-amd64:v1.13.4 k8s.gcr.io/kube-controller-manager:v1.13.4
docker tag docker.io/mirrorgooglecontainers/etcd-amd64:3.2.24  k8s.gcr.io/etcd:3.2.24
docker tag docker.io/mirrorgooglecontainers/pause:3.1  k8s.gcr.io/pause:3.1
docker tag docker.io/coredns/coredns:1.2.6  k8s.gcr.io/coredns:1.2.6
```

## 1.配置master节点

### 配置docker-ce和kubernets的yum源

/etc/yum.repo.d/kubernetes.repo

```
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```

docker-ce

```
cd /etc/yum.repos.d && wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

### 安装，确保yum源配置好了

```
yum install -y docker-ce kubeadm kubectl kubelet ipvsadm
```

````
systemctl enable docker && systemctl start docker
````

```
systemctl enable kubelet && systemctl start kubelet
```



###  配置内核参数 

```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

```
sysctl --system
```

### 忽略swap警告(必须加，不加报错)

/etc/sysconfig/kubelet加入

```
"--fail-swap-on=false"
```

### 初始化master

```
kubeadm init --kubernetes-version v1.13.3 --apiserver-advertise-address=192.168.2.4 --image-repository registry.aliyuncs.com/google_containers --service-cidr=10.1.0.0/16 --pod-network-cidr=10.2.0.0/16 --service-dns-domain=cluster.local --ignore-preflight-errors=Swap --service-dns-domain=cluster.local
```

各初始化参数的意义

- --apiserver-advertise-address：指定用 Master 的哪个IP地址与
  Cluster的其他节点通信
- --service-cidr：指定Service网络的范围，即负载均衡VIP使用的IP地址段
- --pod-network-cidr：指定Pod网络的范围，即Pod的IP地址段
- --image-repository：Kubenetes默认Registries地址是
  k8s.gcr.io，在国内并不能访问
  gcr.io，在1.13版本中我们可以增加-image-repository参数，默认值是
  k8s.gcr.io，将其指定为阿里云镜像地址：registry.aliyuncs.com/google_containers
- --kubernetes-version=v1.13.3：指定要安装的版本号
- --ignore-preflight-errors=：忽略运行时的错误，例如上面目前存在[ERROR
  NumCPU]和[ERROR
  Swap]，忽略这两个报错就是增加--ignore-preflight-errors=NumCPU
  和--ignore-preflight-errors=Swap的配置即可。

```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

其他节点(node)想要加入执行以下

```
kubeadm join 192.168.2.4:6443 --token 3hbct1.2wtwzvk37j4hlhan --discovery-token-ca-cert-hash sha256:69b4ad1190f2fb2fc3a0d3ba3a5d4e5b10e1c4f30c4e6c4afcaf1e50f2554996
```

### 查看组件状态

```
kubectl get componentstatus
```
```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-0               Healthy   {"health": "true"} 
```

### 查看节点信息

```
kubectl get nodes
```

### 部署flannel网络(适合1.7版本以上的kubernetes)

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

部署完后查看

```
[root@node1 ~]# kubectl get node
NAME    STATUS   ROLES    AGE   VERSION
node1   Ready    master   25m   v1.13.3
```

```
[root@node1 ~]# kubectl get pods -n kube-system
NAME                            READY   STATUS    RESTARTS   AGE
coredns-86c58d9df4-fj7td        1/1     Running   0          26m
coredns-86c58d9df4-rj8n6        1/1     Running   0          26m
etcd-node1                      1/1     Running   0          25m
kube-apiserver-node1            1/1     Running   0          26m
kube-controller-manager-node1   1/1     Running   0          26m
kube-flannel-ds-amd64-hmtbd     1/1     Running   0          5m38s
kube-proxy-tztmz                1/1     Running   0          26m
kube-scheduler-node1            1/1     Running   0          25m
```

## 2.配置node节点

### 和mster 节点一样拉取好镜像

````
yum install -y docker-ce kubeadm kubectl kubelet
````

```
systemctl enable docker kubelet
```

```
kubeadm join 192.168.2.4:6443 --token 3hbct1.2wtwzvk37j4hlhan --discovery-token-ca-cert-hash sha256:69b4ad1190f2fb2fc3a0d3ba3a5d4e5b10e1c4f30c4e6c4afcaf1e50f2554996 --ignore-preflight-errors=Swap
```

### 在master节点上查看 

````
[root@node1 ~]# kubectl get nodes
NAME    STATUS   ROLES    AGE     VERSION
node1   Ready    master   41m     v1.13.3
node2   Ready    <none>   8m54s   v1.13.3
````

