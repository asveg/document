# Kubeadm部署集群

## 环境说明

| 节点名称 | ip地址        | 部署说明                          | Pod 网段        | Service网段    | 系统说明   |
| -------- | ------------- | --------------------------------- | --------------- | -------------- | ---------- |
| master   | 192.168.56.11 | docker、kubeadm、kubectl、kubelet | `10.244.0.0/16` | `10.96.0.0/12` | Centos 7.4 |
| node01   | 192.168.56.12 | docker、kubeadm、kubelet          | `10.244.0.0/16` | `10.96.0.0/12` | Centos 7.4 |
| node02   | 192.168.56.13 | docker、kubeadm、kubelet          | `10.244.0.0/16` | `10.96.0.0/12` | Centos 7.4 |

## 初始化配置

配置主机名

```sh
设置永久主机名称，然后重新登录
$ sudo hostnamectl set-hostname master
$ sudo hostnamectl set-hostname node1
$ sudo hostnamectl set-hostname node2
```

修改 /etc/hosts  文件，添加主机名和 IP 的对应关系

```sh
cat << eof > /etc/hosts
192.168.10.103 master
192.168.10.104 node1
192.168.10.105 node2
eof
```

 同步系统时间

```sh
yum -y install ntpdate
ntpdate cn.pool.ntp.org
```

 关闭防火墙

```sh
systemctl stop firewalld; systemctl disable firewalld　
```

关闭 swap 分区
如果开启了 swap 分区，kubelet 会启动失败(可以通过将参数 --fail-swap-on 设置为false 来忽略 swap on)，故需要在每台机器上关闭 swap 分区

```sh
swapoff -a
# 为了防止开机自动挂载 swap 分区，可以注释  /etc/fstab  中相应的条目
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

关闭 SELinux

```sh
 setenforce 0
 sed -i s#SELINUX=enforcing#SELINUX=disabled# /etc/selinux/config 
```

 

## 配置安装源

添加docker-ce 源信息

```
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
```


配置kubernetes仓库

```sh
[root@master ~]# cat << eof > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes Repo
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
gpgcheck=0
enable=1
eof
```

更新yum仓库

```sh
[root@master yum.repos.d]# yum clean all
[root@master yum.repos.d]# yum repolist
repo id                         repo name                                                    status
base                                    base                                                                                          9,363
docker-ce-stable/x86_64         Docker CE Stable - x86_64                                     20
epel/x86_64                     Extra Packages for Enterprise Linux 7 - x86_64                12,663
kubernetes                      Kubernetes Repo                                               246
repolist: 22,292
```

## 安装配置docker
```sh
yum install -y yum-utils device-mapper-persistent-data lvm2
yum -y install docker-ce
```

添加加速器到配置文件

```sh
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://registry.docker-cn.com"]
}
EOF
```

启动服务

```sh
systemctl daemon-reload; systemctl start docker; systemctl enable docker.service
```



## 安装部署kubeadm

```sh
yum install -y kubelet-1.19.5 kubeadm-1.19.5 kubectl-1.19.5
```

打开iptables内生的桥接相关功能，已经默认开启了，没开启的自行开启

```sh
modprobe br_netfilter
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

配置启动kubelet服务
修改配置文件

```sh
cat << eof > /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
KUBE_PROXY=MODE=ipvs
eof
```

开机自启

```sh
systemctl enable kubelet.service; systemctl start kubelet.service
```

配置kubelet使用的cgroup驱动。

在使用docker时，kubeadm会自动检测docker使用的cgroup驱动, 其它的CRI需要手动修改配置文件：
`/etc/sysconfig/kubelet` CentOS系
`/etc/default/kubelet` Ubuntu系

配置文件里可以添加额外的kubelet参数。
这里还是手动添加一下配置，不添加也没事。

```
KUBELET_EXTRA_ARGS=--cgroup-driver=systemd
```

暂时不需要启动kubelet， 初始化的时候会自动启动。



下载镜像

```sh
cat << 'eof' > get-k8s-images.sh
#!/bin/bash
# Script For Quick Pull K8S Docker Images
# Modify according to your own version to download the corresponding mirror
KUBE_VERSION=v1.19.5
PAUSE_VERSION=3.2
CORE_DNS_VERSION=1.6.7
ETCD_VERSION=3.4.3-0

# pull kubernetes images from hub.docker.com

docker pull kubeimage/kube-proxy-amd64:$KUBE_VERSION
docker pull kubeimage/kube-controller-manager-amd64:$KUBE_VERSION
docker pull kubeimage/kube-apiserver-amd64:$KUBE_VERSION
docker pull kubeimage/kube-scheduler-amd64:$KUBE_VERSION

# pull aliyuncs mirror docker images

docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$CORE_DNS_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:$ETCD_VERSION

# retag to k8s.gcr.io prefix

docker tag kubeimage/kube-proxy-amd64:$KUBE_VERSION  k8s.gcr.io/kube-proxy:$KUBE_VERSION
docker tag kubeimage/kube-controller-manager-amd64:$KUBE_VERSION k8s.gcr.io/kube-controller-manager:$KUBE_VERSION
docker tag kubeimage/kube-apiserver-amd64:$KUBE_VERSION k8s.gcr.io/kube-apiserver:$KUBE_VERSION
docker tag kubeimage/kube-scheduler-amd64:$KUBE_VERSION k8s.gcr.io/kube-scheduler:$KUBE_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION k8s.gcr.io/pause:$PAUSE_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$CORE_DNS_VERSION k8s.gcr.io/coredns:$CORE_DNS_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:$ETCD_VERSION k8s.gcr.io/etcd:$ETCD_VERSION

# untag origin tag, the images won't be delete.

docker rmi kubeimage/kube-proxy-amd64:$KUBE_VERSION
docker rmi kubeimage/kube-controller-manager-amd64:$KUBE_VERSION
docker rmi kubeimage/kube-apiserver-amd64:$KUBE_VERSION
docker rmi kubeimage/kube-scheduler-amd64:$KUBE_VERSION
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$CORE_DNS_VERSION
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:$ETCD_VERSION
eof
```

添加脚本执行并执行脚本

```sh
chmod +x get-k8s-images.sh
./get-k8s-images.sh  
```

## 初始化master节点

使用kubeadm init初始化

**方式一:**

```sh
kubeadm init \
  --kubernetes-version=v1.19.5 \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --ignore-preflight-errors=Swap
```

注解：

* --kubernetes-version：指定kubeadm版本；我这里下载的时候kubeadm最高时1.19.5版本
*  --pod-network-cidr：指定pod所属网络
*  --service-cidr：指定service网段
*  --ignore-preflight-errors=Swap: 忽略 swap报错

**方式二:** 该方式不需要执行get-k8s-images.sh脚本

```sh
kubeadm init \
  --image-repository=registry.aliyuncs.com/google_containers \
  --kubernetes-version=v1.19.5 \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/16  \
  --ignore-preflight-errors=Swap
```


初始化命令成功后，创建.kube目录
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## node01节点加入集群

将init生成的以下命令添加到node节点上

```sh
kubeadm join 192.168.10.225:6443 --token 8mk3nb.pauj3ipu0je6wtax \
    --discovery-token-ca-cert-hash sha256:c5ac18010cdc93f3df88e17c82ff8c5d30e15e8393b44f6ba427bd868f9bc6b4 
```

该令牌用于主节点和加入节点之间的相互认证。这里包含的令牌是秘密的。保持安全，因为拥有此令牌的任何人都可以向集群添加经过身份验证的节点。可以使用该`kubeadm token`命令列出，创建和删除这些令牌。到此，集群的初始化已经完成，可以使用kubectl get cs进行查看集群的健康状态信息

**kubeadm join 使用**

此命令用来初始化 Kubernetes 工作节点并将其加入集群

```
kubeadm join ip:port --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
```

指定token, 主节点ip:port， 以及ca证书的hash。

如果token忘记, 可以通过 `kubeadm token list`查看token。

```go
[root@master ~]# kubeadm token list
TOKEN                     TTL         EXPIRES                     USAGES                   DESCRIPTION                                                EXTRA GROUPS9zjetu.b0cwkxhb7xo77k98   23h         2020-12-24T16:27:29+08:00   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token
```

token有效期是24小时，如果过期了，执行下面的命令，会显示新的token。

```go
kubeadm token create
```

或者这个命令直接打印出来跟init一样的，可以直接执行的join命令。

```go
kubeadm token create --print-join-command
```

CA_HASH忘记，通过执行下面的命令计算CA证书的HASH。

```go
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

join节点就是通过bootstrap-token做临时认证，然后获取证书。那个token就是bootstrap-token。
主节点可以看到自动签署的证书。 而CA证书的HASH主要是kubelet用来验证主节点的。

```sh
[root@master ~]# kubectl get csr
NAME        AGE   SIGNERNAME                                    REQUESTOR                 CONDITION
csr-wdr47   13m   kubernetes.io/kube-apiserver-client-kubelet   system:bootstrap:9zjetu   Approved,Issued
[root@k8sadminmaster1 ~]# 
```

**使用kubectl命令查询集群状态**

```sh
[root@k8s-master ~]# kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-0               Healthy   {"health": "true"}
[root@k8s-master ~]# kubectl get node
NAME         STATUS     ROLES     AGE       VERSION
k8s-master   NotReady   master    43m       v1.19.5
k8s-node01   NotReady     <none>    1d        v1.19.5
```

从上面的结果可以看到，master的组件controller-manager、scheduler、etcd都处于正常状态。那么apiserver到哪去了？要知道kubectl是通过apiserver进行通信，从而在etcd中获取到集群的状态信息，所以可以获取到集群的状态信息，即表示apiserver是处于正常运行的状态。使用kubectl get node获取节点信息，可以看到master节点的状态是NotReady，这是因为还没有部署好Pod网络。

## 安装网络插件CNI

安装Pod网络插件，用于保证Pod之间的相互通信。在每个集群当中只能有一个Pod网络，在部署flannel之前需要更改的内核参数将桥接的IPv4的流量进行转发给iptables链，这是CNI插件的运行的前提要求。

```
[root@master ~]# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/c5d10c8/Documentation/kube-flannel.yml
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds created

[root@master ~]# kubectl get node　　#再查看master节点的状态信息，就是已经是Ready状态了
NAME         STATUS    ROLES     AGE       VERSION
master   Ready     master    3h        v1.19.5
node01   Ready     <none>    1d        v1.19.5
```

　

## 增加集群的节点node02

```
[root@node02 ~]# yum install -y docker kubeadm-1.19.5 kubelet-1.19.5
[root@node02 ~]# systemctl enable docker kubelet
[root@node02 ~]# systemctl start docker
```

同样预先拉取好node节点所需镜像，在此处犯错的还有，根据官方说明tonken的默认有效时间为24h，由于时间差，导致这里的token失效，可以使用kubeadm token list查看token，发现之前初始化的tonken已经失效了。

```
[root@master ~]# kubeadm token list
TOKEN                     TTL       EXPIRES                     USAGES                   DESCRIPTION   EXTRA GROUPS

dx7mko.j2ug1lqjra5bf6p2   <invalid>   2018-08-22T18:15:43-04:00   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token
```

那么此处需要重新生成token，生成的方法如下：

```
[root@master ~]# kubeadm token create
1vxhuq.qi11t7yq2wj20cpe
```

如果没有值--discovery-token-ca-cert-hash，可以通过在master节点上运行以下命令链来获取：

```
[root@master ~]# openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'

8cb2de97839780a412b93877f8507ad6c94f73add17d5d7058e91741c9d5ec78
```

此时，再运行kube join命令将node02加入到集群当中，此处的--discovery-token-ca-cert-hash依旧可以使用初始化时的证书

```sh
[root@node02 ~]# kubeadm join 192.168.56.11:6443 --token 1vxhuq.qi11t7yq2wj20cpe --discovery-token-ca-cert-hash sha256:93fe958796db44dcc23764cb8d9b6a2e67bead072e51a3d4d3c2d36b5d1007cf

[root@master ~]# kubectl get node
NAME         STATUS    ROLES     AGE       VERSION
master   Ready     master    1d        v1.19.5
node01   Ready     <none>    1d        v1.19.5
node02   Ready     <none>    2h        v1.19.5
```

## k8s集群卸载方式

如果在集群安装过程中有遇到其他问题，可以使用以下命令进行重置：

```sh
$ kubeadm reset
$ ifconfig cni0 down && ip link delete cni0
$ ifconfig flannel.1 down && ip link delete flannel.1
$ rm -rf /var/lib/cni/

# 删除rpm包
rpm -qa|grep kube*|xargs rpm --nodeps -e
# 删除容器及镜像
docker images -qa|xargs docker rmi -f
```

