# Ubuntu 20.04 安装 Kubernetes - *by kubeadm*

## 环境相关

### 虚拟机平台

> VMware® Workstation 16 Pro
>
> 16.2.4 build-20089737

### 系统版本

```bash
steve@umaster:~$ cat /etc/os-release
NAME="Ubuntu"
VERSION="20.04.5 LTS (Focal Fossa)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 20.04.5 LTS"
VERSION_ID="20.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=focal
UBUNTU_CODENAME=focal
```

### 主机规划

| 主机名 | IP地址        | 系统版本           |
| ------ | ------------- | ------------------ |
| master | 192.168.1.120 | Ubuntu 20.04.5 LTS |
| node1  | 192.168.1.121 | Ubuntu 20.04.5 LTS |
| node2  | 192.168.1.122 | Ubuntu 20.04.5 LTS |



## 准备阶段

### 更新软件包

```bash
sudo apt update
sudo apt upgrade -y
```

### 固定IP

#### 查看网络配置

```bash
steve@umaster:~$ ip address
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens32: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:4d:f7:0c brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.106/24 brd 192.168.1.255 scope global dynamic ens32
       valid_lft 5313sec preferred_lft 5313sec
    inet6 fe80::20c:29ff:fe4d:f70c/64 scope link
       valid_lft forever preferred_lft forever
steve@umaster:~$ ls /etc/netplan/
00-installer-config.yaml
steve@umaster:~$ cat /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens32:
      dhcp4: true
  version: 2
```

#### 设置固定IP

```bash
# 配置固定IP
steve@umaster:~$ sudo vim /etc/netplan/00-installer-config.yaml

# 配置好之后应该类似这样
steve@umaster:~$ cat /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens32:      # 网卡名称
      dhcp4: false      # 关闭DHCP
      addresses: [192.168.1.120/24]     # 配置静态IP和掩码
      gateway4: 192.168.1.1     # 网关地址
      optional: true    # An optional device is not required for booting.
      nameservers:
        addresses: [192.168.1.1,114.114.114.114]        # DNS服务器
  version: 2
```

```bash
# 使配置生效
steve@umaster:~$ sudo netplan apply

# 查看效果: 192.168.1.106 -> 192.168.1.120
steve@umaster:~$ ip address
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens32: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:4d:f7:0c brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.120/24 brd 192.168.1.255 scope global ens32
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe4d:f70c/64 scope link
       valid_lft forever preferred_lft forever
```

> 更多 `netplan` 介绍请参考: https://netplan.io/reference

### 修改主机名

```bash
# 修改前
steve@umaster:~$ hostname
umaster
steve@umaster:~$ cat /etc/hostname
umaster

# 修改
steve@umaster:~$ sudo hostnamectl set-hostname master

# 修改后
steve@umaster:~$ hostname
master
steve@umaster:~$ cat /etc/hostname
master
```

```bash
# 修改hosts
steve@umaster:~$ sudo vim /etc/hosts

# 修改后类似这样, 主要是把: 主机固定IP -> 主机名
steve@umaster:~$ cat /etc/hosts
127.0.0.1 localhost
192.168.1.120 master

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

# 看下效果
steve@umaster:~$ ping -c 4 master
PING master (192.168.1.120) 56(84) bytes of data.
64 bytes from master (192.168.1.120): icmp_seq=1 ttl=64 time=0.041 ms
64 bytes from master (192.168.1.120): icmp_seq=2 ttl=64 time=0.059 ms
64 bytes from master (192.168.1.120): icmp_seq=3 ttl=64 time=0.025 ms
64 bytes from master (192.168.1.120): icmp_seq=4 ttl=64 time=0.022 ms

--- master ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3048ms
rtt min/avg/max/mdev = 0.022/0.036/0.059/0.014 ms
```

> 如果细心一点可能会发现, 虽然修改主机名成功了, 但是SSH连接中并没有变化: steve@umaster
>
> 此时只需要退出重新连接一下就好了, 如下:
>
> steve@master:~$ hostname
> master

### 关闭交换分区

```bash
# 关闭前
steve@master:~$ free -m
              total        used        free      shared  buff/cache   available
Mem:           7927         341        4795           1        2789        7287
Swap:          4095           0        4095

# 关闭
steve@master:~$ sudo sed -ri 's/.*swap.*/#&/' /etc/fstab	# 永久
steve@master:~$ sudo swapoff -a	# 临时

# 关闭后
steve@master:~$ free -m
              total        used        free      shared  buff/cache   available
Mem:           7927         344        4792           1        2789        7284
Swap:             0           0           0
```



----

> 温馨提示:
>
> 1. 此时最好应该重启一下系统, 然后再次检查一下, 上述操作是否均已生效;
> 2. 上面只是在 `master` 节点做了配置, 应该根据自己的主机规划, 对 **所有节点** 都配置一下;
> 3. 如果和我一样用的是虚拟机, 此时最好对已配置好的节点, 建立一个快照.

----



## Docker 安装

> 略有改动, 主要还是参考官方文档: https://docs.docker.com/engine/install/ubuntu/

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg lsb-release
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 添加软件仓库 - 清华大学软件源
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER && newgrp docker

# 安装完毕, 查看docker版本信息
docker version
```

```bash
# 修改配置 - containerd
sudo systemctl stop containerd.service
sudo cp /etc/containerd/config.toml /etc/containerd/config.toml.bak
sudo containerd config default > $HOME/config.toml
sudo cp $HOME/config.toml /etc/containerd/config.toml
sudo sed -i "s#registry.k8s.io/pause#registry.cn-hangzhou.aliyuncs.com/google_containers/pause#g" /etc/containerd/config.toml
sudo sed -i "s#SystemdCgroup = false#SystemdCgroup = true#g" /etc/containerd/config.toml

# 修改配置 - docker
sudo systemctl stop docker.socket
sudo systemctl stop docker.service
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://hnkfbj7x.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

# 重启服务, 也可以重启系统
sudo systemctl daemon-reload
sudo systemctl restart docker

# 查看运行状态 - 应该都是 Active: active (running)
systemctl status docker.service
systemctl status containerd.service

# 此时查看docker信息 - 应该能看到: 1.Cgroup驱动改为systemd; 2.镜像源改为阿里云.
docker info
# Cgroup Driver: systemd
# systemd Registry Mirrors: https://hnkfbj7x.mirror.aliyuncs.com/
```



----

> 温馨提示:
>
> 1. 同样地, 应该根据自己的主机规划, **所有节点** 都需要安装 docker;
> 2. 同样地, 此时最好建立一个快照.

----



## 工具安装

> kubeadm 官方安装文档: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

> 阿里云 K8S 镜像配置文档: https://developer.aliyun.com/mirror/kubernetes

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
# 安装工具 - kubelet kubeadm kubectl
sudo apt-get install -y kubelet kubeadm kubectl
# 版本锁定
sudo apt-mark hold kubelet kubeadm kubectl
```

```bash
# 查询某个版本 - 例如: 1.26, 显示的版本号会重复三次, 因为有三个工具
curl -s https://mirrors.aliyun.com/kubernetes/apt/dists/kubernetes-xenial/main/binary-amd64/Packages | grep "Version: 1.26"
Version: 1.26.0-00
Version: 1.26.0-00
Version: 1.26.1-00
Version: 1.26.0-00
Version: 1.26.1-00
Version: 1.26.0-00
Version: 1.26.1-00
Version: 1.26.0-1

# 安装特定版本
sudo apt-get install -y kubelet=1.26.0-00 kubeadm=1.26.0-00 kubectl=1.26.0-00

# 版本锁定
sudo apt-mark hold kubelet kubeadm kubectl
```

```bash
# 安装后查看版本
kubeadm version
kubectl version
kubelet --version

# 此时查看kubelet状态 - 应该是: activating (auto-restart)
# The kubelet is now restarting every few seconds, as it waits in a crashloop for kubeadm to tell it what to do.
systemctl status kubelet
```



----

> 温馨提示:
>
> 1. 同样地, **所有节点** 都需要安装 kubelet kubeadm kubectl;
> 2. 同样地, 此时最好建立一个快照.

----



## 集群搭建

### Master 节点

```bash
# 初始化
sudo kubeadm init --image-repository=registry.aliyuncs.com/google_containers
```

> 安装成功, 应该会有如下信息:
>
> Your Kubernetes control-plane has initialized successfully!
>
> To start using your cluster, you need to run the following as a regular user:
>
> mkdir -p $HOME/.kube
> sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
> sudo chown $(id -u):$(id -g) $HOME/.kube/config
>
> Alternatively, if you are the root user, you can run:
>
> export KUBECONFIG=/etc/kubernetes/admin.conf
>
> You should now deploy a pod network to the cluster.
> Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
> https://kubernetes.io/docs/concepts/cluster-administration/addons/
>
> Then you can join any number of worker nodes by running the following on each as root:
>
> kubeadm join 192.168.1.120:6443 --token zxg02f.8pdotfmmcbrke2ix \
> 	--discovery-token-ca-cert-hash sha256:af7ce57381d7c54417cf19865897eb9ad670309ee392d0647ab95ce918e69696

```bash
# 按照上述提示信息, 继续操作
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 如果初始化失败, 可用以下命令进行重置后, 再重新初始化
kubeadm reset
```

### Worker 节点

```bash
# 加入集群 - master节点初始化生成
sudo kubeadm join 192.168.1.120:6443 --token zxg02f.8pdotfmmcbrke2ix \
	--discovery-token-ca-cert-hash sha256:af7ce57381d7c54417cf19865897eb9ad670309ee392d0647ab95ce918e69696
```
> 加入集群成功, 有如下提示信息:
>
> This node has joined the cluster:
>
> * Certificate signing request was sent to apiserver and a response was received.
> * The Kubelet was informed of the new secure connection details.
>
> Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```bash
# 如果上述信息找不到或过期了, 可以在 master节点 执行以下命令重新生成:
kubeadm token create --print-join-command
```

```bash
# master节点 - 查看节点状态 - 可以查看到节点状态(目前都是 NotReady, 因为网络还未配置), 和各自内网IP
steve@master:~$ kubectl get nodes -o wide
NAME     STATUS     ROLES           AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master   NotReady   control-plane   46m     v1.26.1   192.168.1.120   <none>        Ubuntu 20.04.5 LTS   5.4.0-139-generic   containerd://1.6.18
node1    NotReady   <none>          3m35s   v1.26.1   192.168.1.121   <none>        Ubuntu 20.04.5 LTS   5.4.0-139-generic   containerd://1.6.18
node2    NotReady   <none>          3m30s   v1.26.1   192.168.1.122   <none>        Ubuntu 20.04.5 LTS   5.4.0-139-generic   containerd://1.6.18

# master节点 - 查看pods, 可以查看到 coredns-* 的状态是 Pending，其他均为 Running，原因是网络还未配置
steve@master:~$ kubectl get pods --all-namespaces
NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE
kube-system   coredns-5bbd96d687-k7lhr         0/1     Pending   0          47m
kube-system   coredns-5bbd96d687-r4w7s         0/1     Pending   0          47m
kube-system   etcd-master                      1/1     Running   0          47m
kube-system   kube-apiserver-master            1/1     Running   0          47m
kube-system   kube-controller-manager-master   1/1     Running   0          47m
kube-system   kube-proxy-2dmmp                 1/1     Running   0          4m52s
kube-system   kube-proxy-2z446                 1/1     Running   0          4m57s
kube-system   kube-proxy-mfwdh                 1/1     Running   0          47m
kube-system   kube-scheduler-master            1/1     Running   0          47m
```

### 配置网络

> Calico 官方文档(要求): https://docs.tigera.io/calico/3.25/getting-started/kubernetes/requirements
>
> > Calico v3.25 支持的 Kubernetes 版本:
> >
> > - v1.23
> > - v1.24
> > - v1.25
> > - v1.26
>
> Calico 配置清单: https://github.com/projectcalico/calico/tree/master/manifests

```bash
# master节点
# 下载配置文件
wget --no-check-certificate https://projectcalico.docs.tigera.io/archive/v3.25/manifests/calico.yaml
# 修改配置文件
vim calico.yaml

# Cluster type to identify the deployment type
- name: CLUSTER_TYPE
  value: "k8s,bgp"
# 配置网卡 - 以下为新增内容
- name: IP_AUTODETECTION_METHOD
  value: "interface=ens32"

# 配置网络
kubectl apply -f calico.yaml
```

```bash
# master节点
# 此时查看pods, 可以看到三个 calico-node-* 状态为: Init:0/3, 正在初始化, 需要等待几分钟.
kubectl get pods --all-namespaces

# 网络配置完成后, 再次查看pods, 应该可以看到如下信息: 状态全部为 Running
steve@master:~$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-57b57c56f-j7gmv   1/1     Running   0          4m8s
kube-system   calico-node-2pbjm                         1/1     Running   0          4m8s
kube-system   calico-node-7tq8p                         1/1     Running   0          4m8s
kube-system   calico-node-frhq7                         1/1     Running   0          4m8s
kube-system   coredns-5bbd96d687-k7lhr                  1/1     Running   0          41h
kube-system   coredns-5bbd96d687-r4w7s                  1/1     Running   0          41h
kube-system   etcd-master                               1/1     Running   0          41h
kube-system   kube-apiserver-master                     1/1     Running   0          41h
kube-system   kube-controller-manager-master            1/1     Running   0          41h
kube-system   kube-proxy-2dmmp                          1/1     Running   0          40h
kube-system   kube-proxy-2z446                          1/1     Running   0          40h
kube-system   kube-proxy-mfwdh                          1/1     Running   0          41h
kube-system   kube-scheduler-master                     1/1     Running   0          41h

# 此时再查看节点信息, 状态应该全部为就绪: Ready
steve@master:~$ kubectl get nodes
NAME     STATUS   ROLES           AGE   VERSION
master   Ready    control-plane   41h   v1.26.1
node1    Ready    <none>          40h   v1.26.1
node2    Ready    <none>          40h   v1.26.1
```



----

> 温馨提示:
>
> 1. 此时正常应该是环境搭建好了, 最好也建立一个快照.

----



### 集群测试

```bash
# 创建服务: nginx
vim nginx.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.23.3
        ports:
        - containerPort: 80
```

```bash
# 部署 nginx
kubectl apply -f nginx.yaml
```

```bash
# 查看是否部署成功
steve@master:~$ kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE    NOMINATED NODE   READINESS GATES
nginx-deployment-866c5d4565-5mnjd   1/1     Running   0          32s   172.16.104.4     node2   <none>           <none>
nginx-deployment-866c5d4565-dmtqx   1/1     Running   0          32s   172.16.166.129   node1   <none>           <none>

# 访问任一IP:
steve@master:~$ curl 172.16.104.4
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

```bash
# 设置服务
kubectl expose deployment nginx-deployment --type=NodePort --name=nginx-service
# 查看服务
steve@master:~$ kubectl get svc -o wide
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE     SELECTOR
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP        41h     <none>
nginx-service   NodePort    10.110.82.209   <none>        80:32466/TCP   2m21s   app=nginx
# 内网通过CLUSTER-IP访问服务
curl 10.110.82.209
# 外网通过节点IP访问服务
curl 192.168.1.120:32466
curl 192.168.1.121:32466
curl 192.168.1.122:32466
```

>设置服务的好处:
>
>1. 对外暴露应用;
>2. 外部访问任一节点都可以访问到服务;
>3. 多实例时, 自动地负载均衡;
>4. 重启后, 即使 `pod` 的 IP 会变, 但是`service` 的 CLUSTER-IP 和端口固定不变.
