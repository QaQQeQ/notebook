k8s：容器编排平台


1）关闭防火墙，关闭selinux    -----   node1、node2、node3
2）各节点主机名解析    -----   node1、node2、node3
3）关闭交换分区    -----   node1、node2、node3
4）配置时间同步服务    -----   node1(服务端), node2、node3(客户端)
5）配置网络模块与参数    -----   node1、node2、node3
6）准备yum源（base源、epel源、docker源）    -----   node1、node2、node3
7）安装docker引擎和container  -----   node1、node2、node3
8）准备k8s组件的yum源    -----   node1、node2、node3
9）安装核心软件包    -----   node1、node2、node3
10）修改容器运行时配置文件    -----   node1、node2、node3
11）初始化集群    -----   node1 (Master节点)
12）管理配置文件（按输出提示执行）    -----   node1 (Master节点)
13）安装网络插件    -----   node1 (Master节点)
14）安装配置container服务（同主节点）    -----   node2、node3
15）执行加入命令（主节点初始化生成）    -----   node2、node3
16）查看集群状态，验证部署成功    -----   node1 (Master节点)



#k8s安装部署
1、环境准备：
node1 192.168.100.10  master
node2 192.168.100.20  work node
node3 192.168.100.30  word node
1）关闭防火墙，关闭selinux
systemctl stop firewalld
systemctl disable firewalld
sed -i 's/enforcing/disabled/' /etc/selinux/config  #永久
setenforce 0  #临时

2）各节点主机名解析
修改对应的主机名

vim /etc/hosts
192.168.100.10 node1
192.168.100.20 node2
192.168.100.30 node3

3）关闭交换分区
sed -ri 's/.*swap.*/#&/' /etc/fstab  #永久关闭
swapoff -a                     #临时关闭

4）配置时间同步服务
master是服务端    vim /etc/chrony.conf
work node是客户端

5）配置网络模块与参数
查看模块是否加载：
    #  lsmod | grep overlay
lsmod | grep br_netfilter
如果没有，手动加载： modprobe -a 模块名称

修改网桥网络参数：解决于iptables被绕过而导致流量路由不正确的问题
cat > /etc/sysctl.d/k8s.conf << EOF
net.ipv4.ip_forward = 1
vm.swappiness = 0
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system  #生效

2、安装核心组件
1）准备yum源（base源、epel源、docker源）
yum -y install wget
wget -O /etc/yum.repos.d/ali.repo http://mirrors.aliyun.com/repo/Centos-7.repo
wget -O /etc/yum.repos.d/epel-7.repo  https://mirrors.aliyun.com/repo/epel-7.repo
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo

2）安装docker引擎和container 
1.24版本后容器引擎可以直接安装container（效率和性能更换）

也可以单独只安装container： yum -y install containerd


 3）准备k8s组件的yum源
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

 4）安装核心软件包
yum install -y kubelet-1.26.3 kubeadm-1.26.3 kubectl-1.26.3 --disableexcludes=kubernetes
systemctl enable kubelet  
说明：先不直接启动服务，启动也会停掉，因为/etc/kubernetes/mainfests/没有配置文件
以上所有节点执行
kubelet 的版本不可以超过 API server 的版本
kubeadm：k8集群的一键部署工具，通过把k8的各类核心组件和插件以pod的方式部署来简化安装过程
kubelet：运行在每个节点上的node agent，k8集群通过kubelet真正的去操作每个节点上的容器，由于需要直接操作宿主机的各类资源，所以没有放在pod里面，还是通过服务的形式装在系统里面。

5）修改容器运行时配置文件
初始化前需要修改container服务的配置文件：
cd /etc/containerd
mv config.toml config.toml.bak
containerd config default > config.toml
vim config.toml 修改如下内容：
........
sandbox_image = "registry.k8s.io/pause:3.6"
sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.6"

SystemdCgroup = true
修改完后重启容器服务：
systemctl restart containerd

6）初始化集群   ###注意：只能在master节点运行
禁用swap
swapoff -a
free -h

kubeadm init \
--apiserver-advertise-address=192.168.10.10 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.26.3 \
--service-cidr=10.96.0.0/12 \
--pod-network-cidr=10.244.0.0/16 \
--ignore-preflight-errors=all
说明：
--apiserver-advertise-address=192.168.10.10 \           #换成自己的master地址  
初始化会去下载各组件需要的镜像，如果下载失败。可以先单独去下载好镜像。
kubeadm config images list
该命令可以查看kubeadm初始化需要的镜像。

7）管理配置文件（按输出提示执行）
普通用户：
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
如果是管理员只需执行：
export KUBECONFIG=/etc/kubernetes/admin.conf
执行完就可以启动集群(加到/etc/profile里实现开机自动启动)
[root@master ~]# systemctl status kubelet.service  #确认服务已经启动

8）安装网络插件
方式一：calico
下载：
wget http://manongbiji.oss-cn-beijing.aliyuncs.com/ittailkshow/k8s/download/calico.yaml
应用：
kubectl apply -f calico.yaml

方式二：flannel
下载kube-flannel.yml文件
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
应用kube-flannel.yml文件得到运行时容器
kubectl apply -f kube-flannel.yml 

3、节点加入集群
1）安装配置container服务（同主节点）
镜像加速配置：  （安装时可以先不做）
编辑/etc/containerd/config.toml文件
[plugins."io.containerd.grpc.v1.cri".registry]
   config_path = "/etc/containerd/certs.d"
.....
 mkdir -p /etc/containerd/certs.d
 mkdir -p /etc/containerd/certs.d/docker.io
mkdir -p /etc/containerd/certs.d/docker.io
cat > /etc/containerd/certs.d/docker.io/hosts.toml << EOF
server = "https://docker.io"
[host."https://dockerpull.com"]
  capabilities = ["pull", "resolve"]
[host."https://register.liberx.info"]
  capabilities = ["pull", "resolve"]
[host."https://docker.m.daocloud.io"]
  capabilities = ["pull", "resolve"]
EOF
 systemctl restart containerd

2）执行加入命令（主节点初始化生成）
kubeadm join 192.168.10.10:6443 --token 4xew3f.jt4mfuxbcuklz094 \
	--discovery-token-ca-cert-hash sha256:7908dfe8a32fbbf27907d521043b3646f5a788a1c297e71a5ed8fa557acd07a0 
说明：
如果无法查看初始化信息，重新生成加入集群命令
kubeadm token create --print-join-command

4、查看集群状态，验证部署成功
kubectl get nodes #查看所有节点信息
kubectl get pod -n kube-system  #查看核心组件以pod（容器）的方式运行






