[TOC]

## 使用kubeadm部署kubernetes

### 1. 准备机器

+ 物理机机群
+ 虚拟机机群（使用libvirt + qume+ kvm）
+ 操作系统使用`centos 7.6`

### 2.  部署

#### 2.1 准备

关闭防火墙

```bash
systemctl disable firewalld
systemctl stop firewalld
```

禁用selinux

```bash
setenforce 0  # 临时，重启恢复

# 永久修改
vi /etc/sysconfig/selinux  # 设置SELINUX=enforcing 为 SELINUX=disabled
reboot #使修改生效
```

禁用交换区

```bash
swapoff -a  # 停用

free -m     # check一下
```

配置`yum`源，这样才能用yum安装kubernetes和docker-ce

```bash
# 添加kubernetes的yum源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF


# 添加docker-ce的yum源 并安装
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3: 更新并安装Docker-CE
sudo yum makecache fast
sudo yum -y install docker-ce
# Step 4: 开启Docker服务
sudo service docker start
sudo systemctl enable docker

# 注意：
# 官方软件源默认启用了最新的软件，您可以通过编辑软件源的方式获取各个版本的软件包。例如官方并没有将测试版本的软件源置为可用，您可以通过以下方式开启。同理可以开启各种测试版本等。
# vim /etc/yum.repos.d/docker-ee.repo
#   将[docker-ce-test]下方的enabled=0修改为enabled=1
#
# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# yum list docker-ce.x86_64 --showduplicates | sort -r
#   Loading mirror speeds from cached hostfile
#   Loaded plugins: branch, fastestmirror, langpacks
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            @docker-ce-stable
#   docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable
#   Available Packages
# Step2: 安装指定版本的Docker-CE: (VERSION例如上面的17.03.0.ce.1-1.el7.centos)
# sudo yum -y install docker-ce-[VERSION]
```

安装`kubectl` 和 `kubeadm` 和 `kubelet`

```bash
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet && systemctl start kubelet
```



#### 2.2 安装

`kubeadm`相关命令介绍：

```bash
kubeadm init    ## 启动一个 Kubernetes 主节点
kubeadm join    ## 启动一个 Kubernetes 工作节点并且将其加入到集群
kubeadm upgrade ## 更新一个 Kubernetes 集群到新版本
kubeadm config  ## 如果使用 v1.7.x 或者更低版本的 kubeadm 初始化集群，您需要对集群做一些配置以便使用 kubeadm upgrade 命令
kubeadm token   ## 管理 kubeadm join 使用的令牌
kubeadm reset   ## 还原 kubeadm init 或者 kubeadm join 对主机所做的任何更改
```

`kubeadm init`的参数:

```bash
--apiserver-advertise-address string
## API Server将要广播的监听地址。如指定为 `0.0.0.0` 将使用缺省的网卡地址。
--apiserver-bind-port int32     缺省值: 6443
## API Server绑定的端口
--apiserver-cert-extra-sans stringSlice
## 可选的额外提供的证书主题别名（SANs）用于指定API Server的服务器证书。可以是IP地址也可以是DNS名称。
--cert-dir string     缺省值: "/etc/kubernetes/pki"
## 证书的存储路径。
--config string
## kubeadm配置文件的路径。警告：配置文件的功能是实验性的。
--cri-socket string     缺省值: "/var/run/dockershim.sock"
## 指明要连接的CRI socket文件
--dry-run
## 不会应用任何改变；只会输出将要执行的操作。
--feature-gates string
## 键值对的集合，用来控制各种功能的开关。可选项有:
Auditing=true|false (当前为ALPHA状态 - 缺省值=false)
CoreDNS=true|false (缺省值=true)
DynamicKubeletConfig=true|false (当前为BETA状态 - 缺省值=false)
-h, --help
## 获取init命令的帮助信息
--ignore-preflight-errors stringSlice
## 忽视检查项错误列表，列表中的每一个检查项如发生错误将被展示输出为警告，而非错误。 例如: 'IsPrivilegedUser,Swap'. 如填写为 'all' 则将忽视所有的检查项错误。
--kubernetes-version string     缺省值: "stable-1"
## 为control plane选择一个特定的Kubernetes版本。
--node-name string
## 指定节点的名称。
--pod-network-cidr string
## 指明pod网络可以使用的IP地址段。 如果设置了这个参数，control plane将会为每一个节点自动分配CIDRs。
--service-cidr string     缺省值: "10.96.0.0/12"
## 为service的虚拟IP地址另外指定IP地址段
--service-dns-domain string     缺省值: "cluster.local"
## 为services另外指定域名, 例如： "myorg.internal".
--skip-token-print
## 不打印出由 `kubeadm init` 命令生成的默认令牌。
--token string
## 这个令牌用于建立主从节点间的双向受信链接。格式为 [a-z0-9]{6}\.[a-z0-9]{16} - 示例： abcdef.0123456789abcdef
--token-ttl duration     缺省值: 24h0m0s
## 令牌被自动删除前的可用时长 (示例： 1s, 2m, 3h). 如果设置为 '0', 令牌将永不过期。
```

安装`master`节点

```bash
kubeadm init --kubernetes-version=vx.x.x --pod-network-cidr=10.244.0.0/16

# vx.x.x 可以自行指定，但需要外网拉取镜像，一般拉取失败，所以写个脚本，用国内的镜像站拉取，然后再docker tag一下

# 脚本
#!/bin/bash
while read line
do
  echo "handle:" $line "\n" 
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$line
  docker tag  registry.cn-hangzhou.aliyuncs.com/google_containers/$line k8s.gcr.io/$line
done <$"pull.list"

# pull.list   版本号依据自己需要
kube-controller-manager:v1.19.3
kube-apiserver:v1.19.3
kube-scheduler:v1.19.3
pause:3.2
etcd:3.4.13-0
coredns:1.7.0
kube-proxy:v1.19.3

# 重新init
kubeadm init --kubernetes-version=v1.19.3 --pod-network-cidr=10.244.0.0/16
```

完成以后，复制配置文件到普通的用户home目录下：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

将node节点加入进来

```bash
# token自己查
kubeadm join 192.168.140.134:6443 --token XXX \
    --discovery-token-ca-cert-hash XXX
```



#### 2.3 安装网络插件

安装flannel插件，github：https://github.com/coreos/flannel ，查看readme.md

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

如果镜像拉不下来，接入代理直接拉取上面yaml里指定的image，使用`docker save` 和 `docker load`, 进行镜像传递。