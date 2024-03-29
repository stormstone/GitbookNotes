# Kubernetes学习

## Kubernetes

Kubernetes是容器集群管理系统，是一个开源的平台，可以实现容器集群的自动化部署、自动扩缩容、维护等功能。

* 快速部署应用
* 快速扩展应用
* 无缝对接新的应用功能
* 节省资源，优化硬件资源的使用

Kubernetes架构：

![](../.gitbook/assets/k8s_architecture.png)

Kubernetes主要由以下几个核心组件组成：

* etcd保存了整个集群的状态；
* apiserver提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
* controller manager负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
* scheduler负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；
* kubelet负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管理；
* Container runtime负责镜像管理以及Pod和容器的真正运行（CRI）；
* kube-proxy负责为Service提供cluster内部的服务发现和负载均衡；

除了核心组件，还有一些推荐的Add-ons：

* kube-dns负责为整个集群提供DNS服务
* Ingress Controller为服务提供外网入口
* Heapster提供资源监控
* Dashboard提供GUI
* Federation提供跨可用区的集群
* Fluentd-elasticsearch提供集群日志采集、存储与查询

![](../.gitbook/assets/k8s_architecture02.png)

Master:

![](../.gitbook/assets/k8s_architecture_master.png)

Node:

![](../.gitbook/assets/k8s_node.png)

## 虚拟机centos7环境搭建kubernetes集群

### 一. 环境准备

总共三台虚拟机，一台master，两台node。

基本配置三台机器都是一样的，使用VMware创建虚拟机，可以配置好一台之后通过复制虚拟机创建其他两台，减少配置步骤。

操作系统：

```bash
[root@master ~]# uname -a
Linux master 3.10.0-957.el7.x86_64 #1 SMP Thu Nov 8 23:39:32 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
​
[root@master ~]# cat /etc/redhat-release 
CentOS Linux release 7.6.1810 (Core) 
```

关闭防火墙：

```bash
[root@master ~]# systemctl disable firewalld.service
[root@master ~]# systemctl stop firewalld.service
```

关闭selinux：

```bash
[root@master ~]# vim /etc/selinux/config
​
设置：SELINUX=disabled
```

开启ntpd：

```bash
[root@master ~]# systemctl enable ntpd
[root@master ~]# systemctl start ntpd
```

配置hosts：

```bash
[root@master ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.236.128 master
192.168.236.129 node1
192.168.236.130 node2
```

### 二.etcd安装和部署

安装：

```bash
[root@master ~]# yum install etcd -y
```

配置：

CLIENT\_URLS后面加入每台机器对应的ip，NAME为每台机器对应名称。

```bash
[root@master ~]# cat /etc/etcd/etcd.conf | grep -v "^#"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_CLIENT_URLS="http://192.168.236.128:2379,http://localhost:2379"
ETCD_NAME="master"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.236.128:2379,http://localhost:2379"
```

启动etcd集群（所有节点）：

```bash
[root@master ~]# systemctl enable etcd
[root@master ~]# systemctl start etcd
[root@master ~]#  etcdctl set testdir/testkey0 0
0
[root@master ~]# etcdctl -C http://master:2379 cluster-health
member 8e9e05c52164694d is healthy: got healthy result from http://192.168.236.128:2379
cluster is healthy
```

### 三.kubernetes安装和部署

安装：

```bash
[root@master ~]# yum install kubernetes -y
```

如果报错docker安装冲突，先卸载docker。

配置：

* master

```bash
[root@master ~]# cat /etc/kubernetes/apiserver | grep -v "^#"
KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"
KUBE_API_PORT="--port=8080"
KUBE_ETCD_SERVERS="--etcd-servers=http://192.168.236.128:2379"
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,ResourceQuota"
KUBE_API_ARGS="--service_account_key_file=/tmp/serviceaccount.key"
```

```bash
[root@master ~]# cat /etc/kubernetes/config | grep -v "^#"
KUBE_LOGTOSTDERR="--logtostderr=true"
KUBE_LOG_LEVEL="--v=0"
KUBE_ALLOW_PRIV="--allow-privileged=false"
KUBE_MASTER="--master=http://master:8080"
```

* node

```bash
[root@node1 ~]# cat /etc/kubernetes/kubelet | grep -v "^#"
KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_HOSTNAME="--hostname-override=192.168.236.129"
KUBELET_API_SERVER="--api-servers=http://192.168.236.128:8080"
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"
KUBELET_ARGS=""
```

```bash
[root@node1 ~]# cat /etc/kubernetes/config | grep -v "^#"
KUBE_LOGTOSTDERR="--logtostderr=true"
KUBE_LOG_LEVEL="--v=0"
KUBE_ALLOW_PRIV="--allow-privileged=false"
KUBE_MASTER="--master=http://master:8080"
```

启动docker（所有节点）：

```bash
[root@master ~]# systemctl enable docker
[root@master ~]# systemctl start docker
[root@master ~]# docker --version
Docker version 1.13.1, build b2f74b2/1.13.1
```

启动kubernets服务：

* master

```bash
[root@master ~]# systemctl enable kube-apiserver kube-controller-manager kube-scheduler
[root@master ~]# systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

* node

```bash
[root@node1 ~]# systemctl enable kubelet kube-proxy
[root@node1 ~]# systemctl start kubelet kube-proxy
```

查看服务端口：

* master

```bash
[root@master ~]# netstat -tnlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:10248         0.0.0.0:*               LISTEN      12156/kubelet       
tcp        0      0 127.0.0.1:10249         0.0.0.0:*               LISTEN      9336/kube-proxy     
tcp        0      0 127.0.0.1:10250         0.0.0.0:*               LISTEN      12156/kubelet       
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      9340/etcd           
tcp        0      0 192.168.236.128:2379    0.0.0.0:*               LISTEN      9340/etcd           
tcp        0      0 127.0.0.1:2380          0.0.0.0:*               LISTEN      9340/etcd           
tcp        0      0 127.0.0.1:10255         0.0.0.0:*               LISTEN      12156/kubelet       
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      9335/sshd           
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      9555/master         
tcp6       0      0 :::6443                 :::*                    LISTEN      9936/kube-apiserver 
tcp6       0      0 :::10251                :::*                    LISTEN      8559/kube-scheduler 
tcp6       0      0 :::10252                :::*                    LISTEN      8985/kube-controlle 
tcp6       0      0 :::8080                 :::*                    LISTEN      9936/kube-apiserver 
tcp6       0      0 :::22                   :::*                    LISTEN      9335/sshd           
tcp6       0      0 ::1:25                  :::*                    LISTEN      9555/master         
tcp6       0      0 :::4194                 :::*                    LISTEN      12156/kubelet
```

* node

```bash
[root@node1 ~]# netstat -ntlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:10248         0.0.0.0:*               LISTEN      120274/kubelet      
tcp        0      0 127.0.0.1:10249         0.0.0.0:*               LISTEN      119983/kube-proxy   
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      19317/etcd          
tcp        0      0 192.168.236.129:2379    0.0.0.0:*               LISTEN      19317/etcd          
tcp        0      0 127.0.0.1:2380          0.0.0.0:*               LISTEN      19317/etcd          
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      9276/sshd           
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      9470/master         
tcp6       0      0 :::10250                :::*                    LISTEN      120274/kubelet      
tcp6       0      0 :::10255                :::*                    LISTEN      120274/kubelet      
tcp6       0      0 :::22                   :::*                    LISTEN      9276/sshd           
tcp6       0      0 ::1:25                  :::*                    LISTEN      9470/master         
tcp6       0      0 :::4194                 :::*                    LISTEN      120274/kubelet
```

查看状态：

master上查看集群节点状态.

```bash
[root@master ~]# kubectl get node
NAME              STATUS    AGE
127.0.0.1         Ready     1d
192.168.236.129   Ready     1d
192.168.236.130   Ready     1d
```

### 四. flannel安装配置

安装：

```bash
[root@node1 ~]# yum install flannel
```

配置（所有节点）：

```bash
[root@master ~]# cat /etc/sysconfig/flanneld | grep -v "^#"
FLANNEL_ETCD_ENDPOINTS="http://192.168.236.128:2379"
FLANNEL_ETCD_PREFIX="/k8s/network"
```

添加网络（master节点）：

```bash
[root@master ~]# etcdctl mk //k8s/network/config ‘{“Network”:“172.8.0.0/16”}’
```

启动服务（所有节点）：

```bash
[root@master ~]# systemctl enable flanneld
[root@master ~]# systemctl start flanneld
```

重启k8s服务：

* master

```bash
[root@master ~]# for SERVICES in docker kube-apiserver kube-controller-manager kube-scheduler; do systemctl restart $SERVICES ; done
```

* node

```bash
[root@master ~]# systemctl restart kube-proxy kubelet docker
```

查看flannel网络:

```bash
[root@master ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:74:1d:08 brd ff:ff:ff:ff:ff:ff
    inet 192.168.236.128/24 brd 192.168.236.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::5e83:4f0:885a:d7a5/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: flannel0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1472 qdisc pfifo_fast state UNKNOWN group default qlen 500
    link/none 
    inet 172.8.63.0/16 scope global flannel0
       valid_lft forever preferred_lft forever
    inet6 fe80::bf60:4070:fda6:71b4/64 scope link flags 800 
       valid_lft forever preferred_lft forever
4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1472 qdisc noqueue state UP group default 
    link/ether 02:42:ad:d1:a8:58 brd ff:ff:ff:ff:ff:ff
    inet 172.8.63.1/24 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:adff:fed1:a858/64 scope link 
       valid_lft forever preferred_lft forever
6: vethdb5cea3@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1472 qdisc noqueue master docker0 state UP group default 
    link/ether 1e:36:38:bb:35:ba brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::1c36:38ff:febb:35ba/64 scope link 
       valid_lft forever preferred_lft forever
```

### 五. 测试

测试一个MySQL示例：

```bash
[root@master ~]# vim mysql-rc.yaml
```

```bash
apiVersion: v1
kind: ReplicationController
metadata:
  name: mysql
spec:
  replicas: 3
  selector:
    app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "123456"
```

发布到k8s集群：

```bash
[root@master ~]# kubectl create -f mysql-rc.yaml
```

查看：

```bash
[root@master ~]# kubectl get rc
NAME      DESIRED   CURRENT   READY     AGE
mysql     3         3         0         13s
​
[root@master ~]# kubectl get pod
NAME                      READY     STATUS              RESTARTS   AGE
mysql-dtd9h               0/1       ContainerCreating   0          3s
mysql-jfd5p               0/1       ContainerCreating   0          3s
mysql-stsb3               0/1       ErrImagePull        0          3s
```

### 六.常见错误

**1. current为０的问题**

如果出现如下情况：

```bash
[root@master ~]# kubectl get rc
NAME      DESIRED   CURRENT   READY     AGE
mysql     3         0         0         1m
```

解决方案：

```bash
# 1.Generate a signing key:
[root@master ~]# openssl genrsa -out /tmp/serviceaccount.key 2048
​
# 2.修改apiserver配置
[root@master ~]# vim /etc/kubernetes/apiserver
KUBE_API_ARGS="--service_account_key_file=/tmp/serviceaccount.key"
​
# 3.修改controller-manager配置
[root@master ~]# vim /etc/kubernetes/controller-manager
KUBE_CONTROLLER_MANAGER_ARGS="--service_account_private_key_file=/tmp/serviceaccount.key"
​
# 4.重启服务
[root@master ~]# systemctl restart kube-apiserver kube-controller-manager
​
# 5.删除原来的创建的RC
kubectl delete -f mysql-rc.yaml
​
# 6.重新创建
kubectl create -f mysql-rc.yaml
​
# 7.验证一下
[root@master ~]# kubectl get rc
NAME      DESIRED   CURRENT   READY     AGE
mysql     3         3         0         5m
```

**2. pod一直处于ContainerCreating**

通过`kubectl describe pod [name]`命令查看，最主要的问题是：details: \(open /etc/docker/certs.d/registry.access.redhat.com/redhat-ca.crt: no such file or directory\)

解决方案：

查看/etc/docker/certs.d/registry.access.redhat.com/redhat-ca.crt 是一个软链接，但是链接过去后并没有真实的/etc/rhsm，所以需要使用yum安装：

```bash
yum install *rhsm*
```

安装完成后，执行一下`docker pull registry.access.redhat.com/rhel7/pod-infrastructure:latest` 更新镜像。

如果依然报错，可参考下面的方案：

```bash
wget <http://mirror.centos.org/centos/7/os/x86_64/Packages/python-rhsm-certificates-1.19.10-1.el7_4.x86_64.rpm>
​
rpm2cpio python-rhsm-certificates-1.19.10-1.el7_4.x86_64.rpm | cpio -iv --to-stdout ./etc/rhsm/ca/redhat-uep.pem | tee /etc/rhsm/ca/redhat-uep.pem
```

这两个命令会生成/etc/rhsm/ca/redhat-uep.pem文件.

顺得的话会得到下面的结果:

```bash
docker pull registry.access.redhat.com/rhel7/pod-infrastructure:latest
​
Trying to pull repository registry.access.redhat.com/rhel7/pod-infrastructure ...
​
latest: Pulling from registry.access.redhat.com/rhel7/pod-infrastructure
​
26e5ed6899db: Pull complete
​
66dbe984a319: Pull complete
​
9138e7863e08: Pull complete
​
Digest: sha256:92d43c37297da3ab187fc2b9e9ebfb243c1110d446c783ae1b989088495db931
​
Status: Downloaded newer image for registry.access.redhat.com/rhel7/pod-infrastructure:latest
```

```bash
# 删除原来的创建的RC
kubectl delete -f mysql-rc.yaml
​
# 重新创建
kubectl create -f mysql-rc.yaml
​
# 验证一下
[root@master ~]# kubectl get rc
NAME      DESIRED   CURRENT   READY     AGE
mysql     3         3         0         5m
```

**3.虚拟机静态ip，apiserver启动失败**

将虚拟机的ip方式 由dhcp改成static，apiserver 启动失败。

查看路由：

```bash
[root@master ~]# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
link-local      0.0.0.0         255.255.0.0     U     1002   0        0 ens33
172.8.63.0      0.0.0.0         255.255.255.0   U     0      0        0 docker0
192.168.236.0   0.0.0.0         255.255.255.0   U     100    0        0 ens33
```

缺少默认路由，配置主机默认路由：

```bash
[root@master ~]# ip route add default           via  192.168.236.2  dev    ens33   proto  dhcp  metric  100
```

重启kube-apiserver即可。

### References

1. [Kubernetes中文社区 \| 中文文档](http://docs.kubernetes.org.cn/)
2. [Kubernetes指南](https://kubernetes.feisky.xyz/)
3. [虚拟机环境下centos7搭建kubernetes集群](https://blog.csdn.net/isshpan/article/details/89445693)
4. [k8s坑--current为０的问题](https://blog.csdn.net/a506681571/article/details/86087456)
5. [Kubernetes创建pod一直处于ContainerCreating排查和解决](https://blog.51cto.com/12482328/2120035)
6. [kubernetes APIServer 启动失败](http://dockone.io/question/736)

