# Kubeadm集群部署

## 一、环境说明

| 节点名称   | ip地址        | 部署说明                          | Pod 网段        | Service网段    | 系统说明   |
| ---------- | ------------- | --------------------------------- | --------------- | -------------- | ---------- |
| k8s-master | 192.168.56.11 | docker、kubeadm、kubectl、kubelet | `10.244.0.0/16` | `10.96.0.0/12` | Centos 7.7 |
| k8s-node01 | 192.168.56.12 | docker、kubeadm、kubelet          | `10.244.0.0/16` | `10.96.0.0/12` | Centos 7.7 |
| k8s-node02 | 192.168.56.13 | docker、kubeadm、kubelet          | `10.244.0.0/16` | `10.96.0.0/12` | Centos 7.7 |

 

### （1）配置源

```
[root@k8s-master ~]# cd /etc/yum.repos.d/

配置阿里云的源：https://opsx.alibaba.com/mirror

[root@k8s-master yum.repos.d]# wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo　　#配置dokcer源

[root@k8s-master ~]# cat <<EOF > /etc/yum.repos.d/kubernetes.repo　　#配置kubernetes源
> [kubernetes]
> name=Kubernetes
> baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
> enabled=1
> gpgcheck=1
> repo_gpgcheck=1
> gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
> EOF

[root@k8s-master yum.repos.d]# yum repolist #查看可用源
```

将源拷贝到node01和node02节点

```
[root@k8s-master yum.repos.d]# scp kubernetes.repo docker-ce.repo k8s-node1:/etc/yum.repos.d/
kubernetes.repo                         100%  276   276.1KB/s   00:00    
docker-ce.repo                          100% 2640     1.7MB/s   00:00    
[root@k8s-master yum.repos.d]# scp kubernetes.repo docker-ce.repo k8s-node2:/etc/yum.repos.d/
kubernetes.repo                         100%  276   226.9KB/s   00:00    
docker-ce.repo                          100% 2640     1.7MB/s   00:00    
```

### （2）安装docker、kubelet、kubeadm、还有命令行工具kubectl

```
[root@k8s-master yum.repos.d]# yum install -y docker-ce kubelet kubeadm kubectl 
[root@k8s-master ~]# systemctl start docker
[root@k8s-master ~]# systemctl enable docker
```

启动docker，docker需要到自动到docker仓库中所依赖的镜像文件，这些镜像文件会因为在国外仓库而下载无法完成，所以最好预先下载镜像文件，kubeadm也可以支持本地私有仓库进行获取镜像文件。

### （3）预先下载镜像

```
在master节点上使用docker pull拉取镜像，再通过tag打标签
docker pull xiyangxixia/k8s-proxy-amd64:v1.11.1
docker tag xiyangxixia/k8s-proxy-amd64:v1.11.1 k8s.gcr.io/kube-proxy-amd64:v1.11.1
docker pull xiyangxixia/k8s-scheduler:v1.11.1
docker tag xiyangxixia/k8s-scheduler:v1.11.1 k8s.gcr.io/kube-scheduler-amd64:v1.11.1
docker pull xiyangxixia/k8s-controller-manager:v1.11.1
docker tag xiyangxixia/k8s-controller-manager:v1.11.1 k8s.gcr.io/kube-controller-manager-amd64:v1.11.1
docker pull xiyangxixia/k8s-apiserver-amd64:v1.11.1
docker tag xiyangxixia/k8s-apiserver-amd64:v1.11.1 k8s.gcr.io/kube-apiserver-amd64:v1.11.1
docker pull xiyangxixia/k8s-etcd:3.2.18
docker tag xiyangxixia/k8s-etcd:3.2.18 k8s.gcr.io/etcd-amd64:3.2.18
docker pull xiyangxixia/k8s-coredns:1.1.3
docker tag xiyangxixia/k8s-coredns:1.1.3 k8s.gcr.io/coredns:1.1.3
docker pull xiyangxixia/k8s-pause:3.1
docker tag xiyangxixia/k8s-pause:3.1 k8s.gcr.io/pause:3.1
docker pull xiyangxixia/k8s-flannel:v0.10.0-s390x
docker tag xiyangxixia/k8s-flannel:v0.10.0-s390x quay.io/coreos/flannel:v0.10.0-s390x
docker pull xiyangxixia/k8s-flannel:v0.10.0-ppc64le
docker tag xiyangxixia/k8s-flannel:v0.10.0-ppc64le quay.io/coreos/flannel:v0.10.0-ppc64l
docker pull xiyangxixia/k8s-flannel:v0.10.0-arm
docker tag xiyangxixia/k8s-flannel:v0.10.0-arm quay.io/coreos/flannel:v0.10.0-arm
docker pull xiyangxixia/k8s-flannel:v0.10.0-amd64
docker tag xiyangxixia/k8s-flannel:v0.10.0-amd64 quay.io/coreos/flannel:v0.10.0-amd64

node节点上拉取的镜像

docker pull xiyangxixia/k8s-pause:3.1
docker tag xiyangxixia/k8s-pause:3.1 k8s.gcr.io/pause:3.1
docker pull xiyangxixia/k8s-proxy-amd64:v1.11.1
docker tag xiyangxixia/k8s-proxy-amd64:v1.11.1 k8s.gcr.io/kube-proxy-amd64:v1.11.1
docker pull xiyangxixia/k8s-flannel:v0.10.0-amd64
docker tag xiyangxixia/k8s-flannel:v0.10.0-amd64 quay.io/coreos/flannel:v0.10.0-amd64
```

### （4）kubeadm初始化集群

```
[root@k8s-master ~]# vim /etc/sysconfig/kubelet 　　#修改kubelet禁止提示swap警告
KUBELET_EXTRA_ARGS="--fail-swap-on=false" #如果配置了swap不然提示出错信息
更改kubelet配置，不提示swap警告信息，最好关闭swap
[root@k8s-master ~]# swapoff -a　　#关闭swap

[root@k8s-master ~]# kubeadm init --kubernetes-version=v1.11.1 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap   　　#初始化 
[init] using Kubernetes version: v1.11.1
[preflight] running pre-flight checks
I0821 18:14:22.223765   18053 kernel_validator.go:81] Validating kernel version
I0821 18:14:22.223894   18053 kernel_validator.go:96] Validating kernel config
    [WARNING SystemVerification]: docker version is greater than the most recently validated version. Docker version: 18.06.0-ce. Max validated version: 17.03
[preflight/images] Pulling images required for setting up a Kubernetes cluster
[preflight/images] This might take a minute or two, depending on the speed of your internet connection
[preflight/images] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[preflight] Activating the kubelet service
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.56.11]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated sa key and public key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] Generated etcd/ca certificate and key.
[certificates] Generated etcd/server certificate and key.
[certificates] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [127.0.0.1 ::1]
[certificates] Generated etcd/peer certificate and key.
[certificates] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.56.11 127.0.0.1 ::1]
[certificates] Generated etcd/healthcheck-client certificate and key.
[certificates] Generated apiserver-etcd-client certificate and key.
[certificates] valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[controlplane] wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests" 
[init] this might take a minute or longer if the control plane images have to be pulled
[apiclient] All control plane components are healthy after 51.033696 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.11" in namespace kube-system with the configuration for the kubelets in the cluster
[markmaster] Marking the node k8s-master as master by adding the label "node-role.kubernetes.io/master=''"
[markmaster] Marking the node k8s-master as master by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "k8s-master" as an annotation
[bootstraptoken] using token: dx7mko.j2ug1lqjra5bf6p2
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.56.11:6443 --token dx7mko.j2ug1lqjra5bf6p2 --discovery-token-ca-cert-hash sha256:93fe958796db44dcc23764cb8d9b6a2e67bead072e51a3d4d3c2d36b5d1007cf
```

如果是普通用户部署，要使用kubectl，需要配置kubectl的环境变量，这些命令也是`kubeadm init`输出的一部分：

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

如果是使用root用户部署，可以使用export进行定义环境变量

```
[root@k8s-master ~]# export KUBECONFIG=/etc/kubernetes/admin.conf
```

此处也需要记录输出的`kubeadm join`命令，后面的node节点加入到集群就需要用到此命令：``

```
kubeadm join 192.168.56.11:6443 --token dx7mko.j2ug1lqjra5bf6p2 --discovery-token-ca-cert-hash sha256:93fe958796db44dcc23764cb8d9b6a2e67bead072e51a3d4d3c2d36b5d1007cf
```

该令牌用于主节点和加入节点之间的相互认证。这里包含的令牌是秘密的。保持安全，因为拥有此令牌的任何人都可以向集群添加经过身份验证的节点。可以使用该`kubeadm token`命令列出，创建和删除这些令牌。到此，集群的初始化已经完成，可以使用kubectl get cs进行查看集群的健康状态信息：

```
[root@k8s-master ~]# kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-0               Healthy   {"health": "true"}
[root@k8s-master ~]# kubectl get node
NAME         STATUS     ROLES     AGE       VERSION
k8s-master   NotReady   master    43m       v1.11.2
```

从上面的结果可以看到，master的组件controller-manager、scheduler、etcd都处于正常状态。那么apiserver到哪去了？要知道kubectl是通过apiserver进行通信，从而在etcd中获取到集群的状态信息，所以可以获取到集群的状态信息，即表示apiserver是处于正常运行的状态。使用kubectl get node获取节点信息，可以看到master节点的状态是NotReady，这是因为还没有部署好Pod网络。

### （5）安装网络插件CNI

安装Pod网络插件，用于保证Pod之间的相互通信。在每个集群当中只能有一个Pod网络，在部署flannel之前需要更改的内核参数将桥接的IPv4的流量进行转发给iptables链，这是CNI插件的运行的前提要求。

```
[root@k8s-master ~]# cat /proc/sys/net/bridge/bridge-nf-call-iptables
1
[root@k8s-master ~]# cat /proc/sys/net/bridge/bridge-nf-call-ip6tables
1
[root@k8s-master ~]# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/c5d10c8/Documentation/kube-flannel.yml
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds created

[root@k8s-master ~]# kubectl get node　　#再查看master节点的状态信息，就是已经是Ready状态了
NAME         STATUS    ROLES     AGE       VERSION
k8s-master   Ready     master    3h        v1.11.2
```

###  （6）node01节点加入集群

```
[root@k8s-node01 ~]# yum install -y docker kubeadm kubelet
[root@k8s-node01 ~]# systemctl enable dokcer kubelet
[root@k8s-node01 ~]# systemctl start docker

这里需要提前拉取好镜像，如果docker配置了代理另说，按本次方法，提前拉取node节点所需要的镜像。

[root@k8s-node01 ~]# kubeadm join 192.168.56.11:6443 --token dx7mko.j2ug1lqjra5bf6p2 --discovery-token-ca-cert-hash sha256:93fe958796db44dcc23764cb8d9b6a2e67bead072e51a3d4d3c2d36b5d1007cf

[root@k8s-master ~]# kubectl get node　　
NAME         STATUS     ROLES     AGE       VERSION
k8s-master   Ready      master    7h        v1.11.2
k8s-node01   NotReady   <none>    3h        v1.11.2
```

加入集群后，查看节点状态信息，看到node01节点的状态为NotReady，是因为node01节点上还没有镜像或者是还在拉取镜像。等待拉取完镜像就会启动对应的Pod。

```
[root@k8s-node01 ~]# docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS               NAMES
5502c29b43df        f0fad859c909           "/opt/bin/flanneld -…"   3 minutes ago       Up 3 minutes                            k8s_kube-flannel_kube-flannel-ds-pgpr7_kube-system_23dc27e3-a5af-11e8-84d2-000c2972dc1f_1
db1cc0a6fec4        d5c25579d0ff           "/usr/local/bin/kube…"   3 minutes ago       Up 3 minutes                            k8s_kube-proxy_kube-proxy-vxckf_kube-system_23dc0141-a5af-11e8-84d2-000c2972dc1f_0
bc54ad3399e8        k8s.gcr.io/pause:3.1   "/pause"                 9 minutes ago       Up 9 minutes                            k8s_POD_kube-proxy-vxckf_kube-system_23dc0141-a5af-11e8-84d2-000c2972dc1f_0
cbfca066b71d        k8s.gcr.io/pause:3.1   "/pause"                 10 minutes ago      Up 10 minutes                           k8s_POD_kube-flannel-ds-pgpr7_kube-system_23dc27e3-a5af-11e8-84d2-000c2972dc1f_0

[root@k8s-master ~]# kubectl get pods -n kube-system -o wide
NAME                                 READY     STATUS    RESTARTS   AGE       IP              NODE
coredns-78fcdf6894-nmcmz             1/1       Running   0          1d        10.244.0.3      k8s-master
coredns-78fcdf6894-p5pfm             1/1       Running   0          1d        10.244.0.2      k8s-master
etcd-k8s-master                      1/1       Running   1          1d        192.168.56.11   k8s-master
kube-apiserver-k8s-master            1/1       Running   8          1d        192.168.56.11   k8s-master
kube-controller-manager-k8s-master   1/1       Running   4          1d        192.168.56.11   k8s-master
kube-flannel-ds-n5c86                1/1       Running   0          1d        192.168.56.11   k8s-master
kube-flannel-ds-pgpr7                1/1       Running   1          1d        192.168.56.12   k8s-node01
kube-proxy-rxlt7                     1/1       Running   1          1d        192.168.56.11   k8s-master
kube-proxy-vxckf                     1/1       Running   0          1d        192.168.56.12   k8s-node01
kube-scheduler-k8s-master            1/1       Running   2          1d        192.168.56.11   k8s-master

[root@k8s-master ~]# kubectl get node　　#此时再查看状态已经变成Ready
NAME         STATUS    ROLES     AGE       VERSION
k8s-master   Ready     master    1d        v1.11.2
k8s-node01   Ready     <none>    1d        v1.11.2
```

### （7）增加集群的节点node02

```
[root@k8s-node02 ~]# yum install -y docker kubeadm kubelet
[root@k8s-node02 ~]# systemctl enable docker kubelet
[root@k8s-node02 ~]# systemctl start docker
```

同样预先拉取好node节点所需镜像，在此处犯错的还有，根据官方说明tonken的默认有效时间为24h，由于时间差，导致这里的token失效，可以使用kubeadm token list查看token，发现之前初始化的tonken已经失效了。

```
[root@k8s-master ~]# kubeadm token list
TOKEN                     TTL       EXPIRES                     USAGES                   DESCRIPTION   EXTRA GROUPS

dx7mko.j2ug1lqjra5bf6p2   <invalid>   2018-08-22T18:15:43-04:00   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token
```

那么此处需要重新生成token，生成的方法如下：

```
[root@k8s-master ~]# kubeadm token create
1vxhuq.qi11t7yq2wj20cpe
```

如果没有值--discovery-token-ca-cert-hash，可以通过在master节点上运行以下命令链来获取：

```
[root@k8s-master ~]# openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'

8cb2de97839780a412b93877f8507ad6c94f73add17d5d7058e91741c9d5ec78
```

此时，再运行kube join命令将node02加入到集群当中，此处的--discovery-token-ca-cert-hash依旧可以使用初始化时的证书

```
[root@k8s-node02 ~]# kubeadm join 192.168.56.11:6443 --token 1vxhuq.qi11t7yq2wj20cpe --discovery-token-ca-cert-hash sha256:93fe958796db44dcc23764cb8d9b6a2e67bead072e51a3d4d3c2d36b5d1007cf

[root@k8s-master ~]# kubectl get node
NAME         STATUS    ROLES     AGE       VERSION
k8s-master   Ready     master    1d        v1.11.2
k8s-node01   Ready     <none>    1d        v1.11.2
k8s-node02   Ready     <none>    2h        v1.11.2
```

如果在集群安装过程中有遇到其他问题，可以使用以下命令进行重置：

```
$ kubeadm reset
$ ifconfig cni0 down && ip link delete cni0
$ ifconfig flannel.1 down && ip link delete flannel.1
$ rm -rf /var/lib/cni/
```

安装比较繁琐,这里我提供了自己的脚本:[kubernetes-install](https://github.com/wangjinh/kubeadm-install.git)



# Coredns和Dashboard

### 一、CoreDNS部署

在 Cluster 中，除了可以通过 Cluster IP 访问 Service，Kubernetes 还提供了更为方便的 DNS 访问。

**（1）编辑coredns.yaml文件**

```yaml
[root@linux-node1 ~]# vim coredns.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
  labels:
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: Reconcile
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: EnsureExists
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
  labels:
      addonmanager.kubernetes.io/mode: EnsureExists
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local. in-addr.arpa ip6.arpa {
            pods insecure
            upstream
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        proxy . /etc/resolv.conf
        cache 30
    }
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: coredns
  template:
    metadata:
      labels:
        k8s-app: coredns
    spec:
      serviceAccountName: coredns
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      containers:
      - name: coredns
        image: coredns/coredns:1.0.6
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: coredns
  clusterIP: 10.1.0.2 #改为本地的dns
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
```

**（2）创建coredns**

```
[root@linux-node1 ~]# kubectl create -f coredns.yaml 
serviceaccount "coredns" created
clusterrole.rbac.authorization.k8s.io "system:coredns" created
clusterrolebinding.rbac.authorization.k8s.io "system:coredns" created
configmap "coredns" created
deployment.extensions "coredns" created
service "coredns" created
```

**（3）查看coredns服务**

```
[root@linux-node1 ~]# kubectl get deployment -n kube-system
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
coredns   2         2         2            0           1m

[root@linux-node1 ~]# kubectl get svc -n kube-system
NAME      TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
coredns   ClusterIP   10.1.0.2     <none>        53/UDP,53/TCP   1m

[root@linux-node1 ~]# kubectl get pod -n kube-system
NAME                       READY     STATUS    RESTARTS   AGE
coredns-77c989547b-d84n8   1/1       Running   0          2m
coredns-77c989547b-j4ms2   1/1       Running   0          2m
```

**（4）Pod容器中进行域名解析测试**

```
[root@linux-node1 ~]# kubectl run alpine --rm -ti --image=alpine -- /bin/sh
If you don't see a command prompt, try pressing enter.

/ # nslookup httpd-svc
nslookup: can't resolve '(null)': Name does not resolve

Name:      httpd-svc
Address 1: 10.1.230.129

/ # wget httpd-svc:8080
Connecting to httpd-svc:8080 (10.1.230.129:8080)
index.html           100% |********************************************************************************************************************************************|    45   0:00:00 ETA
```

### 二、Dashboard部署

从github上下载dashboard的yaml文件：https://github.com/unixhot/salt-kubernetes

```
[root@linux-node1 dashboard]# ll
total 20
-rw-r--r-- 1 root root  357 Aug 22 09:26 admin-user-sa-rbac.yaml
-rw-r--r-- 1 root root 4901 Aug 22 09:26 kubernetes-dashboard.yaml
-rw-r--r-- 1 root root  458 Aug 22 09:26 ui-admin-rbac.yaml
-rw-r--r-- 1 root root  477 Aug 22 09:26 ui-read-rbac.yaml

[root@linux-node1 dashboard]# kubectl create -f .
serviceaccount "admin-user" created
clusterrolebinding.rbac.authorization.k8s.io "admin-user" created
secret "kubernetes-dashboard-certs" created
serviceaccount "kubernetes-dashboard" created
role.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" created
rolebinding.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" created
deployment.apps "kubernetes-dashboard" created
service "kubernetes-dashboard" created
clusterrole.rbac.authorization.k8s.io "ui-admin" created
rolebinding.rbac.authorization.k8s.io "ui-admin-binding" created
clusterrole.rbac.authorization.k8s.io "ui-read" created
rolebinding.rbac.authorization.k8s.io "ui-read-binding" created

[root@linux-node1 dashboard]# kubectl get pods  -o wide -n kube-system
NAME                                    READY     STATUS    RESTARTS   AGE       IP           NODE
coredns-77c989547b-d84n8                1/1       Running   0          55m       10.2.99.7    192.168.56.13
coredns-77c989547b-j4ms2                1/1       Running   0          55m       10.2.76.6    192.168.56.12
kubernetes-dashboard-66c9d98865-mps22   1/1       Running   0          4m        10.2.76.12   192.168.56.12

[root@linux-node1 dashboard]# kubectl get svc -n kube-system
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
coredns                ClusterIP   10.1.0.2       <none>        53/UDP,53/TCP   56m
kubernetes-dashboard   NodePort    10.1.234.201   <none>        443:38974/TCP   5m
```

从上可以看到kubernetes的dashboard服务的ip为：10.1.234.201，其映射到宿主机的端口为38974，由于master上没有部署kube-porxy，所以需要直接访问https://192.168.56.12:38974，如图：

选择令牌登陆，获取令牌的方法如下：

```
[root@linux-node1 dashboard]# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-mz7p9
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=admin-user
              kubernetes.io/service-account.uid=c2a85113-acc9-11e8-a800-000c29ce4fa7

Type:  kubernetes.io/service-account-token

Data
====
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLW16N3A5Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJjMmE4NTExMy1hY2M5LTExZTgtYTgwMC0wMDBjMjljZTRmYTciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.V4aEkKDBcK4RkuXRzwdAyoJRBrxAnc8axLLxGCGiduwv5Qa0HFe2WQWtny6FI-MpUP-dzrxahWSwaFcKKvVdzfBuXTbnPDBkhcrpAuzDsL0vo-GwHAAl88n8yZ67QmBwPVWH2CBrrTwWqALAfR2wNKtrUEigg-qbTQ05slP8WmbeckfzHTeZpQqegO3fz0BNBrJqi2TFDaftPm_vWSEsPWzWE9AyvfiVwGrfc_mmzHpOyxXAQXQLxJunfklwt0kuENO6sRRJ2HGvZ6HnCGZYZj0p-kjh5uAv-q_X2cMPIAhXgH7gHdYeiSXvEGA2Qz6tBE2pgN6S4F_xj6b4JT7kAQ
ca.crt:     1359 bytes 
```

![img](https://images2018.cnblogs.com/blog/1349539/201808/1349539-20180831110931763-798097612.png)

**点击登录后的界面如下：**

![img](https://images2018.cnblogs.com/blog/1349539/201808/1349539-20180831112013927-1278322910.png)

 

# kubernetes命令快速创建应用

## 1、使用命令kubectl run创建应用

```
语法:
  kubectl run NAME --image=image [--env="key=value"] [--port=port] [--replicas=replicas] [--dry-run=bool]
[--overrides=inline-json] [--command] -- [COMMAND] [args...] [options]
```

 **实用举例：**

```
[root@k8s-master ~]# kubectl run nginx-deploy --image=nginx:1.14-alpine --port=80 --replicas=1    #创建一个nginx的应用，副本数为1
deployment.apps/nginx-deploy created

[root@k8s-master ~]# kubectl get deployment　　#获取应用信息，查看应用是否符合预期状态
NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   1         1         1            1           40s

[root@k8s-master ~]# kubectl get pods　　#获取pod信息
NAME                          READY     STATUS    RESTARTS   AGE
nginx-deploy-5b595999-44zwq   1/1       Running   0          1m


[root@k8s-master ~]# kubectl get pods -o wide    #查看pod运行在哪个节点上
NAME                          READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-deploy-5b595999-44zwq   1/1       Running   0          1m        10.244.2.2   k8s-node02
```

从上面创建的应用可以得知，nginx-deploy应用的pod的ip为10.244.2.2，这是一个pod ip，仅仅可以在集群内部访问，如下：

```
[root@k8s-master ~]# curl 10.244.2.2 -I
HTTP/1.1 200 OK
Server: nginx/1.14.0
Date: Thu, 28 Feb 2019 06:13:03 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Fri, 06 Jul 2018 16:53:43 GMT
Connection: keep-alive
ETag: "5b3f9e97-264"
Accept-Ranges: bytes

[root@k8s-node01 ~]# curl 10.244.2.2 -I
HTTP/1.1 200 OK
Server: nginx/1.14.0
Date: Thu, 28 Feb 2019 06:12:04 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Fri, 06 Jul 2018 16:53:43 GMT
Connection: keep-alive
ETag: "5b3f9e97-264"
Accept-Ranges: bytes

[root@k8s-node02 ~]# curl 10.244.2.2 -I                
HTTP/1.1 200 OK
Server: nginx/1.14.0
Date: Thu, 23 Aug 2018 09:22:18 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Fri, 06 Jul 2018 16:53:43 GMT
Connection: keep-alive
ETag: "5b3f9e97-264"
Accept-Ranges: bytes
```

这里要注意的是pod的客户端有2类，1类是其他pod，1类是集群外部客户端，那么集群外部的客户端如何访问到pod呢？pod的地址是随时变化的，假设先删除创建的pod：

```
[root@k8s-master ~]# kubectl delete pods nginx-deploy-5b595999-44zwq
pod "nginx-deploy-5b595999-44zwq" deleted
```

要明白pod是通过控制器进行管理的，当控制器发现pod的状态不满足预期的状态时，将会重新创建一个pod

```
[root@k8s-master ~]# kubectl get pods -o wide    #由于在node01节点上没有镜像，需要重新下载
NAME                          READY     STATUS              RESTARTS   AGE       IP        NODE
nginx-deploy-5b595999-872c7   0/1       ContainerCreating   0          24s       <none>    k8s-node01
[root@k8s-master ~]# kubectl get pods -o wide
NAME                          READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-deploy-5b595999-872c7   1/1       Running   0          56s       10.244.1.2   k8s-node01
```

此时可以看到新建的pod的ip地址已经更改了，并且本次创建的pod是在node01节点上，这样就需要提供一个固定端点，给集群外部客户端进行访问。这个固定端点就是service：

```
语法如下:
  kubectl expose (-f FILENAME | TYPE NAME) [--port=port] [--protocol=TCP|UDP] [--target-port=number-or-name]
[--name=name] [--external-ip=external-ip-of-service] [--type=type] [options]

[root@k8s-master ~]# kubectl expose deployment nginx-deploy --name=nginx --port=80 --target-port=80 --protocol=TCP　　#创建一个nginx的service
service/nginx exposed

[root@k8s-master ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   1d
nginx        ClusterIP   10.106.162.254   <none>        80/TCP    19s

[root@k8s-master ~]# curl 10.106.162.254 -I　　#通过ClusterIP进行访问nginx pod
HTTP/1.1 200 OK
Server: nginx/1.14.0
Date: Thu, 23 Aug 2018 09:38:09 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Fri, 06 Jul 2018 16:53:43 GMT
Connection: keep-alive
ETag: "5b3f9e97-264"
Accept-Ranges: bytes
```

10.106.162.254这网段依然是集群内部的网段，只能被集群内部所能访问，外部是无法通过service的ip进行访问的。那么针对pod的客户端除了通过service ip访问还可以通过service的名称进行访问，但是前提是需要对service的名称能够进行解析。而解析时是依赖coredns服务的，而我们本地的dns指向并非coredns，如下：

```
[root@k8s-master ~]# curl nginx
curl: (6) Could not resolve host: nginx; Unknown error
[root@k8s-master ~]# cat /etc/resolv.conf
# Generated by NetworkManager
nameserver 8.8.8.8
nameserver 114.114.114.114
```

下面查看一下coredns的ip地址：

```
[root@k8s-master ~]# kubectl get pods -n kube-system -o wide 
NAME                                 READY     STATUS    RESTARTS   AGE       IP              NODE
coredns-78fcdf6894-nmcmz             1/1       Running   0          1d        10.244.0.3      k8s-master
coredns-78fcdf6894-p5pfm             1/1       Running   0          1d        10.244.0.2      k8s-master
etcd-k8s-master                      1/1       Running   1          1d        192.168.56.11   k8s-master
kube-apiserver-k8s-master            1/1       Running   8          1d        192.168.56.11   k8s-master
kube-controller-manager-k8s-master   1/1       Running   4          1d        192.168.56.11   k8s-master
kube-flannel-ds-n5c86                1/1       Running   0          1d        192.168.56.11   k8s-master
kube-flannel-ds-nrcw2                1/1       Running   0          5h        192.168.56.13   k8s-node02
kube-flannel-ds-pgpr7                1/1       Running   1          1d        192.168.56.12   k8s-node01
kube-proxy-glzth                     1/1       Running   0          5h        192.168.56.13   k8s-node02
kube-proxy-rxlt7                     1/1       Running   1          1d        192.168.56.11   k8s-master
kube-proxy-vxckf                     1/1       Running   0          1d        192.168.56.12   k8s-node01
kube-scheduler-k8s-master            1/1       Running   2          1d        192.168.56.11   k8s-master
```

而一般，也不会直接通过coredns的这个pod ip地址进行访问，而是通过service进行访问，查看一下coredns的service：

```
[root@k8s-master ~]# kubectl get svc -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP   1d
```

那么就可以通过这个service ip：10.96.0.10进行解析上面的nginx服务，如下：

```
[root@k8s-master ~]# yum install -y bind-utils

[root@k8s-master ~]# dig -t A nginx.default.svc.cluster.local @10.96.0.10　　#这里需要使用完整的服务名称，否则会因为dns搜索域的问题而导致无法解析成功

; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7 <<>> -t A nginx.default.svc.cluster.local @10.96.0.10
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 78
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;nginx.default.svc.cluster.local. IN    A

;; ANSWER SECTION:
nginx.default.svc.cluster.local. 5 IN    A    10.106.162.254　　#这样就可以正常解析出nginx的service ip了

;; Query time: 155 msec
;; SERVER: 10.96.0.10#53(10.96.0.10)
;; WHEN: Thu Aug 23 05:40:22 EDT 2018
;; MSG SIZE  rcvd: 107
```

那么再演示通过pod 客户端进行访问：

```
[root@k8s-master ~]# kubectl run client --image=busybox --replicas=1 -it --restart=Never　　#创建pod
[root@k8s-master ~]# kubectl exec -it client /bin/sh    #首次创建如果没进入到容器，可以使用这命令进入
/ # cat /etc/resolv.conf 　　#查看dns，这里就是自动指向coredns
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5

/ # wget -O - -q http://nginx:80　　#请求解析nginx
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
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

这就是service提供给pod的固定访问端点的使用，而pod的增删改查，并不会影响通过service进行访问，可以通过以下命令来查看service的详细信息：

```
[root@k8s-master ~]# kubectl describe svc nginx
Name:              nginx
Namespace:         default
Labels:            run=nginx-deploy
Annotations:       <none>
Selector:          run=nginx-deploy
Type:              ClusterIP
IP:                10.106.162.254
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.1.5:80　　#pod 的ip，会根据资源变化改变，但是实际访问的service 依旧有效
Session Affinity:  None
Events:            <none>
```

那么pod的增删改，service又是如何确定对pod的访问呢？这就需要通过标签选择器进行选定，无论pod的ip如何变化，但是标签不会变化，从而达到固定端点的访问效果，查看一下pod的标签：

```
[root@k8s-master ~]# kubectl get pods --show-labels
NAME                          READY     STATUS    RESTARTS   AGE       LABELS
client                        1/1       Running   0          21h       run=client
nginx-deploy-5b595999-872c7   1/1       Running   2          22h       pod-template-hash=16151555,run=nginx-deploy
```

run=nginx-deploy就是这个应用的标签，所以当pod的改变，并不会影响service的访问。

## 2、应用副本的动态伸缩

```
语法如下:
kubectl scale [--resource-version=version] [--current-replicas=count] --replicas=COUNT (-f FILENAME | TYPE NAME)

（1）创建应用myapp
[root@k8s-master ~]# kubectl run myapp --image=ikubernetes/myapp:v1 --replicas=2
deployment.apps/myapp created

[root@k8s-master ~]# kubectl get deployment 
NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
myapp          2         2         2            1           15s
nginx-deploy   1         1         1            1           22h

（2）查看pod详细信息
[root@k8s-master ~]# kubectl get pods -o wide
NAME                          READY     STATUS         RESTARTS   AGE       IP           NODE
client                        1/1       Running        0          21h       10.244.2.3   k8s-node02
client2                       1/1       Running        0          48m       10.244.1.6   k8s-node01
client3                       1/1       Running        0          27m       10.244.2.4   k8s-node02
myapp-848b5b879b-bdp7t        1/1       Running        0          26s       10.244.1.7   k8s-node01
myapp-848b5b879b-swt2c        0/1       ErrImagePull   0          26s       10.244.2.5   k8s-node02
nginx-deploy-5b595999-872c7   1/1       Running        2          22h       10.244.1.5   k8s-node01

（3）配置service端点
[root@k8s-master ~]# kubectl expose deployment myapp --name=myapp --port=80
service/myapp exposed

（4）查看服务信息
[root@k8s-master ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   2d
myapp        ClusterIP   10.106.67.242    <none>        80/TCP    14s
nginx        ClusterIP   10.106.162.254   <none>        80/TCP    21h

（5）Pod客户端访问
/ #  wget -O - -q http://myapp:80
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>

（6）副本增加到5
[root@k8s-master ~]# kubectl scale --replicas=5 deployment myapp
deployment.extensions/myapp scaled

[root@k8s-master ~]# kubectl get pods
NAME                          READY     STATUS             RESTARTS   AGE
client                        1/1       Running            0          21h
client2                       1/1       Running            0          51m
client3                       1/1       Running            0          30m
myapp-848b5b879b-6p6ml        1/1       Running            0          1m
myapp-848b5b879b-7xmnj        0/1       ImagePullBackOff   0          1m
myapp-848b5b879b-bdp7t        1/1       Running            0          3m
myapp-848b5b879b-swt2c        0/1       ImagePullBackOff   0          3m
myapp-848b5b879b-zlvl2        1/1       Running            0          1m
nginx-deploy-5b595999-872c7   1/1       Running            2          22h

（7）副本收缩到3
[root@k8s-master ~]# kubectl scale --replicas=3 deployment myapp
deployment.extensions/myapp scaled
```

## 3、应用的版本升级

```
语法如下:
kubectl set image (-f FILENAME | TYPE NAME) CONTAINER_NAME_1=CONTAINER_IMAGE_1 ... CONTAINER_NAME_N=CONTAINER_IMAGE_N
（1）版本升级为v2
[root@k8s-master ~]# kubectl set image deployment myapp myapp=ikubernetes/myapp:v2
deployment.extensions/myapp image updated

（2）查看升级过程
[root@k8s-master ~]# kubectl rollout status deployment myapp    #查看更新过程
Waiting for deployment "myapp" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "myapp" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "myapp" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "myapp" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "myapp" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "myapp" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "myapp" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "myapp" rollout to finish: 1 old replicas are pending termination...
deployment "myapp" successfully rolled out

（3）获取pod信息
[root@k8s-master ~]# kubectl get pods
NAME                          READY     STATUS    RESTARTS   AGE
client                        1/1       Running   0          21h
client2                       1/1       Running   0          53m
client3                       1/1       Running   0          33m
myapp-74c94dcb8c-2djgg        1/1       Running   0          1m
myapp-74c94dcb8c-92d9p        1/1       Running   0          28s
myapp-74c94dcb8c-nq7zt        1/1       Running   0          25s
nginx-deploy-5b595999-872c7   1/1       Running   2          22h

[root@k8s-master ~]# kubectl describe pods myapp-74c94dcb8c-2djgg

（4）pod客户端测试访问，可以看到是v2版本
/ #  wget -O - -q http://myapp:80
Hello MyApp | Version: v2 | <a href="hostname.html">Pod Name</a>
```

## 4、应用的版本回滚

**语法如下:**
　**kubectl rollout undo (TYPE NAME | TYPE/NAME) [flags] [options]**

```
[root@k8s-master ~]# kubectl rollout undo deployment myapp    #不指定版本直接回滚到上一个版本
deployment.extensions/myapp

/ #  wget -O - -q http://myapp:80
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
```

## 5、实现外部访问service

```
[root@k8s-master ~]# kubectl edit svc myapp
TYPE:CLUSTER-IP改为
TYPE:NodePort

[root@k8s-master ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        2d
myapp        NodePort    10.106.67.242    <none>        80:32432/TCP   18m
nginx        ClusterIP   10.106.162.254   <none>        80/TCP         22h
```

这里再查看service信息，可以看到myapp进行了端口映射，将myapp的80端口映射到本地32432端口，则可以使用http://192.168.56.11:32432进行访问。如图：

![img](https://images2018.cnblogs.com/blog/1349539/201808/1349539-20180824165515076-1845735488.png)

 





# Kubernetes资源清单定义

## 一、Kubernetes常用资源

以下列举的内容都是 kubernetes 中的 Object，这些对象都可以在 yaml 文件中作为一种 API 类型来配置。

| 类别               | 名称                                                         |
| ------------------ | ------------------------------------------------------------ |
| 工作负载型资源对象 | Pod Replicaset ReplicationController Deployments StatefulSets Daemonset Job CronJob |
| 服务发现及负载均衡 | Service Ingress                                              |
| 配置与存储         | Volume、Persistent Volume、CSl 、 configmap、 secret         |
| 集群资源           | Namespace Node Role ClusterRole RoleBinding ClusterRoleBinding |
| 元数据资源         | HPA PodTemplate LimitRang                                    |

## 二、理解Kubernetes中的对象

在 Kubernetes 系统中，Kubernetes 对象 是持久化的条目。Kubernetes 使用这些条目去表示整个集群的状态。特别地，它们描述了如下信息：

- 什么容器化应用在运行（以及在哪个 Node 上）
- 可以被应用使用的资源
- 关于应用如何表现的策略，比如重启策略、升级策略，以及容错策略

Kubernetes 对象是 “目标性记录” —— 一旦创建对象，Kubernetes 系统将持续工作以确保对象存在。通过创建对象，可以有效地告知 Kubernetes 系统，所需要的集群工作负载看起来是什么样子的，这就是 Kubernetes 集群的 期望状态。

与 Kubernetes 对象工作 —— 是否创建、修改，或者删除 —— 需要使用 Kubernetes API。当使用 `kubectl` 命令行接口时，比如，CLI 会使用必要的 Kubernetes API 调用，也可以在程序中直接使用 Kubernetes API。

## 三、对象的Spec和状态

每个 Kubernetes 对象包含两个嵌套的对象字段，它们负责管理对象的配置：对象 *spec* 和 对象 *status*。*spec* 必须提供，它描述了对象的 *期望状态*—— 希望对象所具有的特征。*status* 描述了对象的 *实际状态*，它是由 Kubernetes 系统提供和更新。在任何时刻，Kubernetes 控制平面一直处于活跃状态，管理着对象的实际状态以与我们所期望的状态相匹配。

例如，Kubernetes Deployment 对象能够表示运行在集群中的应用。当创建 Deployment 时，可能需要设置 Deployment 的 spec，以指定该应用需要有 3 个副本在运行。Kubernetes 系统读取 Deployment spec，启动我们所期望的该应用的 3 个实例 —— 更新状态以与 spec 相匹配。如果那些实例中有失败的（一种状态变更），Kubernetes 系统通过修正来响应 spec 和状态之间的不一致 —— 这种情况，启动一个新的实例来替换。

## 四、Pod的配置格式

当创建 Kubernetes 对象时，必须提供对象的 spec，用来描述该对象的期望状态，以及关于对象的一些基本信息（例如，名称）。当使用 Kubernetes API 创建对象时（或者直接创建，或者基于`kubectl`），API 请求必须在请求体中包含 JSON 格式的信息。更常用的是，需要在 .yaml 文件中为 kubectl 提供这些信息。 `kubectl` 在执行 API 请求时，将这些信息转换成 JSON 格式。查看已经部署好的pod的资源定义格式：

```
[root@k8s-master ~]# kubectl get pod myapp-848b5b879b-5f69p -o yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: 2018-08-24T07:40:57Z
  generateName: myapp-848b5b879b-
  labels:
    pod-template-hash: "4046164356"
    run: myapp
  name: myapp-848b5b879b-5f69p
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: myapp-848b5b879b
    uid: caf2ec54-a76f-11e8-84d2-000c2972dc1f
  resourceVersion: "90507"
  selfLink: /api/v1/namespaces/default/pods/myapp-848b5b879b-5f69p
  uid: 09bc0ba1-a771-11e8-84d2-000c2972dc1f
spec:
  containers:
  - image: ikubernetes/myapp:v1
    imagePullPolicy: IfNotPresent
    name: myapp
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-j5pf5
      readOnly: true
  dnsPolicy: ClusterFirst
  nodeName: k8s-node01
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: default-token-j5pf5
    secret:
      defaultMode: 420
      secretName: default-token-j5pf5
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: 2018-08-24T07:40:57Z
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: 2018-08-24T07:40:59Z
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: null
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: 2018-08-24T07:40:57Z
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: docker://a5e7004f45b1ec4a4297e50db6d0b5b11573e36ed8de814ea8b6cdacd13b8f9a
    image: ikubernetes/myapp:v1
    imageID: docker-pullable://ikubernetes/myapp@sha256:9c3dc30b5219788b2b8a4b065f548b922a34479577befb54b03330999d30d513
    lastState: {}
    name: myapp
    ready: true
    restartCount: 0
    state:
      running:
        startedAt: 2018-08-24T07:40:58Z
  hostIP: 192.168.56.12
  phase: Running
  podIP: 10.244.1.11
  qosClass: BestEffort
  startTime: 2018-08-24T07:40:57Z
```

创建资源的方法：

- apiserver在定义资源时，仅接收JSON格式的资源定义;
- yaml格式提供配置清单，apiservere可自动将其转为json格式，而后再提交；

大部分资源的配置清单格式都由5个一级字段组成：

```
   apiVersion: group/version　　指明api资源属于哪个群组和版本，同一个组可以有多个版本
        $ kubectl api-versions

    kind: 资源类别，标记创建的资源类型，k8s主要支持以下资源类别
        Pod,ReplicaSet,Deployment,StatefulSet,DaemonSet,Job,Cronjob
    
    metadata:元数据，主要提供以下字段
        name：同一类别下的name是唯一的
        namespace：对应的对象属于哪个名称空间
        labels：标签，每一个资源都可以有标签，标签是一种键值数据
        annotations：资源注解
        
        每个的资源引用方式(selflink)：
            /api/GROUP/VERSION/namespace/NAMESPACE/TYPE/NAME
    
    spec: 定义目标资源的期望状态（disired state），资源对象中最重要的字段
    
    status: 显示资源的当前状态（current state），本字段由kubernetes进行维护
```

K8s存在内嵌的格式说明，可以使用kubectl explain 进行查看，如查看Pod这个资源的定义：

```
[root@k8s-master ~]# kubectl explain pods
KIND:     Pod
VERSION:  v1

DESCRIPTION:
     Pod is a collection of containers that can run on a host. This resource is
     created by clients and scheduled onto hosts.

FIELDS:
   apiVersion    <string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#resources

   kind    <string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#types-kinds

   metadata    <Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#metadata

   spec    <Object>
     Specification of the desired behavior of the pod. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status

   status    <Object>
     Most recently observed status of the pod. This data may not be up to date.
     Populated by the system. Read-only. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status
```

从上面可以看到apiVersion，kind等定义的键值都是<string>，而metadata和spec看到是一个<Object>，当看到存在<Object>的提示，说明该字段可以存在多个二级字段，那么可以使用如下命令继续查看二级字段的定义方式：

```
[root@k8s-master ~]# kubectl explain pods.metadata
[root@k8s-master ~]# kubectl explain pods.spec
```

二级字段下，每一种字段都有对应的键值类型，常用类型大致如下：

**<[]string>：**表示是一个字串列表，也就是字串类型的数组

**<Object>：**表示是可以嵌套的字段

**<map[string]string>：**表示是一个由键值组成映射

**<[]Object>：**表示是一个对象列表

**<[]Object> -required-：**required表示该字段是一个必选的字段

## 五、使用配置清单创建自主式Pod资源 

```
[root@k8s-master ~]# mkdir mainfests
[root@k8s-master ~]# cd mainfrests
[root@k8s-master mainfrests]# vim pod-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
  namespace: default
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
  - name:myapp
    image: ikubernetes/myapp:v1
  - name: busybox
    image: busybox:latest
    command:
    - "/bin/sh"
    - "-c"
    - "sleep 3600"
[root@k8s-master mainfrests]# kubectl create -f pod-demo.yaml
[root@k8s-master mainfrests]# kubectl get pods
[root@k8s-master mainfrests]# kubectl describe pods pod-demo　　#获取pod详细信息
[root@k8s-master mainfrests]# kubectl logs pod-demo myapp
[root@k8s-master mainfrests]# kubectl exec -it pod-demo  -c myapp -- /bin/sh
```

## 六、Pod资源下spec的containers必需字段解析

```
[root@k8s-master ~]# kubectl explain pods.spec.containers

name    <string> -required-    #containers 的名字
image    <string>  #镜像地址
imagePullPolicy    <string>  #如果标签是latest  就是Always(总是下载镜像)  IfNotPresent（先看本地是否有此镜像，如果没有就下载） Never （就是使用本地镜像）

ports    <[]Object>  #是给对象列表  可以暴露多个端口  可以对每个端口的属性定义 例如：（名称（可后期调用）端口号  协议  暴露在的地址上） 暴露端口只是提供额外信息的，不能限制系统是否真的暴露

　　　- containerPort 容器端口

　　　  hostIP  主机地址（基本不会使用）

　　　  hostPort 节点端口

　　　  name 名称

　　　  protocol  （默认是TCP）

args  <[]string>   传递参数给command 相当于docker中的CMD

command    <[]string> 相当于docker中的ENTRYPOINT
```

如果Pod不提供`command`或`args`使用Container，则使用Docker镜像中的cmd或者ENTRYPOINT。

如果Pod提供`command`但不提供`args`，则仅使用提供 `command`的。将忽略Docker镜像中定义EntryPoint和Cmd。

如果Pod中仅提供`args`，则`args`将作为参数提供给Docker镜像中EntryPoint。

如果提供了`command`和`args`，则Docker镜像中的ENTRYPOINT和CMD都将不会生效，Pod中的`args`将作为参数给`command运行`。

## 七、标签及标签选择器 

**1、标签**

key=value

- key：只能使用 字母 数字 _ - . (只能以字母数字开头，不能超过63给字符)
- value： 可以为空 只能使用 字母 数字开头

```
[root@k8s-master mainfests]# kubectl get pods --show-labels　　#查看pod标签
NAME       READY     STATUS    RESTARTS   AGE       LABELS
pod-demo   2/2       Running   0          25s       app=myapp,tier=frontend

[root@k8s-master mainfests]# kubectl get pods -l app　　#过滤包含app的标签
NAME       READY     STATUS    RESTARTS   AGE
pod-demo   2/2       Running   0          1m
[root@k8s-master mainfests]# kubectl get pods -L app
NAME       READY     STATUS    RESTARTS   AGE       APP
pod-demo   2/2       Running   0          1m        myapp

[root@k8s-master mainfests]# kubectl label pods pod-demo release=canary　　#给pod-demo打上标签
pod/pod-demo labeled
[root@k8s-master mainfests]# kubectl get pods -l app --show-labels
NAME       READY     STATUS    RESTARTS   AGE       LABELS
pod-demo   2/2       Running   0          1m        app=myapp,release=canary,tier=frontend

[root@k8s-master mainfests]# kubectl label pods pod-demo release=stable --overwrite　　#修改标签
pod/pod-demo labeled
[root@k8s-master mainfests]# kubectl get pods -l release
NAME       READY     STATUS    RESTARTS   AGE
pod-demo   2/2       Running   0          2m
[root@k8s-master mainfests]# kubectl get pods -l release,app
NAME       READY     STATUS    RESTARTS   AGE
pod-demo   2/2       Running   0          2m
```

**2、标签选择器**

- 等值关系标签选择器：=， == ， ！= （kubectl get pods -l app=test,app=dev）
- 集合关系标签选择器: KEY in (v1,v2,v3)， KEY notin (v1,v2,v3)  !KEY （kubectl get pods -l "app in (test,dev)")

许多资源支持内嵌字段

- matchLabels: 直接给定建值
- matchExpressions: 基于给定的表达式来定义使用标签选择器，{key:"KEY",operator:"OPERATOR",values:[V1,V2,....]}
- 操作符: in notin：Values字段的值必须是非空列表 Exists NotExists: Values字段的值必须是空列表

**3、节点标签选择器**

```
[root@k8s-master mainfests]# kubectl explain pod.spec
   nodeName    <string>
     NodeName is a request to schedule this pod onto a specific node. If it is
     non-empty, the scheduler simply schedules this pod onto that node, assuming
     that it fits resource requirements.

   nodeSelector    <map[string]string>
     NodeSelector is a selector which must be true for the pod to fit on a node.
     Selector which must match a node's labels for the pod to be scheduled on
     that node. More info:
     https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
```

nodeSelector可以限定pod创建在哪个节点上，举个例子，给节点k8s-node01打上标签disktype=ssd，让pod-demo指定创建在k8s-node01上

（1）给k8s-node01节点打标签

```

[root@k8s-master mainfests]# kubectl label nodes k8s-node01 disktype=ssd
node/k8s-node01 labeled

[root@k8s-master mainfests]# kubectl get nodes --show-labels
NAME         STATUS    ROLES     AGE       VERSION   LABELS
k8s-master   Ready     master    10d       v1.11.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=k8s-master,node-role.kubernetes.io/master=
k8s-node01   Ready     <none>    10d       v1.11.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disktype=ssd,kubernetes.io/hostname=k8s-node01
k8s-node02   Ready     <none>    9d        v1.11.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=k8s-node02

（2）修改yaml文件，增加标签选择器
[root@k8s-master mainfests]# cat pod-demo.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
  namespace: default
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
  - name: busybox
    image: busybox:latest
    command:
    - "/bin/sh"
    - "-c"
    - "sleep 3600"
  nodeSeletor:
    disktype: ssd

（3）重新创建pod-demo，可以看到固定调度在k8s-node01节点上
[root@k8s-master mainfests]# kubectl delete -f pod-demo.yaml 
pod "pod-demo" deleted

[root@k8s-master mainfests]# kubectl create -f pod-demo.yaml 
pod/pod-demo created
[root@k8s-master mainfests]# kubectl get pods -o wide
NAME       READY     STATUS    RESTARTS   AGE       IP            NODE
pod-demo   2/2       Running   0          20s       10.244.1.13   k8s-node01
[root@k8s-master mainfests]# kubectl describe pod pod-demo
......
Events:
  Type    Reason     Age   From                 Message
  ----    ------     ----  ----                 -------
  Normal  Scheduled  42s   default-scheduler    Successfully assigned default/pod-demo to k8s-node01
......
```

**annotations:**

　　与label不同的地方在于，annotations不能用于挑选资源对象，仅用于为对象提供"元数据"，没有键值长度限制。

 

# Pod状态和生命周期管理

## 一、什么是Pod？

Pod是kubernetes中可以创建和部署的最小也是最简的单位。一个Pod代表着集群中运行的一个进程。

Pod中封装着应用的容器（有的情况下是好几个容器），存储、独立的网络IP，管理容器如何运行的策略选项。Pod代表着部署的一个单位：kubernetes中应用的一个实例，可能由一个或者多个容器组合在一起共享资源。

在Kubrenetes集群中Pod有如下两种使用方式：

- 一个Pod中运行一个容器。“每个Pod中一个容器”的模式是最常见的用法；在这种使用方式中，你可以把Pod想象成是单个容器的封装，kuberentes管理的是Pod而不是直接管理容器。
- 在一个Pod中同时运行多个容器。一个Pod中也可以同时封装几个需要紧密耦合互相协作的容器，它们之间共享资源。这些在同一个Pod中的容器可以互相协作成为一个service单位——一个容器共享文件，另一个“sidecar”容器来更新这些文件。Pod将这些容器的存储资源作为一个实体来管理。

Pod中共享的环境包括Linux的namespace，cgroup和其他可能的隔绝环境，这一点跟Docker容器一致。在Pod的环境中，每个容器中可能还有更小的子隔离环境。

Pod中的容器共享IP地址和端口号，它们之间可以通过`localhost`互相发现。它们之间可以通过进程间通信，需要明白的是同一个Pod下的容器是通过lo网卡进行通信。例如[SystemV](https://en.wikipedia.org/wiki/UNIX_System_V)信号或者POSIX共享内存。不同Pod之间的容器具有不同的IP地址，不能直接通过IPC通信。

Pod中的容器也有访问共享volume的权限，这些volume会被定义成pod的一部分并挂载到应用容器的文件系统中。

就像每个应用容器，pod被认为是临时实体。在Pod的生命周期中，pod被创建后，被分配一个唯一的ID（UID），调度到节点上，并一致维持期望的状态直到被终结（根据重启策略）或者被删除。如果node死掉了，分配到了这个node上的pod，在经过一个超时时间后会被重新调度到其他node节点上。一个给定的pod（如UID定义的）不会被“重新调度”到新的节点上，而是被一个同样的pod取代，如果期望的话甚至可以是相同的名字，但是会有一个新的UID（查看replication controller获取详情）。

## 二、Pod中如何管理多个容器？

Pod中可以同时运行多个进程（作为容器运行）协同工作。同一个Pod中的容器会自动的分配到同一个 node 上。同一个Pod中的容器共享资源、网络环境和依赖，它们总是被同时调度。

注意在一个Pod中同时运行多个容器是一种比较高级的用法。只有当你的容器需要紧密配合协作的时候才考虑用这种模式。例如，你有一个容器作为web服务器运行，需要用到共享的volume，有另一个“sidecar”容器来从远端获取资源更新这些文件，如下图所示：

![img](https://images2018.cnblogs.com/blog/1349539/201809/1349539-20180901112128577-178843.png)

Pod中可以共享两种资源：网络和存储。

-  **网络：**

　　每个Pod都会被分配一个唯一的IP地址。Pod中的所有容器共享网络空间，包括IP地址和端口。Pod内部的容器可以使用`localhost`互相通信。Pod中的容器与外界通信时，必须分配共享网络资源（例如使用宿主机的端口映射）。

- **存储：**

　　可以Pod指定多个共享的Volume。Pod中的所有容器都可以访问共享的volume。Volume也可以用来持久化Pod中的存储资源，以防容器重启后文件丢失。

##  三、使用Pod

通常把Pod分为两类：

- **自主式Pod ：**这种Pod本身是不能自我修复的，当Pod被创建后（不论是由你直接创建还是被其他Controller），都会被Kuberentes调度到集群的Node上。直到Pod的进程终止、被删掉、因为缺少资源而被驱逐、或者Node故障之前这个Pod都会一直保持在那个Node上。Pod不会自愈。如果Pod运行的Node故障，或者是调度器本身故障，这个Pod就会被删除。同样的，如果Pod所在Node缺少资源或者Pod处于维护状态，Pod也会被驱逐。
- **控制器管理的Pod：**Kubernetes使用更高级的称为Controller的抽象层，来管理Pod实例。Controller可以创建和管理多个Pod，提供副本管理、滚动升级和集群级别的自愈能力。例如，如果一个Node故障，Controller就能自动将该节点上的Pod调度到其他健康的Node上。虽然可以直接使用Pod，但是在Kubernetes中通常是使用Controller来管理Pod的。如下图：

**每个Pod都有一个特殊的被称为“根容器”的Pause 容器。 Pause容器对应的镜像属于Kubernetes平台的一部分，除了Pause容器，每个Pod还包含一个或者多个紧密相关的用户业务容器。**

<img src="https://images2018.cnblogs.com/blog/1349539/201809/1349539-20180901112712875-923099036.png" align="center" style="zoom:50%;" />

**Kubernetes设计这样的Pod概念和特殊组成结构有什么用意？**

原因一：在一组容器作为一个单元的情况下，难以对整体的容器简单地进行判断及有效地进行行动。比如，一个容器死亡了，此时是算整体挂了么？那么引入与业务无关的Pause容器作为Pod的根容器，以它的状态代表着整个容器组的状态，这样就可以解决该问题。

原因二：Pod里的多个业务容器共享Pause容器的IP，共享Pause容器挂载的Volume，这样简化了业务容器之间的通信问题，也解决了容器之间的文件共享问题。

## 四、Pod的持久性和终止

### **（1）Pod的持久性**

Pod在设计支持就不是作为持久化实体的。在调度失败、节点故障、缺少资源或者节点维护的状态下都会死掉会**被驱逐**。

通常，用户不需要手动直接创建Pod，而是应该使用controller（例如[Deployments](https://jimmysong.io/kubernetes-handbook/concepts/deployment.html)），即使是在创建单个Pod的情况下。Controller可以提供集群级别的自愈功能、复制和升级管理。

### **（2）Pod的终止**

因为Pod作为在集群的节点上运行的进程，所以在不再需要的时候能够优雅的终止掉是十分必要的（比起使用发送KILL信号这种暴力的方式）。用户需要能够放松删除请求，并且知道它们何时会被终止，是否被正确的删除。用户想终止程序时发送删除pod的请求，在pod可以被强制删除前会有一个宽限期，会发送一个TERM请求到每个容器的主进程。一旦超时，将向主进程发送KILL信号并从API server中删除。如果kubelet或者container manager在等待进程终止的过程中重启，在重启后仍然会重试完整的宽限期。

示例流程如下：

1. 用户发送删除pod的命令，默认宽限期是30秒；
2. 在Pod超过该宽限期后API server就会更新Pod的状态为“dead”；
3. 在客户端命令行上显示的Pod状态为“terminating”；
4. 跟第三步同时，当kubelet发现pod被标记为“terminating”状态时，开始停止pod进程：
   1. 如果在pod中定义了preStop hook，在停止pod前会被调用。如果在宽限期过后，preStop hook依然在运行，第二步会再增加2秒的宽限期；
   2. 向Pod中的进程发送TERM信号；
5. 跟第三步同时，该Pod将从该service的端点列表中删除，不再是replication controller的一部分。关闭慢的pod将继续处理load balancer转发的流量；
6. 过了宽限期后，将向Pod中依然运行的进程发送SIGKILL信号而杀掉进程。
7. Kublete会在API server中完成Pod删除，通过将优雅周期设置为0（立即删除）。Pod在API中消失，并且在客户端也不可见。

删除宽限期默认是30秒。 **`kubectl delete`**命令支持 **`—grace-period=<seconds>`** 选项，允许用户设置自己的宽限期。如果设置为0将强制删除pod。在kubectl>=1.5版本的命令中，你必须同时使用 `--force` 和 `--grace-period=0` 来强制删除pod。

Pod的强制删除是通过在集群和etcd中将其定义为删除状态。当执行强制删除命令时，API server不会等待该pod所运行在节点上的kubelet确认，就会立即将该pod从API server中移除，这时就可以创建跟原pod同名的pod了。这时，在节点上的pod会被立即设置为terminating状态，不过在被强制删除之前依然有一小段优雅删除周期。 

##  五、Pause容器

 Pause容器，又叫Infra容器。我们检查node节点的时候会发现每个node上都运行了很多的pause容器，例如如下。

```
[root@k8s-node01 ~]# docker ps |grep pause
0cbf85d4af9e    k8s.gcr.io/pause:3.1   "/pause"     7 days ago  Up 7 days   k8s_POD_myapp-848b5b879b-ksgnv_default_0af41a40-a771-11e8-84d2-000c2972dc1f_0
d6e4d77960a7    k8s.gcr.io/pause:3.1   "/pause"     7 days ago  Up 7 days   k8s_POD_myapp-848b5b879b-5f69p_default_09bc0ba1-a771-11e8-84d2-000c2972dc1f_0
5f7777c55d2a    k8s.gcr.io/pause:3.1   "/pause"     7 days ago  Up 7 days   k8s_POD_kube-flannel-ds-pgpr7_kube-system_23dc27e3-a5af-11e8-84d2-000c2972dc1f_1
8e56ef2564c2    k8s.gcr.io/pause:3.1   "/pause"     7 days ago  Up 7 days   k8s_POD_client2_default_17dad486-a769-11e8-84d2-000c2972dc1f_1
7815c0d69e99    k8s.gcr.io/pause:3.1   "/pause"     7 days ago  Up 7 days   k8s_POD_nginx-deploy-5b595999-872c7_default_7e9df9f3-a6b6-11e8-84d2-000c2972dc1f_2
b4e806fa7083    k8s.gcr.io/pause:3.1   "/pause"     7 days ago  Up 7 days   k8s_POD_kube-proxy-vxckf_kube-system_23dc0141-a5af-11e8-84d2-000c2972dc1f_2
```

kubernetes中的pause容器主要为每个业务容器提供以下功能：

- **在pod中担任Linux命名空间共享的基础；**
- **启用pid命名空间，开启init进程。**

如图：

<img src="https://images2018.cnblogs.com/blog/1349539/201809/1349539-20180901114609166-101965180.png" align="center" style="zoom:50%;" />

 

```
[root@k8s-node01 ~]# docker run -d --name pause -p 8880:80 k8s.gcr.io/pause:3.1
d3057ceb54bc6565d28ded2c33ad2042010be73d76117775c130984c3718d609
[root@k8s-node01 ~]# cat <<EOF >> nginx.conf
> error_log stderr;
> events { worker_connections  1024; }
> http {
>     access_log /dev/stdout combined;
>     server {
>         listen 80 default_server;
>         server_name example.com www.example.com;
>         location / {
>             proxy_pass http://127.0.0.1:2368;
>         }
>     }
> }
> EOF
[root@k8s-node01 ~]# docker run -d --name nginx -v `pwd`/nginx.conf:/etc/nginx/nginx.conf --net=container:pause --ipc=container:pause --pid=container:pause nginx
d04f848b7386109085ee350ebb81103e4efc7df8e48da18404efb9712f926082
[root@k8s-node01 ~]#  docker run -d --name ghost --net=container:pause --ipc=container:pause --pid=container:pause ghost
332c86a722f71680b76b3072e85228a8d8e9608456c653edd214f06c2a77f112
```

现在访问[http://192.168.56.12:8880/](http://localhost:8880/)就可以看到ghost博客的界面了。

**解析**

pause容器将内部的80端口映射到宿主机的8880端口，pause容器在宿主机上设置好了网络namespace后，nginx容器加入到该网络namespace中，我们看到nginx容器启动的时候指定了`--net=container:pause`，ghost容器同样加入到了该网络namespace中，这样三个容器就共享了网络，互相之间就可以使用`localhost`直接通信，`--ipc=contianer:pause --pid=container:pause`就是三个容器处于同一个namespace中，init进程为`pause`，这时我们进入到ghost容器中查看进程情况。

```
[root@k8s-node01 ~]# docker exec -it ghost /bin/bash
root@d3057ceb54bc:/var/lib/ghost# ps axu 
USER        PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root          1  0.0  0.0   1012     4 ?        Ss   03:48   0:00 /pause
root          6  0.0  0.0  32472   780 ?        Ss   03:53   0:00 nginx: master process nginx -g daemon off;
systemd+     11  0.0  0.1  32932  1700 ?        S    03:53   0:00 nginx: worker process
node         12  0.4  7.5 1259816 74868 ?       Ssl  04:00   0:07 node current/index.js
root         77  0.6  0.1  20240  1896 pts/0    Ss   04:29   0:00 /bin/bash
root         82  0.0  0.1  17496  1156 pts/0    R+   04:29   0:00 ps axu
```

在ghost容器中同时可以看到pause和nginx容器的进程，并且pause容器的PID是1。而在kubernetes中容器的PID=1的进程即为容器本身的业务进程。

## 六、init容器 

[Pod](https://kubernetes.io/docs/concepts/abstractions/pod/) 能够具有多个容器，应用运行在容器里面，但是它也可能有一个或多个先于应用容器启动的 Init 容器。

Init 容器与普通的容器非常像，除了如下两点：

- Init 容器总是运行到成功完成为止。
- 每个 Init 容器都必须在下一个 Init 容器启动之前成功完成。

如果 Pod 的 Init 容器失败，Kubernetes 会不断地重启该 Pod，直到 Init 容器成功为止。然而，如果 Pod 对应的 `restartPolicy` 为 Never，它不会重新启动。

## 七、Pod的生命周期

### **（1）Pod phase（Pod的相位）**

Pod 的 `status` 在信息保存在 [PodStatus](https://kubernetes.io/docs/resources-reference/v1.7/#podstatus-v1-core) 中定义，其中有一个 `phase` 字段。

Pod 的相位（phase）是 Pod 在其生命周期中的简单宏观概述。该阶段并不是对容器或 Pod 的综合汇总，也不是为了做为综合状态机。

Pod 相位的数量和含义是严格指定的。除了本文档中列举的状态外，不应该再假定 Pod 有其他的 `phase`值。

下面是 `phase` 可能的值：

- **挂起（Pending）**：API Server创建了Pod资源对象并已经存入了etcd中，但是它并未被调度完成，或者仍然处于从仓库下载镜像的过程中。
- **运行中（Running）**：Pod已经被调度到某节点之上，并且所有容器都已经被kubelet创建完成。
- **成功（Succeeded）**：Pod 中的所有容器都被成功终止，并且不会再重启。
- **失败（Failed）**：Pod 中的所有容器都已终止了，并且至少有一个容器是因为失败终止。也就是说，容器以非0状态退出或者被系统终止。
- **未知（Unknown）**：因为某些原因无法取得 Pod 的状态，通常是因为与 Pod 所在主机通信失败。

下图是Pod的生命周期示意图，从图中可以看到Pod状态的变化。

![img](https://images2018.cnblogs.com/blog/1349539/201809/1349539-20180901151105739-984439707.png)

### **（2）Pod的创建过程**

Pod是Kubernetes的基础单元，了解其创建的过程，更有助于理解系统的运作。

* 用户通过kubectl或其他API客户端提交Pod Spec给API Server。

* API Server尝试将Pod对象的相关信息存储到etcd中，等待写入操作完成，API Server返回确认信息到客户端。

* API Server开始反映etcd中的状态变化。

* 所有的Kubernetes组件通过"watch"机制跟踪检查API Server上的相关信息变动。

* kube-scheduler（调度器）通过其"watcher"检测到API Server创建了新的Pod对象但是没有绑定到任何工作节点。

* kube-scheduler为Pod对象挑选一个工作节点并将结果信息更新到API Server。

* 调度结果新消息由API Server更新到etcd，并且API Server也开始反馈该Pod对象的调度结果。

* Pod被调度到目标工作节点上的kubelet尝试在当前节点上调用docker engine进行启动容器，并将容器的状态结果返回到API Server。

* API Server将Pod信息存储到etcd系统中。

* 在etcd确认写入操作完成，API Server将确认信息发送到相关的kubelet。

### **（3）Pod的状态**

Pod 有一个 PodStatus 对象，其中包含一个 [PodCondition](https://kubernetes.io/docs/resources-reference/v1.7/#podcondition-v1-core) 数组。 PodCondition 数组的每个元素都有一个 `type` 字段和一个 `status` 字段。`type` 字段是字符串，可能的值有 PodScheduled、Ready、Initialized 和 Unschedulable。`status` 字段是一个字符串，可能的值有 True、False 和 Unknown。

### **（4）Pod存活性探测**

在pod生命周期中可以做的一些事情。主容器启动前可以完成初始化容器，初始化容器可以有多个，他们是串行执行的，执行完成后就推出了，在主程序刚刚启动的时候可以指定一个post start 主程序启动开始后执行一些操作，在主程序结束前可以指定一个 pre stop 表示主程序结束前执行的一些操作。在程序启动后可以做两类检测 **liveness probe（存活性探测） 和 readness probe（就绪性探测）**。如下图：

 ![img](https://images2018.cnblogs.com/blog/1349539/201809/1349539-20180901151402364-916874449.png)

探针是由 kubelet 对容器执行的定期诊断。要执行诊断，kubelet 调用由容器实现的Handler。其存活性探测的方法有以下三种：

- **ExecAction：**在容器内执行指定命令。如果命令退出时返回码为 0 则认为诊断成功。
- **TCPSocketAction：**对指定端口上的容器的 IP 地址进行 TCP 检查。如果端口打开，则诊断被认为是成功的。
- **HTTPGetAction：**对指定的端口和路径上的容器的 IP 地址执行 HTTP Get 请求。如果响应的状态码大于等于200 且小于 400，则诊断被认为是成功的。

**设置exec探针举例：**

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness-exec
  name: liveness-exec
spec:
  containers:
  - name: liveness-exec-demo
    image: busybox
    args: ["/bin/sh","-c","touch /tmp/healthy;sleep 60;rm -rf /tmp/healthy;"sleep 600]
    livenessProbe:
      exec:
        command: ["test","-e","/tmp/healthy"]
```

上面的资源清单中定义了一个Pod 对象， 基于 busybox 镜像 启动 一个 运行“ touch/ tmp/ healthy； sleep 60； rm- rf/ tmp/ healthy； sleep 600” 命令 的 容器， 此 命令 在 容器 启动 时 创建/ tmp/ healthy 文件， 并于 60 秒 之后 将其 删除。 存活 性 探针 运行“ test -e/ tmp/ healthy” 命令 检查/ tmp/healthy 文件 的 存在 性， 若 文件 存在 则 返回 状态 码 0， 表示 成功 通过 测试。

**设置HTTP探针举例：**

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness-http
  name: liveness-http
spec:
  containers:
  - name: liveness-http-demo
    image: nginx:1.12-alpine
    ports:
    - name: http
      containerPort: 80
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh","-c","echo healthy > /usr/share/nginx/html/healthy"]
    livenessProbe:
      httpGet:
        path: /healthy
        port: http
        scheme: HTTP
```

上面 清单 文件 中 定义 的 httpGet 测试 中， 请求 的 资源 路径 为“/ healthy”， 地址 默认 为 Pod IP， 端口 使用 了 容器 中 定义 的 端口 名称 HTTP， 这也 是 明确 为 容器 指明 要 暴露 的 端口 的 用途 之一。

**设置TCP探针举例：**

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness-tcp
  name: liveness-tcp
spec:
  containers:
  - name: liveness-tcp-demo
    image: nginx:1.12-alpine
    ports:
    - name: http
      containerPort: 80
    livenessProbe:
      tcpSocket:
        port: http
```

上面的资源清单文件，向Pod IP的80/tcp端口发起连接请求，并根据连接建立的状态判断Pod存活状态。

每次探测都将获得以下三种结果之一：

- 成功：容器通过了诊断。
- 失败：容器未通过诊断。
- 未知：诊断失败，因此不会采取任何行动。

Kubelet 可以选择是否执行在容器上运行的两种探针执行和做出反应：

- **`livenessProbe`：**指示容器是否正在运行。如果存活探测失败，则 kubelet 会杀死容器，并且容器将受到其 [重启策略](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) 的影响。如果容器不提供存活探针，则默认状态为 `Success`。
- **`readinessProbe`：**指示容器是否准备好服务请求。如果就绪探测失败，端点控制器将从与 Pod 匹配的所有 Service 的端点中删除该 Pod 的 IP 地址。初始延迟之前的就绪状态默认为 `Failure`。如果容器不提供就绪探针，则默认状态为 `Success`。

### **（5）livenessProbe和readinessProbe使用场景**

如果容器中的进程能够在遇到问题或不健康的情况下自行崩溃，则不一定需要存活探针; kubelet 将根据 Pod 的`restartPolicy` 自动执行正确的操作。

如果希望容器在探测失败时被杀死并重新启动，那么请指定一个存活探针，并指定`restartPolicy` 为 Always 或 OnFailure。

如果要仅在探测成功时才开始向 Pod 发送流量，请指定就绪探针。在这种情况下，就绪探针可能与存活探针相同，但是 spec 中的就绪探针的存在意味着 Pod 将在没有接收到任何流量的情况下启动，并且只有在探针探测成功后才开始接收流量。

如果您希望容器能够自行维护，您可以指定一个就绪探针，该探针检查与存活探针不同的端点。

请注意，如果您只想在 Pod 被删除时能够排除请求，则不一定需要使用就绪探针；在删除 Pod 时，Pod 会自动将自身置于未完成状态，无论就绪探针是否存在。当等待 Pod 中的容器停止时，Pod 仍处于未完成状态。

### **（6）Pod的重启策略**

PodSpec 中有一个 `restartPolicy` 字段，可能的值为 **Always、OnFailure 和 Never**。默认为 Always。 `restartPolicy` 适用于 Pod 中的所有容器。`restartPolicy` 仅指通过同一节点上的 kubelet 重新启动容器。失败的容器由 kubelet 以五分钟为上限的指数退避延迟（10秒，20秒，40秒...）重新启动，并在成功执行十分钟后重置。pod一旦绑定到一个节点，Pod 将永远不会重新绑定到另一个节点。

### **（7）Pod的生命**

一般来说，Pod 不会消失，直到人为销毁他们。这可能是一个人或控制器。这个规则的唯一例外是成功或失败的 `phase` 超过一段时间（由 master 确定）的Pod将过期并被自动销毁。

有三种可用的控制器：

- 使用 [Job](https://kubernetes.io/docs/concepts/jobs/run-to-completion-finite-workloads/) 运行预期会终止的 Pod，例如批量计算。Job 仅适用于重启策略为 `OnFailure` 或 `Never` 的 Pod。

- 对预期不会终止的 Pod 使用 [ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)、[ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) 和 [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) ，例如 Web 服务器。 ReplicationController 仅适用于具有 `restartPolicy` 为 Always 的 Pod。
- 提供特定于机器的系统服务，使用 [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) 为每台机器运行一个 Pod 。

所有这三种类型的控制器都包含一个 PodTemplate。建议创建适当的控制器，让它们来创建 Pod，而不是直接自己创建 Pod。这是因为单独的 Pod 在机器故障的情况下没有办法自动复原，而控制器却可以。

如果节点死亡或与集群的其余部分断开连接，则 Kubernetes 将应用一个策略将丢失节点上的所有 Pod 的 `phase` 设置为 Failed。

###  **（8）livenessProbe解析**

```
[root@k8s-master ~]# kubectl explain pod.spec.containers.livenessProbe

KIND:     Pod
VERSION:  v1

RESOURCE: livenessProbe <Object>

exec  command 的方式探测 例如 ps 一个进程

failureThreshold 探测几次失败 才算失败 默认是连续三次

periodSeconds 每次的多长时间探测一次  默认10s

timeoutSeconds 探测超市的秒数 默认1s

initialDelaySeconds  初始化延迟探测，第一次探测的时候，因为主程序未必启动完成

tcpSocket 检测端口的探测

httpGet http请求探测
```

举个例子：定义一个liveness的pod资源类型，基础镜像为busybox，在busybox这个容器启动后会执行创建/tmp/test的文件啊，并删除，然后等待3600秒。随后定义了存活性探测，方式是以exec的方式执行命令判断/tmp/test是否存在，存在即表示存活，不存在则表示容器已经挂了。

```
[root@k8s-master ~]# vim liveness.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec-pod
  namespace: default
  labels:
    name: myapp
spec:
  containers:
  - name: livess-exec
    image: busybox:latest
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh","-c","touch /tmp/test; sleep 30; rm -f /tmp/test; sleep 3600"]
    livenessProbe:
      exec:
        command: ["test","-e","/tmp/test"]
      initialDelaySeconds: 1
      periodSeconds: 3
[root@k8s-master ~]# kubectl apply -f lineness.yaml 
```

###  （9）资源需求和资源限制

 在Docker的范畴内，我们知道可以对运行的容器进行请求或消耗的资源进行限制。而在Kubernetes中，也有同样的机制，容器或Pod可以进行申请和消耗的资源就是CPU和内存。CPU属于可压缩型资源，即资源的额度可以按照需求进行收缩。而内存属于不可压缩型资源，对内存的收缩可能会导致无法预知的问题。

资源的隔离目前是属于容器级别，CPU和内存资源的配置需要Pod中的容器spec字段下进行定义。其具体字段，可以使用"requests"进行定义请求的确保资源可用量。也就是说容器的运行可能用不到这样的资源量，但是必须确保有这么多的资源供给。而"limits"是用于限制资源可用的最大值，属于硬限制。

在Kubernetes中，1个单位的CPU相当于虚拟机的1颗虚拟CPU（vCPU）或者是物理机上一个超线程的CPU，它支持分数计量方式，一个核心（1core）相当于1000个微核心（millicores），因此500m相当于是0.5个核心，即二分之一个核心。内存的计量方式也是一样的，默认的单位是字节，也可以使用E、P、T、G、M和K作为单位后缀，或者是Ei、Pi、Ti、Gi、Mi、Ki等形式单位后缀。 

**资源需求举例：**

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "128Mi"
        cpu: "200m"
```

上面的配置清单中，nginx请求的CPU资源大小为200m，这意味着一个CPU核心足以满足nginx以最快的方式运行，其中对内存的期望可用大小为128Mi，实际运行时不一定会用到这么多的资源。考虑到内存的资源类型，在超出指定大小运行时存在会被OOM killer杀死的可能性，于是该请求值属于理想中使用的内存上限。

**资源限制举例：**

容器的资源需求只是能够确保容器运行时所需要的最少资源量，但是并不会限制其可用的资源上限。当应用程序存在Bug时，也有可能会导致系统资源被长期占用的情况，这就需要另外一个limits属性对容器进行定义资源使用的最大可用量。CPU是属于可压缩资源，可以进行自由地调节。而内存属于硬限制性资源，当进程申请分配超过limit属性定义的内存大小时，该Pod将会被OOM killer杀死。如下：

```
[root@k8s-master ~]# vim memleak-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: memleak-pod
  labels:
    app: memleak
spec:
  containers:
  - name: simmemleak
    image: saadali/simmemleak
    resources:
      requests:
        memory: "64Mi"
        cpu: "1"
      limits:
        memory: "64Mi"
        cpu: "1"

[root@k8s-master ~]# kubectl apply -f memleak-pod.yaml 
pod/memleak-pod created
[root@k8s-master ~]# kubectl get pods -l app=memleak
NAME          READY     STATUS      RESTARTS   AGE
memleak-pod   0/1       OOMKilled   2          12s
[root@k8s-master ~]# kubectl get pods -l app=memleak
NAME          READY     STATUS             RESTARTS   AGE
memleak-pod   0/1       CrashLoopBackOff   2          28s
```

Pod资源默认的重启策略为Always，在memleak因为内存限制而终止会立即重启，此时该Pod会被OOM killer杀死，在多次重复因为内存资源耗尽重启会触发Kunernetes系统的重启延迟，每次重启的时间会不断拉长，后面看到的Pod的状态通常为"CrashLoopBackOff"。

 这里还需要明确的是，在一个Kubernetes集群上，运行的Pod众多，那么当节点都无法满足多个Pod对象的资源使用时，是按照什么样的顺序去终止这些Pod对象呢？？

Kubernetes是无法自行去判断的，需要借助于Pod对象的优先级进行判定终止Pod的优先问题。根据Pod对象的requests和limits属性，Kubernetes将Pod对象分为三个服务质量类别：

- Guaranteed：每个容器都为CPU和内存资源设置了相同的requests和limits属性的Pod都会自动归属于该类别，属于最高优先级。
- Burstable：至少有一个容器设置了CPU或内存资源的requests属性，单不满足Guaranteed类别要求的资源归于该类别，属于中等优先级。
- BestEffort：未对任何容器设置requests属性和limits属性的Pod资源，自动归于该类别，属于最低级别。



# Pod控制器

## 一、Pod控制器及其功用

Pod控制器是用于实现管理pod的中间层，确保pod资源符合预期的状态，pod的资源出现故障时，会尝试 进行重启，当根据重启策略无效，则会重新新建pod的资源。

**pod控制器有多种类型：**

* **ReplicaSet:** 代用户创建指定数量的pod副本数量，确保pod副本数量符合预期状态，并且支持滚动式自动扩容和缩容功能。
  ReplicaSet主要三个组件组成：
  　　（1）用户期望的pod副本数量 
  　　（2）标签选择器，判断哪个pod归自己管理 
  　　（3）当现存的pod数量不足，会根据pod资源模板进行新建
  帮助用户管理无状态的pod资源，精确反应用户定义的目标数量，但是RelicaSet不是直接使用的控制器，而是使用Deployment。

* **Deployment：**工作在ReplicaSet之上，用于管理无状态应用，目前来说最好的控制器。支持滚动更新和回滚功能，还提供声明式配置。

* **DaemonSet：**用于确保集群中的每一个节点只运行特定的pod副本，通常用于实现系统级后台任务。比如ELK服务
  特性：服务是无状态的
  服务必须是守护进程

* **Job：**只要完成就立即退出，不需要重启或重建。

* **Cronjob：**周期性任务控制，不需要持续后台运行，

* **StatefulSet：**管理有状态应用

## 二、ReplicaSet控制器

ReplicationController用来确保容器应用的副本数始终保持在用户定义的副本数，即如果有容器异常退出，会自动创建新的Pod来替代；而如果异常多出来的容器也会自动回收。

在新版本的Kubernetes中建议使用ReplicaSet来取代ReplicationController。ReplicaSet跟ReplicationController没有本质的不同，只是名字不一样，并且ReplicaSet支持集合式的selector。

虽然ReplicaSet可以独立使用，但一般还是建议使用 Deployment 来自动管理ReplicaSet，这样就无需担心跟其他机制的不兼容问题（比如ReplicaSet不支持rolling-update但Deployment支持）。

**ReplicaSet示例：**

```
（1）命令行查看ReplicaSet清单定义规则
[root@k8s-master ~]# kubectl explain rs
[root@k8s-master ~]# kubectl explain rs.spec
[root@k8s-master ~]# kubectl explain rs.spec.template

（2）新建ReplicaSet示例
[root@k8s-master ~]# vim rs-demo.yaml
apiVersion: apps/v1　　#api版本定义
kind: ReplicaSet　　#定义资源类型为ReplicaSet
metadata:　　#元数据定义
    name: myapp
    namespace: default
spec:　　#ReplicaSet的规格定义
    replicas: 2　　#定义副本数量为2个
    selector:　　　　#标签选择器，定义匹配pod的标签
        matchLabels:
            app: myapp
            release: canary
    template:　　#pod的模板定义
        metadata:　　#pod的元数据定义
            name: myapp-pod　　　#自定义pod的名称　
            labels: 　　#定义pod的标签，需要和上面定义的标签一致，也可以多出其他标签
                app: myapp
                release: canary
                environment: qa
        spec:　　#pod的规格定义
            containers:　　#容器定义
            - name: myapp-container　　#容器名称
              image: ikubernetes/myapp:v1　　#容器镜像
              ports:　　#暴露端口
              - name: http
                containerPort: 80

（3）创建ReplicaSet定义的pod                
[root@k8s-master ~]# kubectl create -f rs-demo.yaml
[root@k8s-master ~]# kubectl get pods　　#获取pod信息
[root@k8s-master ~]# kubectl describe pods myapp-***　　#查看pod详细信息

（4）修改pod的副本数量
[root@k8s-master ~]# kubectl edit rs myapp
replicas: 5
[root@k8s-master ~]# kubectl get rs -o wide

（5）修改pod的镜像版本
[root@k8s-master ~]# kubectl edit rs myapp
image: ikubernetes/myapp:v2　　
[root@k8s-master ~]# kubectl delete pods myapp-*** 　　#修改了pod镜像版本，pod需要重建才能达到最新版本
[root@k8s-master ~]# kubectl create -f rs-demo.yaml
```

##  三、Deployment控制器

Deployment为Pod和Replica Set（下一代Replication Controller）提供声明式更新。

只需要在 Deployment 中描述想要的目标状态是什么，Deployment controller 就会帮您将 Pod 和ReplicaSet 的实际状态改变到您的目标状态。也可以定义一个全新的 Deployment 来创建 ReplicaSet 或者删除已有的 Deployment 并创建一个新的来替换。

<img src="https://images2018.cnblogs.com/blog/1349539/201809/1349539-20180903143316602-2134091802.png" alt="img" style="zoom:67%;" />

典型的用例如下：

- （1）使用Deployment来创建ReplicaSet。ReplicaSet在后台创建pod。检查启动状态，看它是成功还是失败。
- （2）然后，通过更新Deployment的PodTemplateSpec字段来声明Pod的新状态。这会创建一个新的ReplicaSet，Deployment会按照控制的速率将pod从旧的ReplicaSet移动到新的ReplicaSet中。
- （3）如果当前状态不稳定，回滚到之前的Deployment revision。每次回滚都会更新Deployment的revision。
- （4）扩容Deployment以满足更高的负载。
- （5）暂停Deployment来应用PodTemplateSpec的多个修复，然后恢复上线。
- （6）根据Deployment 的状态判断上线是否hang住了。
- （7）清除旧的不必要的 ReplicaSet。

### **1、解析Deployment Spec**

首先看一个官方的nginx-deployment.yaml的例子： 

```
apiVersion: v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
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
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

 在所有的 Kubernetes 配置中，Deployment 也需要`apiVersion`，`kind`和`metadata`这些配置项。如下：

![img](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)

```
[root@k8s-master ~]# kubectl explain deployment
KIND:     Deployment
VERSION:  extensions/v1beta1

DESCRIPTION:
     DEPRECATED - This group version of Deployment is deprecated by
     apps/v1beta2/Deployment. See the release notes for more information.
     Deployment enables declarative updates for Pods and ReplicaSets.

FIELDS:
   apiVersion    <string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#resources

   kind    <string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#types-kinds

   metadata    <Object>
     Standard object metadata.

   spec    <Object>
     Specification of the desired behavior of the Deployment.

   status    <Object>
     Most recently observed status of the Deployment.
```

使用kubectl explain deployment.spec查看具体Deployment spec的配置选项，解析如下：

![img](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)

```
[root@k8s-master ~]# kubectl explain deployment.spec
KIND:     Deployment
VERSION:  extensions/v1beta1

RESOURCE: spec <Object>

DESCRIPTION:
     Specification of the desired behavior of the Deployment.

     DeploymentSpec is the specification of the desired behavior of the
     Deployment.

FIELDS:
   minReadySeconds    <integer>
     Minimum number of seconds for which a newly created pod should be ready
     without any of its container crashing, for it to be considered available.
     Defaults to 0 (pod will be considered available as soon as it is ready)

   paused    <boolean>
     Indicates that the deployment is paused and will not be processed by the
     deployment controller.

   progressDeadlineSeconds    <integer>
     The maximum time in seconds for a deployment to make progress before it is
     considered to be failed. The deployment controller will continue to process
     failed deployments and a condition with a ProgressDeadlineExceeded reason
     will be surfaced in the deployment status. Note that progress will not be
     estimated during the time a deployment is paused. This is not set by
     default.

   replicas    <integer>
     Number of desired pods. This is a pointer to distinguish between explicit
     zero and not specified. Defaults to 1.

   revisionHistoryLimit    <integer>
     The number of old ReplicaSets to retain to allow rollback. This is a
     pointer to distinguish between explicit zero and not specified.

   rollbackTo    <Object>
     DEPRECATED. The config this deployment is rolling back to. Will be cleared
     after rollback is done.

   selector    <Object>
     Label selector for pods. Existing ReplicaSets whose pods are selected by
     this will be the ones affected by this deployment.

   strategy    <Object>
     The deployment strategy to use to replace existing pods with new ones.

   template    <Object> -required-
     Template describes the pods that will be created.
```

**Replicas（副本数量）：**
　　.spec.replicas 是可以选字段，指定期望的pod数量，默认是1。

**Selector（选择器）：**
　　.spec.selector是可选字段，用来指定 label selector ，圈定Deployment管理的pod范围。如果被指定， .spec.selector 必须匹配 .spec.template.metadata.labels，否则它将被API拒绝。如果 .spec.selector 没有被指定， .spec.selector.matchLabels 默认是.spec.template.metadata.labels。

　　在Pod的template跟.spec.template不同或者数量超过了.spec.replicas规定的数量的情况下，Deployment会杀掉label跟selector不同的Pod。

**Pod Template（Pod模板）：**
　　.spec.template 是 .spec中唯一要求的字段。

　　.spec.template 是 pod template. 它跟 Pod有一模一样的schema，除了它是嵌套的并且不需要apiVersion 和 kind字段。

　　另外为了划分Pod的范围，Deployment中的pod template必须指定适当的label（不要跟其他controller重复了，参考selector）和适当的重启策略。

　　.spec.template.spec.restartPolicy 可以设置为 Always , 如果不指定的话这就是默认配置。

**strategy（更新策略）：**
　　.spec.strategy 指定新的Pod替换旧的Pod的策略。 .spec.strategy.type 可以是"**Recreate**"或者是 "**RollingUpdate**"。**"RollingUpdate"是默认值**。

　　**Recreate：** 重建式更新，就是删一个建一个。类似于ReplicaSet的更新方式，即首先删除现有的Pod对象，然后由控制器基于新模板重新创建新版本资源对象。

　　**rollingUpdate：**滚动更新，简单定义 更新期间pod最多有几个等。可以指定**`maxUnavailable`** 和 **`maxSurge`** 来控制 rolling update 进程。

　　**maxSurge：**`.spec.strategy.rollingUpdate.maxSurge` 是可选配置项，用来指定可以超过期望的Pod数量的最大个数。该值可以是一个绝对值（例如5）或者是期望的Pod数量的百分比（例如10%）。当`MaxUnavailable`为0时该值不可以为0。通过百分比计算的绝对值向上取整。默认值是1。

　　**例如，**该值设置成30%，启动rolling update后新的ReplicatSet将会立即扩容，新老Pod的总数不能超过期望的Pod数量的130%。旧的Pod被杀掉后，新的ReplicaSet将继续扩容，旧的ReplicaSet会进一步缩容，确保在升级的所有时刻所有的Pod数量和不会超过期望Pod数量的130%。

　　**maxUnavailable：**`.spec.strategy.rollingUpdate.maxUnavailable` 是可选配置项，用来指定在升级过程中不可用Pod的最大数量。该值可以是一个绝对值（例如5），也可以是期望Pod数量的百分比（例如10%）。通过计算百分比的绝对值向下取整。 如果`.spec.strategy.rollingUpdate.maxSurge` 为0时，这个值不可以为0。默认值是1。

　　**例如，**该值设置成30%，启动rolling update后旧的ReplicatSet将会立即缩容到期望的Pod数量的70%。新的Pod ready后，随着新的ReplicaSet的扩容，旧的ReplicaSet会进一步缩容确保在升级的所有时刻可以用的Pod数量至少是期望Pod数量的70%。

<img src="https://img2018.cnblogs.com/blog/1349539/201902/1349539-20190226135633385-476220101.png" alt="img" style="zoom:67%;" />

**PS：maxSurge和maxUnavailable的属性值不可同时为0，否则Pod对象的副本数量在符合用户期望的数量后无法做出合理变动以进行更新操作。**

　　**在配置时，用户还可以使用Deployment控制器的spec.minReadySeconds属性来控制应用升级的速度。新旧更替过程中，新创建的Pod对象一旦成功响应就绪探测即被认为是可用状态，然后进行下一轮的替换。而spec.minReadySeconds能够定义在新的Pod对象创建后至少需要等待多长的时间才能会被认为其就绪，在该段时间内，更新操作会被阻塞。**

 

**revisionHistoryLimit（历史版本记录）：**
　　Deployment revision history存储在它控制的ReplicaSets中。默认保存记录10个　　

　　.spec.revisionHistoryLimit 是一个可选配置项，用来指定可以保留的旧的ReplicaSet数量。该理想值取决于心Deployment的频率和稳定性。如果该值没有设置的话，默认所有旧的Replicaset或会被保留，将资源存储在etcd中，是用kubectl get rs查看输出。每个Deployment的该配置都保存在ReplicaSet中，然而，一旦删除的旧的RepelicaSet，Deployment就无法再回退到那个revison了。

　　如果将该值设置为0，所有具有0个replica的ReplicaSet都会被删除。在这种情况下，新的Deployment rollout无法撤销，因为revision history都被清理掉了。

**PS：为了保存版本升级的历史，需要再创建Deployment对象时，在命令中使用"--record"选项**

 

**rollbackTo：**　　　　　　

 　`.spec.rollbackTo` 是一个可以选配置项，用来配置Deployment回退的配置。设置该参数将触发回退操作，每次回退完成后，该值就会被清除。

　　 **revision：**`.spec.rollbackTo.revision`是一个可选配置项，用来指定回退到的revision。默认是0，意味着回退到上一个revision。

**progressDeadlineSeconds：**　　

`　　.spec.progressDeadlineSeconds` 是可选配置项，用来指定在系统报告Deployment的[failed progressing](https://kubernetes.io/docs/concepts/workloads/controllers/deployment.md#failed-deployment)——表现为resource的状态中`type=Progressing`、`Status=False`、 `Reason=ProgressDeadlineExceeded`前可以等待的Deployment进行的秒数。Deployment controller会继续重试该Deployment。未来，在实现了自动回滚后， deployment controller在观察到这种状态时就会自动回滚。

　　如果设置该参数，该值必须大于 `.spec.minReadySeconds`。

**paused：**

　`.spec.paused`是可以可选配置项，boolean值。用来指定暂停和恢复Deployment。Paused和没有paused的Deployment之间的唯一区别就是，所有对paused deployment中的PodTemplateSpec的修改都不会触发新的rollout。Deployment被创建之后默认是非paused。　

### **2、创建Deployment**

```
[root@k8s-master ~]# vim deploy-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: myapp-deploy
    namespace: default
spec:
    replicas: 2
    selector:
        matchLabels:
            app: myapp
            release: canary
    template:
        metadata:
            labels: 
                app: myapp
                release: canary
        spec:
            containers:
            - name: myapp
              image: ikubernetes/myapp:v1
              ports:
              - name: http
                containerPort: 80

[root@k8s-master ~]# kubectl apply -f deploy-demo.yaml
[root@k8s-master ~]# kubectl get deploy
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
myapp-deploy       2         0         0            0           1s
[root@k8s-master ~]# kubectl get rs
```

输出结果表明我们希望的repalica数是2（根据deployment中的`.spec.replicas`配置）当前replica数（ `.status.replicas`）是0, 最新的replica数（`.status.updatedReplicas`）是0，可用的replica数（`.status.availableReplicas`）是0。过几秒后再执行`get`命令，将获得如下输出：

```
[root@k8s-master ~]# kubectl get deploy
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
myapp-deploy       2         2         2            2           10s
```

我们可以看到Deployment已经创建了2个 replica，所有的 replica 都已经是最新的了（包含最新的pod template），可用的（根据Deployment中的`.spec.minReadySeconds`声明，处于已就绪状态的pod的最少个数）。执行`kubectl get rs`和`kubectl get pods`会显示Replica Set（RS）和Pod已创建。

```
[root@k8s-master ~]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
myapp-deploy-2035384211       2         2         0       18s
```

 ReplicaSet 的名字总是`<Deployment的名字>-<pod template的hash值>`。

```
[root@k8s-master ~]# kubectl get pods --show-labels
NAME                            READY     STATUS    RESTARTS   AGE       LABELS
myapp-deploy-2035384211-7ci7o   1/1       Running   0          10s       app=myapp,release=canary,pod-template-hash=2035384211
myapp-deploy-2035384211-kzszj   1/1       Running   0          10s       app=myapp,release=canary,pod-template-hash=2035384211
```

刚创建的Replica Set将保证总是有2个myapp的 pod 存在。

**注意：** **在 Deployment 中的 selector 指定正确的 pod template label（在该示例中是 `app = myapp,release=canary`），不要跟其他的 controller 的 selector 中指定的 pod template label 搞混了（包括 Deployment、Replica Set、Replication Controller 等）。**

上面示例输出中的 pod label 里的 pod-template-hash label。当 Deployment 创建或者接管 ReplicaSet 时，Deployment controller 会自动为 Pod 添加 pod-template-hash label。这样做的目的是防止 Deployment 的子ReplicaSet 的 pod 名字重复。通过将 ReplicaSet 的PodTemplate 进行哈希散列，使用生成的哈希值作为 label 的值，并添加到 ReplicaSet selector 里、 pod template label 和 ReplicaSet 管理中的 Pod 上。

### **3、Deployment更新升级**

**（1）通过直接更改yaml的方式进行升级，如下：**

```
[root@k8s-master ~]# vim deploy-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: myapp-deploy
    namespace: default
spec:
    replicas: 2
    selector:
        matchLabels:
            app: myapp
            release: canary
    template:
        metadata:
            labels: 
                app: myapp
                release: canary
        spec:
            containers:
            - name: myapp
              image: ikubernetes/myapp:v2
              ports:
              - name: http
                containerPort: 80
[root@k8s-master ~]# kubectl apply -f deploy.yaml
```

**升级过程(我们看到，是停止一台，升级一台的这种循环。)**

![img](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)

```
[root@k8s-master ~]# kubectl get pods -l app=myapp -w
NAME                           READY     STATUS    RESTARTS   AGE
myapp-deploy-f4bcc4799-cs5xc   1/1       Running   0          23m
myapp-deploy-f4bcc4799-cwzd9   1/1       Running   0         14m
myapp-deploy-f4bcc4799-k4tq5   1/1       Running   0         23m
myapp-deploy-f4bcc4799-shbmb   1/1       Running   0         14m
myapp-deploy-f4bcc4799-vtk6m   1/1       Running   0         14m


myapp-deploy-f4bcc4799-shbmb   1/1       Terminating   0         16m
myapp-deploy-869b888f66-mv5c6   0/1       Pending   0         0s
myapp-deploy-869b888f66-mv5c6   0/1       Pending   0         0s
myapp-deploy-869b888f66-vk9j6   0/1       Pending   0         0s
myapp-deploy-869b888f66-vk9j6   0/1       Pending   0         0s
myapp-deploy-869b888f66-mv5c6   0/1       ContainerCreating   0         0s
myapp-deploy-869b888f66-r57t5   0/1       Pending   0         0s
myapp-deploy-869b888f66-r57t5   0/1       Pending   0         0s
myapp-deploy-869b888f66-vk9j6   0/1       ContainerCreating   0         1s
myapp-deploy-869b888f66-r57t5   0/1       ContainerCreating   0         1s
myapp-deploy-869b888f66-r57t5   0/1       ContainerCreating   0         1s
myapp-deploy-869b888f66-mv5c6   0/1       ContainerCreating   0         1s
myapp-deploy-869b888f66-vk9j6   0/1       ContainerCreating   0         2s
myapp-deploy-f4bcc4799-shbmb   0/1       Terminating   0         16m
myapp-deploy-f4bcc4799-shbmb   0/1       Terminating   0         16m
myapp-deploy-869b888f66-r57t5   1/1       Running   0         4s
myapp-deploy-f4bcc4799-vtk6m   1/1       Terminating   0         16m
myapp-deploy-869b888f66-rxzbb   0/1       Pending   0         1s
myapp-deploy-869b888f66-rxzbb   0/1       Pending   0         1s
myapp-deploy-869b888f66-rxzbb   0/1       ContainerCreating   0         1s
myapp-deploy-869b888f66-vk9j6   1/1       Running   0         5s
myapp-deploy-f4bcc4799-cwzd9   1/1       Terminating   0         16m
myapp-deploy-869b888f66-vvwwv   0/1       Pending   0         0s
myapp-deploy-869b888f66-vvwwv   0/1       Pending   0         0s
.....
```

**查看一下 rs的情况，以下可以看到原的rs作为备份，而现在是启动新的rs**

```
[root@k8s-master ~]# kubectl get rs -o wide
NAME                      DESIRED   CURRENT   READY     AGE       CONTAINER(S)       IMAGE(S)               SELECTOR
myapp-deploy-869b888f66   2         2         2         3m        myapp-containers   ikubernetes/myapp:v2   app=myapp,pod-template-hash=4256444922,release=canary
myapp-deploy-f4bcc4799    0         0         0         29m       myapp-containers   ikubernetes/myapp:v1   app=myapp,pod-template-hash=906770355,release=canary
```

**（2）通过set 命令直接修改image的版本进行升级，如下：**

```
[root@k8s-master ~]# kubectl set image deployment/myapp-deploy myapp=ikubernetes/myapp:v2
```

### **4、Deployment扩容**

**（1）使用以下命令扩容 Deployment：**

```
[root@k8s-master ~]# kubectl scale deployment myapp-deploy --replicas 5
```

**（2）直接修改yaml文件的方式进行扩容：**

```
[root@k8s-master ~]# vim demo.yaml
修改.spec.replicas的值
spec:
  replicas: 5
[root@k8s-master ~]# kubectl apply -f demo.yaml
```

**（3）通过打补丁的方式进行扩容：**

```
[root@k8s-master ~]# kubectl patch deployment myapp-deploy -p '{"spec":{"replicas":5}}'
[root@k8s-master ~]# kuebctl get pods
```

### **5、修改滚动更新策略**

**可以通过打补丁的方式进行修改更新策略，如下：**

```
[root@k8s-master ~]# kubectl patch deployment myapp-deploy -p '{"spec":{"strategy":{"rollingupdate":{"maxsurge“:1,"maxUnavailable":0}}}}'
[root@k8s-master ~]# kubectl describe deploy myapp-deploy
Name:            myapp-deploy
Namespace:        default
CreationTimestamp:    Tue, 28 Aug 2018 09:52:03 -0400
Labels:            app=myapp
            release=canary
Annotations:        deployment.kubernetes.io/revision=4
            kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"myapp-deploy","namespace":"default"},"spec":{"replicas":3,"selector":{...
Selector:        app=myapp,release=dev
Replicas:        3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:        RollingUpdate
MinReadySeconds:    0
RollingUpdateStrategy:    0 max unavailable, 1 max surge
Pod Template:
....
```

###  **6、金丝雀发布** 

Deployment控制器支持自定义控制更新过程中的滚动节奏，如“暂停(pause)”或“继续(resume)”更新操作。比如等待第一批新的Pod资源创建完成后立即暂停更新过程，此时，仅存在一部分新版本的应用，主体部分还是旧的版本。然后，再筛选一小部分的用户请求路由到新版本的Pod应用，继续观察能否稳定地按期望的方式运行。确定没问题之后再继续完成余下的Pod资源滚动更新，否则立即回滚更新操作。这就是所谓的金丝雀发布（Canary Release），如下命令演示：

```
（1）更新deployment的v3版本，并配置暂停deployment
[root@k8s-master ~]# kubectl set image deployment myapp-deploy myapp=ikubernetes/myapp:v3 && kubectl rollout pause deployment myapp-deploy    
deployment "myapp-deploy" image updated
deployment "myapp-deploy" paused
[root@k8s-master ~]# kubectl rollout status deployments myapp-deploy　　#观察更新状态

（2）监控更新的过程，可以看到已经新增了一个资源，但是并未按照预期的状态去删除一个旧的资源，就是因为使用了pause暂停命令
[root@k8s-master ~]# kubectl get pods -l app=myapp -w 
NAME                            READY     STATUS    RESTARTS   AGE
myapp-deploy-869b888f66-dpwvk   1/1       Running   0          24m
myapp-deploy-869b888f66-frspv   1/1       Running   0         24m
myapp-deploy-869b888f66-sgsll   1/1       Running   0         24m
myapp-deploy-7cbd5b69b9-5s4sq   0/1       Pending   0         0s
myapp-deploy-7cbd5b69b9-5s4sq   0/1       Pending   0         0s
myapp-deploy-7cbd5b69b9-5s4sq   0/1       ContainerCreating   0         1s
myapp-deploy-7cbd5b69b9-5s4sq   0/1       ContainerCreating   0         2s
myapp-deploy-7cbd5b69b9-5s4sq   1/1       Running   0         19s

（3）确保更新的pod没问题了，继续更新
[root@k8s-master ~]# kubectl rollout resume deploy  myapp-deploy

（4）查看最后的更新情况
[root@k8s-master ~]# kubectl get pods -l app=myapp -w 
NAME                            READY     STATUS    RESTARTS   AGE
myapp-deploy-869b888f66-dpwvk   1/1       Running   0          24m
myapp-deploy-869b888f66-frspv   1/1       Running   0         24m
myapp-deploy-869b888f66-sgsll   1/1       Running   0         24m
myapp-deploy-7cbd5b69b9-5s4sq   0/1       Pending   0         0s
myapp-deploy-7cbd5b69b9-5s4sq   0/1       Pending   0         0s
myapp-deploy-7cbd5b69b9-5s4sq   0/1       ContainerCreating   0         1s
myapp-deploy-7cbd5b69b9-5s4sq   0/1       ContainerCreating   0         2s
myapp-deploy-7cbd5b69b9-5s4sq   1/1       Running   0         19s
myapp-deploy-869b888f66-dpwvk   1/1       Terminating   0         31m
myapp-deploy-7cbd5b69b9-p6kzm   0/1       Pending   0         1s
myapp-deploy-7cbd5b69b9-p6kzm   0/1       Pending   0         1s
myapp-deploy-7cbd5b69b9-p6kzm   0/1       ContainerCreating   0         1s
myapp-deploy-7cbd5b69b9-p6kzm   0/1       ContainerCreating   0         2s
myapp-deploy-869b888f66-dpwvk   0/1       Terminating   0         31m
myapp-deploy-869b888f66-dpwvk   0/1       Terminating   0         31m
myapp-deploy-869b888f66-dpwvk   0/1       Terminating   0         31m
myapp-deploy-7cbd5b69b9-p6kzm   1/1       Running   0         18s
myapp-deploy-869b888f66-frspv   1/1       Terminating   0         31m
myapp-deploy-7cbd5b69b9-q8mvs   0/1       Pending   0         0s
myapp-deploy-7cbd5b69b9-q8mvs   0/1       Pending   0         0s
myapp-deploy-7cbd5b69b9-q8mvs   0/1       ContainerCreating   0         0s
myapp-deploy-7cbd5b69b9-q8mvs   0/1       ContainerCreating   0         1s
myapp-deploy-869b888f66-frspv   0/1       Terminating   0         31m
myapp-deploy-869b888f66-frspv   0/1       Terminating   0         31m
myapp-deploy-869b888f66-frspv   0/1       Terminating   0         31m
myapp-deploy-869b888f66-frspv   0/1       Terminating   0         31m
......
```

### 7**、Deployment版本回退**

　　默认情况下，kubernetes 会在系统中保存前两次的 Deployment 的 rollout 历史记录，以便可以随时回退（您可以修改`revision history limit`来更改保存的revision数）。

　　注意： 只要 Deployment 的 rollout 被触发就会创建一个 revision。也就是说当且仅当 Deployment 的 Pod template（如`.spec.template`）被更改，例如更新template 中的 label 和容器镜像时，就会创建出一个新的 revision。

其他的更新，比如扩容 Deployment 不会创建 revision——因此我们可以很方便的手动或者自动扩容。这意味着当您回退到历史 revision 时，只有 Deployment 中的 Pod template 部分才会回退。

```
[root@k8s-master ~]#  kubectl rollout history deploy  myapp-deploy　　#检查Deployment升级记录
deployments "myapp-deploy"
REVISION    CHANGE-CAUSE
0        <none>
3        <none>
4        <none>
5        <none>
```

这里在创建deployment时没有增加--record参数，所以并不能看到revision的变化。在创建 Deployment 的时候使用了`--record`参数可以记录命令，就可以方便的查看每次 revision 的变化。

**查看单个revision 的详细信息：**

```
[root@k8s-master ~]# kubectl rollout history deployment/myapp-deploy --revision=2
```

回退历史版本，默认是回退到上一个版本：

```
[root@k8s-master ~]# kubectl rollout undo deployment/myapp-deploy
deployment "myapp-deploy" rolled back
```

也可以使用 `--revision`参数指定某个历史版本：

```
[root@k8s-master ~]# kubectl rollout undo deployment/myapp-deploy --to-revision=2
deployment "myapp-deploy" rolled back
```

 

## 四、DaemonSet控制器

DaemonSet 确保全部（或者一些）Node 上运行一个 Pod 的副本。当有 Node 加入集群时，也会为他们新增一个 Pod 。当有 Node 从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod。

使用 DaemonSet 的一些典型用法：

- 运行集群存储 daemon，例如在每个 Node 上运行 `glusterd`、`ceph`。
- 在每个 Node 上运行日志收集 daemon，例如`fluentd`、`logstash`。
- 在每个 Node 上运行监控 daemon，例如 [Prometheus Node Exporter](https://github.com/prometheus/node_exporter)、`collectd`、Datadog 代理、New Relic 代理，或 Ganglia `gmond`。

一个简单的用法是，在所有的 Node 上都存在一个 DaemonSet，将被作为每种类型的 daemon 使用。 一个稍微复杂的用法可能是，对单独的每种类型的 daemon 使用多个 DaemonSet，但具有不同的标志，和/或对不同硬件类型具有不同的内存、CPU要求。

#### 1、编写DaemonSet Spec

**（1）必需字段**

和其它所有 Kubernetes 配置一样，DaemonSet 需要 `apiVersion`、`kind` 和 `metadata`字段。

```
[root@k8s-master ~]# kubectl explain daemonset
KIND:     DaemonSet
VERSION:  extensions/v1beta1

DESCRIPTION:
     DEPRECATED - This group version of DaemonSet is deprecated by
     apps/v1beta2/DaemonSet. See the release notes for more information.
     DaemonSet represents the configuration of a daemon set.

FIELDS:
   apiVersion    <string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#resources

   kind    <string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#types-kinds

   metadata    <Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#metadata

   spec    <Object>
     The desired behavior of this daemon set. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status

   status    <Object>
     The current status of this daemon set. This data may be out of date by some
     window of time. Populated by the system. Read-only. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status
```

**（2）Pod模板**

`.spec` 唯一必需的字段是 `.spec.template`。

`.spec.template` 是一个 [Pod 模板](https://kubernetes.io/docs/user-guide/replication-controller/#pod-template)。 它与 [Pod](https://kubernetes.io/docs/user-guide/pods) 具有相同的 schema，除了它是嵌套的，而且不具有 `apiVersion` 或 `kind` 字段。

Pod 除了必须字段外，在 DaemonSet 中的 Pod 模板必须指定合理的标签（查看 [pod selector](https://jimmysong.io/kubernetes-handbook/concepts/daemonset.html#pod-selector)）。

在 DaemonSet 中的 Pod 模板必需具有一个值为 `Always` 的 [`RestartPolicy`](https://kubernetes.io/docs/user-guide/pod-states)，或者未指定它的值，默认是 `Always`。

```
[root@k8s-master ~]# kubectl explain daemonset.spec.template.spec
```

 **（3）Pod Seletor**

`.spec.selector` 字段表示 Pod Selector，它与 [Job](https://kubernetes.io/docs/concepts/jobs/run-to-completion-finite-workloads/) 或其它资源的 `.sper.selector` 的原理是相同的。

`spec.selector` 表示一个对象，它由如下两个字段组成：

- `matchLabels` - 与 [ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) 的 `.spec.selector` 的原理相同。
- `matchExpressions` - 允许构建更加复杂的 Selector，可以通过指定 key、value 列表，以及与 key 和 value 列表的相关的操作符。

当上述两个字段都指定时，结果表示的是 AND 关系。

如果指定了 `.spec.selector`，必须与 `.spec.template.metadata.labels` 相匹配。如果没有指定，它们默认是等价的。如果与它们配置的不匹配，则会被 API 拒绝。

如果 Pod 的 label 与 selector 匹配，或者直接基于其它的 DaemonSet、或者 Controller（例如 ReplicationController），也不可以创建任何 Pod。 否则 DaemonSet Controller 将认为那些 Pod 是它创建的。Kubernetes 不会阻止这样做。一个场景是，可能希望在一个具有不同值的、用来测试用的 Node 上手动创建 Pod。

**（4）Daemon Pod通信**

与 DaemonSet 中的 Pod 进行通信，几种可能的模式如下：

- Push：配置 DaemonSet 中的 Pod 向其它 Service 发送更新，例如统计数据库。它们没有客户端。
- NodeIP 和已知端口：DaemonSet 中的 Pod 可以使用 `hostPort`，从而可以通过 Node IP 访问到 Pod。客户端能通过某种方法知道 Node IP 列表，并且基于此也可以知道端口。
- DNS：创建具有相同 Pod Selector 的 [Headless Service](https://kubernetes.io/docs/user-guide/services/#headless-services)，然后通过使用 `endpoints` 资源或从 DNS 检索到多个 A 记录来发现 DaemonSet。
- Service：创建具有相同 Pod Selector 的 Service，并使用该 Service 访问到某个随机 Node 上的 daemon。（没有办法访问到特定 Node）

####  2、创建redis-filebeat的DaemonSet演示

```
（1）编辑daemonSet的yaml文件
可以在同一个yaml文件中定义多个资源，这里将redis和filebeat定在一个文件当中

[root@k8s-master mainfests]# vim ds-demo.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
      role: logstor
  template:
    metadata:
      labels:
        app: redis
        role: logstor
    spec:
      containers:
      - name: redis
        image: redis:4.0-alpine
        ports:
        - name: redis
          containerPort: 6379
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat-ds
  namespace: default
spec:
  selector:
    matchLabels:
      app: filebeat
      release: stable
  template:
    metadata:
      labels: 
        app: filebeat
        release: stable
    spec:
      containers:
      - name: filebeat
        image: ikubernetes/filebeat:5.6.5-alpine
        env:
        - name: REDIS_HOST
          value: redis.default.svc.cluster.local
        - name: REDIS_LOG_LEVEL
          value: info

（2）创建pods                
[root@k8s-master mainfests]# kubectl apply -f ds-demo.yaml
deployment.apps/redis created
daemonset.apps/filebeat-ds created

（3）暴露端口
[root@k8s-master mainfests]# kubectl expose deployment redis --port=6379
service/redis exposed
[root@k8s-master mainfests]# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        16d
myapp        NodePort    10.106.67.242    <none>        80:32432/TCP   13d
nginx        ClusterIP   10.106.162.254   <none>        80/TCP         14d
redis        ClusterIP   10.107.163.143   <none>        6379/TCP       4s

[root@k8s-master mainfests]# kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
filebeat-ds-rpp9p        1/1       Running   0          5m
filebeat-ds-vwx7d        1/1       Running   0          5m
pod-demo                 2/2       Running   6          5d
redis-5b5d6fbbbd-v82pw   1/1       Running   0          36s

（4）测试redis是否收到日志
[root@k8s-master mainfests]# kubectl exec -it redis-5b5d6fbbbd-v82pw -- /bin/sh
/data # netstat -tnl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       
tcp        0      0 0.0.0.0:6379            0.0.0.0:*               LISTEN      
tcp        0      0 :::6379                 :::*                    LISTEN      

/data # nslookup redis.default.svc.cluster.local
nslookup: can't resolve '(null)': Name does not resolve

Name:      redis.default.svc.cluster.local
Address 1: 10.107.163.143 redis.default.svc.cluster.local

/data # redis-cli -h redis.default.svc.cluster.local
redis.default.svc.cluster.local:6379> KEYS *　　#由于redis在filebeat后面才启动，日志可能已经发走了，所以查看key为空
(empty list or set)

[root@k8s-master mainfests]# kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
filebeat-ds-rpp9p        1/1       Running   0          14m
filebeat-ds-vwx7d        1/1       Running   0          14m
pod-demo                 2/2       Running   6          5d
redis-5b5d6fbbbd-v82pw   1/1       Running   0          9m
[root@k8s-master mainfests]# kubectl exec -it filebeat-ds-rpp9p -- /bin/sh
/ # cat /etc/filebeat/filebeat.yml 
filebeat.registry_file: /var/log/containers/filebeat_registry
filebeat.idle_timeout: 5s
filebeat.spool_size: 2048

logging.level: info

filebeat.prospectors:
- input_type: log
  paths:
    - "/var/log/containers/*.log"
    - "/var/log/docker/containers/*.log"
    - "/var/log/startupscript.log"
    - "/var/log/kubelet.log"
    - "/var/log/kube-proxy.log"
    - "/var/log/kube-apiserver.log"
    - "/var/log/kube-controller-manager.log"
    - "/var/log/kube-scheduler.log"
    - "/var/log/rescheduler.log"
    - "/var/log/glbc.log"
    - "/var/log/cluster-autoscaler.log"
  symlinks: true
  json.message_key: log
  json.keys_under_root: true
  json.add_error_key: true
  multiline.pattern: '^\s'
  multiline.match: after
  document_type: kube-logs
  tail_files: true
  fields_under_root: true

output.redis:
  hosts: ${REDIS_HOST:?No Redis host configured. Use env var REDIS_HOST to set host.}
  key: "filebeat"

[root@k8s-master mainfests]# kubectl get pods -l app=filebeat -o wide
NAME                READY     STATUS    RESTARTS   AGE       IP            NODE
filebeat-ds-rpp9p   1/1       Running   0          16m       10.244.2.12   k8s-node02
filebeat-ds-vwx7d   1/1       Running   0          16m       10.244.1.15   k8s-node01
```



#### 3、DaemonSet的滚动更新

DaemonSet有两种更新策略类型：

- OnDelete：这是向后兼容性的默认更新策略。使用 `OnDelete`更新策略，在更新DaemonSet模板后，只有在手动删除旧的DaemonSet pod时才会创建新的DaemonSet pod。这与Kubernetes 1.5或更早版本中DaemonSet的行为相同。
- RollingUpdate：使用`RollingUpdate`更新策略，在更新DaemonSet模板后，旧的DaemonSet pod将被终止，并且将以受控方式自动创建新的DaemonSet pod。

要启用DaemonSet的滚动更新功能，必须将其设置 `.spec.updateStrategy.type`为`RollingUpdate`。

（1）查看当前的更新策略：

```
[root@k8s-master mainfests]# kubectl get ds/filebeat-ds -o go-template='{{.spec.updateStrategy.type}}{{"\n"}}'
RollingUpdate
```

（2）更新DaemonSet模板

对`RollingUpdate`DaemonSet的任何更新都`.spec.template`将触发滚动更新。这可以通过几个不同的`kubectl`命令来完成。

声明式命令方式：

如果使用配置文件进行更新DaemonSet，可以使用kubectl aapply：

```
kubectl apply -f ds-demo.yaml
```

补丁式命令方式：

```
kubectl edit ds/filebeat-ds

kubectl patch ds/filebeat-ds -p=<strategic-merge-patch>
```

仅仅更新容器镜像还可以使用以下命令：

```
kubectl set image ds/<daemonset-name> <container-name>=<container-new-image>
```

下面对filebeat-ds的镜像进行版本更新，如下：

```
[root@k8s-master mainfests]# kubectl set image daemonsets filebeat-ds filebeat=ikubernetes/filebeat:5.6.6-alpine
daemonset.extensions/filebeat-ds image updated

[root@k8s-master mainfests]# kubectl get pods -w　　#观察滚动更新状态
NAME                     READY     STATUS        RESTARTS   AGE
filebeat-ds-rpp9p        1/1       Running       0          27m
filebeat-ds-vwx7d        0/1       Terminating   0          27m
pod-demo                 2/2       Running       6          5d
redis-5b5d6fbbbd-v82pw   1/1       Running       0          23m
filebeat-ds-vwx7d   0/1       Terminating   0         27m
filebeat-ds-vwx7d   0/1       Terminating   0         27m
filebeat-ds-s466l   0/1       Pending   0         0s
filebeat-ds-s466l   0/1       ContainerCreating   0         0s
filebeat-ds-s466l   1/1       Running   0         13s
filebeat-ds-rpp9p   1/1       Terminating   0         28m
filebeat-ds-rpp9p   0/1       Terminating   0         28m
filebeat-ds-rpp9p   0/1       Terminating   0         28m
filebeat-ds-rpp9p   0/1       Terminating   0         28m
filebeat-ds-hxgdx   0/1       Pending   0         0s
filebeat-ds-hxgdx   0/1       ContainerCreating   0         0s
filebeat-ds-hxgdx   1/1       Running   0         28s

[root@k8s-master mainfests]# kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
filebeat-ds-hxgdx        1/1       Running   0          2m
filebeat-ds-s466l        1/1       Running   0          2m
pod-demo                 2/2       Running   6          5d
redis-5b5d6fbbbd-v82pw   1/1       Running   0          25m
```

从上面的滚动更新，可以看到在更新过程中，是先终止旧的pod，再创建一个新的pod，逐步进行替换的，这就是DaemonSet的滚动更新策略！



# Service服务发现

## 一、Service的概念

　　运行在Pod中的应用是向客户端提供服务的守护进程，比如，nginx、tomcat、etcd等等，它们都是受控于控制器的资源对象，存在生命周期，我们知道Pod资源对象在自愿或非自愿终端后，只能被重构的Pod对象所替代，属于不可再生类组件。而在动态和弹性的管理模式下，Service为该类Pod对象提供了一个固定、统一的访问接口和负载均衡能力。

　　其实，就是说Pod存在生命周期，有销毁，有重建，无法提供一个固定的访问接口给客户端。并且为了同类的Pod都能够实现工作负载的价值，由此Service资源出现了，可以为一类Pod资源对象提供一个固定的访问接口和负载均衡，类似于阿里云的负载均衡或者是LVS的功能。

　　但是要知道的是，Service和Pod对象的IP地址，一个是虚拟地址，一个是Pod IP地址，都仅仅在集群内部可以进行访问，无法接入集群外部流量。而为了解决该类问题的办法可以是在单一的节点上做端口暴露（hostPort）以及让Pod资源共享工作节点的网络名称空间（hostNetwork）以外，还可以使用NodePort或者是LoadBalancer类型的Service资源，或者是有7层负载均衡能力的Ingress资源。

　　Service是Kubernetes的核心资源类型之一，Service资源基于标签选择器将一组Pod定义成一个逻辑组合，并通过自己的IP地址和端口调度代理请求到组内的Pod对象，如下图所示，它向客户端隐藏了真是的，处理用户请求的Pod资源，使得从客户端上看，就像是由Service直接处理并响应一样，是不是很像负载均衡器呢！

![img](https://img2018.cnblogs.com/blog/1349539/201902/1349539-20190226153708890-1966593460.png)

　　Service对象的IP地址也称为Cluster IP，它位于为Kubernetes集群配置指定专用的IP地址范围之内，是一种虚拟的IP地址，它在Service对象创建之后保持不变，并且能够被同一集群中的Pod资源所访问。Service端口用于接受客户端请求，并将请求转发至后端的Pod应用的相应端口，这样的代理机制，也称为端口代理，它是基于TCP/IP 协议栈的传输层。

## 二、Service的实现模型

　　在 Kubernetes 集群中，每个 Node 运行一个 `kube-proxy` 进程。`kube-proxy` 负责为 `Service` 实现了一种 VIP（虚拟 IP）的形式，而不是 `ExternalName` 的形式。 在 Kubernetes v1.0 版本，代理完全在 userspace。在 Kubernetes v1.1 版本，新增了 iptables 代理，但并不是默认的运行模式。 从 Kubernetes v1.2 起，默认就是 iptables 代理。在Kubernetes v1.8.0-beta.0中，添加了ipvs代理。在 Kubernetes v1.0 版本，`Service` 是 “4层”（TCP/UDP over IP）概念。 在 Kubernetes v1.1 版本，新增了 `Ingress` API（beta 版），用来表示 “7层”（HTTP）服务。

kube-proxy 这个组件始终监视着apiserver中有关service的变动信息，获取任何一个与service资源相关的变动状态，通过watch监视，一旦有service资源相关的变动和创建，kube-proxy都要转换为当前节点上的能够实现资源调度规则（例如：iptables、ipvs）

![img](https://images2018.cnblogs.com/blog/1349539/201809/1349539-20180907165901269-453271720.png)

### 2.1、userspace代理模式

​		该模式下kube-proxy会为每一个Service创建一个监听端口。当客户端Pod发向Cluster IP的请求被Iptables规则重定向到Kube-proxy监听的端口上，Kube-proxy根据LB算法选择一个提供服务的Pod并和其建立链接，以将请求转发到Pod上。

　　由此可见这个模式有很大的问题，Kube-proxy充当了一个四层Load balancer的角色。由于kube-proxy运行在userspace中，在进行转发处理时会增加两次内核和用户空间之间的数据拷贝，这样流量从用户空间进出内核带来的性能损耗是不可接受的。好处是当后端的Pod不可用时，kube-proxy可以重试其他Pod。在Kubernetes 1.1版本之前，userspace是默认的代理模型。

<img src="https://images2018.cnblogs.com/blog/1349539/201809/1349539-20180907170415435-1497905641.png" alt="img" style="zoom:67%;" />

### 2.2、 iptables代理模式

​		在该模式下，Kube-proxy为service后端的每个Pod创建对应的iptables规则，客户端IP请求时，直接将发向Cluster IP的请求重定向到一个Pod IP。因为使用iptable NAT来完成转发，也存在不可忽视的性能损耗。另外，如果集群中存在上万的Service/Endpoint，那么Node上的iptables rules将会非常庞大，性能还会再打折扣。iptables代理模式由Kubernetes 1.1版本引入，自1.2版本开始成为默认类型。

​		该模式下Kube-proxy不承担四层代理的角色，只负责创建iptables规则。该模式的优点是较userspace模式效率更高，但不能提供灵活的LB策略，当后端Pod不可用时也无法进行重试。

<img src="https://images2018.cnblogs.com/blog/1349539/201809/1349539-20180907171245389-9573372.png" alt="img" style="zoom:67%;" />

###  2.3、ipvs代理模式

　　Kubernetes自1.9-alpha版本引入了ipvs代理模式，自1.11版本开始成为默认设置。客户端IP请求时到达内核空间时，根据ipvs的规则直接分发到各pod上。kube-proxy会监视Kubernetes `Service`对象和`Endpoints`，调用`netlink`接口以相应地创建ipvs规则并定期与Kubernetes `Service`对象和`Endpoints`对象同步ipvs规则，以确保ipvs状态与期望一致。访问服务时，流量将被重定向到其中一个后端Pod。

​		该模式和iptables类似，kube-proxy监控Pod的变化并创建相应的ipvs rules。ipvs也是在kernel模式下通过netfilter实现的，但采用了hash table来存储规则，因此在规则较多的情况下，Ipvs相对iptables转发效率更高。除此以外，ipvs支持更多的LB算法。如果要设置kube-proxy为ipvs模式，必须在操作系统中安装IPVS内核模块。

​		与iptables类似，ipvs基于netfilter 的 hook 功能，但使用哈希表作为底层数据结构并在内核空间中工作。这意味着ipvs可以更快地重定向流量，并且在同步代理规则时具有更好的性能。此外，ipvs为负载均衡算法提供了更多选项，例如：

- rr：`轮询调度`
- lc：最小连接数
- `dh`：目标哈希
- `sh`：源哈希
- `sed`：最短期望延迟
- `nq`：不排队调度

**注意： ipvs模式假定在运行kube-proxy之前在节点上都已经安装了IPVS内核模块。当kube-proxy以ipvs代理模式启动时，kube-proxy将验证节点上是否安装了IPVS模块，如果未安装，则kube-proxy将回退到iptables代理模式。**

<img src="https://images2018.cnblogs.com/blog/1349539/201809/1349539-20180907171512167-1381663777.png" alt="img" style="zoom:67%;" />

 如果某个服务后端pod发生变化，标签选择器适应的pod有多一个，适应的信息会立即反映到apiserver上,而kube-proxy一定可以watch到etcd中的信息变化，而将它立即转为ipvs或者iptables中的规则，这一切都是动态和实时的，删除一个pod也是同样的原理。如图：

![img](https://img2018.cnblogs.com/blog/1349539/201809/1349539-20180927135812407-1162084485.png)

## 三、Service的定义

**3.1、清单创建Service**

```shell
[root@master ~]# kubectl explain svc
KIND:     Service
VERSION:  v1

DESCRIPTION:
     Service is a named abstraction of software service (for example, mysql)
     consisting of local port (for example 3306) that the proxy listens on, and
     the selector that determines which pods will answer requests sent through
     the proxy.

FIELDS:
   apiVersion    <string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#resources

   kind    <string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#types-kinds

   metadata    <Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#metadata

   spec    <Object>
     Spec defines the behavior of a service.
     https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status

   status    <Object>
     Most recently observed status of the service. Populated by the system.
     Read-only. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status
```

其中重要的4个字段：

```yaml
apiVersion:
kind:
metadata:
spec:
　　clusterIP: 可以自定义，也可以动态分配
　　ports:（与后端容器端口关联）
　　selector:（关联到哪些pod资源上）
　　type：服务类型
```



**3.2、service的类型**

对一些应用（如 Frontend）的某些部分，可能希望通过外部（Kubernetes 集群外部）IP 地址暴露 Service。

Kubernetes `ServiceTypes` 允许指定一个需要的类型的 Service，默认是 `ClusterIP` 类型。

`Type` 的取值以及行为如下：

- hostNetwork
- hostPort

- **`ClusterIP`：**通过集群的内部 IP 暴露服务，选择该值，服务只能够在集群内部可以访问，这也是默认的 `ServiceType`。
- **`NodePort`：**通过每个 Node 上的 IP 和静态端口（`NodePort`）暴露服务。`NodePort` 服务会路由到 `ClusterIP` 服务，这个 `ClusterIP` 服务会自动创建。通过请求 `<NodeIP>:<NodePort>`，可以从集群的外部访问一个 `NodePort` 服务。
- **`LoadBalancer`：**使用云提供商的负载均衡器，可以向外部暴露服务。外部的负载均衡器可以路由到 `NodePort` 服务和 `ClusterIP` 服务。
- **`ExternalName`：**通过返回 `CNAME` 和它的值，可以将服务映射到 `externalName` 字段的内容（例如， `foo.bar.example.com`）。 没有任何类型代理被创建，这只有 Kubernetes 1.7 或更高版本的 `kube-dns` 才支持。

####  3.2.1、ClusterIP的service类型演示：

```
[root@k8s-master mainfests]# cat redis-svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: default
spec:
  selector:　　#标签选择器，必须指定pod资源本身的标签
    app: redis
    role: logstor
  type: ClusterIP　　#指定服务类型为ClusterIP
  ports: 　　#指定端口
  - port: 6379　　#暴露给服务的端口
  - targetPort: 6379　　#容器的端口
[root@k8s-master mainfests]# kubectl apply -f redis-svc.yaml 
service/redis created
[root@k8s-master mainfests]# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    36d
redis        ClusterIP   10.107.238.182   <none>        6379/TCP   1m

[root@k8s-master mainfests]# kubectl describe svc redis
Name:              redis
Namespace:         default
Labels:            <none>
Annotations:       kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"redis","namespace":"default"},"spec":{"ports":[{"port":6379,"targetPort":6379}...
Selector:          app=redis,role=logstor
Type:              ClusterIP
IP:                10.107.238.182　　#service ip
Port:              <unset>  6379/TCP
TargetPort:        6379/TCP
Endpoints:         10.244.1.16:6379　　#此处的ip+端口就是pod的ip+端口
Session Affinity:  None
Events:            <none>

[root@k8s-master mainfests]# kubectl get pod redis-5b5d6fbbbd-v82pw -o wide
NAME                     READY     STATUS    RESTARTS   AGE       IP            NODE
redis-5b5d6fbbbd-v82pw   1/1       Running   0          20d       10.244.1.16   k8s-node01
```

从上演示可以总结出：service不会直接到pod，service是直接到endpoint资源，就是地址加端口，再由endpoint再关联到pod。

service只要创建完，就会在dns中添加一个资源记录进行解析，添加完成即可进行解析。资源记录的格式为：SVC_NAME.NS_NAME.DOMAIN.LTD.

默认的集群service 的A记录：svc.cluster.local.

redis服务创建的A记录：redis.default.svc.cluster.local.

#### 3.2.2、NodePort的service类型演示： 

　　NodePort即节点Port，通常在部署Kubernetes集群系统时会预留一个端口范围用于NodePort，其范围默认为：30000~32767之间的端口。定义NodePort类型的Service资源时，需要使用.spec.type进行明确指定。

```
[root@k8s-master mainfests]# kubectl get pods --show-labels |grep myapp-deploy
myapp-deploy-69b47bc96d-4hxxw   1/1       Running   0          12m       app=myapp,pod-template-hash=2560367528,release=canary
myapp-deploy-69b47bc96d-95bc4   1/1       Running   0          12m       app=myapp,pod-template-hash=2560367528,release=canary
myapp-deploy-69b47bc96d-hwbzt   1/1       Running   0          12m       app=myapp,pod-template-hash=2560367528,release=canary
myapp-deploy-69b47bc96d-pjv74   1/1       Running   0          12m       app=myapp,pod-template-hash=2560367528,release=canary
myapp-deploy-69b47bc96d-rf7bs   1/1       Running   0          12m       app=myapp,pod-template-hash=2560367528,release=canary

[root@k8s-master mainfests]# cat myapp-svc.yaml #为myapp创建service
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: default
spec:
  selector:
    app: myapp
    release: canary
  type: NodePort
  ports: 
  - port: 80
    targetPort: 80
    nodePort: 30080
[root@k8s-master mainfests]# kubectl apply -f myapp-svc.yaml 
service/myapp created
[root@k8s-master mainfests]# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        36d
myapp        NodePort    10.101.245.119   <none>        80:30080/TCP   5s
redis        ClusterIP   10.107.238.182   <none>        6379/TCP       28m

[root@k8s-master mainfests]# while true;do curl http://192.168.56.11:30080/hostname.html;sleep 1;done
myapp-deploy-69b47bc96d-95bc4
myapp-deploy-69b47bc96d-4hxxw
myapp-deploy-69b47bc96d-pjv74
myapp-deploy-69b47bc96d-rf7bs
myapp-deploy-69b47bc96d-95bc4
myapp-deploy-69b47bc96d-rf7bs
myapp-deploy-69b47bc96d-95bc4
myapp-deploy-69b47bc96d-pjv74
myapp-deploy-69b47bc96d-4hxxw
myapp-deploy-69b47bc96d-pjv74
myapp-deploy-69b47bc96d-pjv74
myapp-deploy-69b47bc96d-4hxxw
myapp-deploy-69b47bc96d-pjv74
myapp-deploy-69b47bc96d-pjv74
myapp-deploy-69b47bc96d-pjv74
myapp-deploy-69b47bc96d-95bc4
myapp-deploy-69b47bc96d-hwbzt

[root@k8s-master mainfests]# while true;do curl http://192.168.56.11:30080/;sleep 1;done
```

 Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
 Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
 Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
 Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
 Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
 Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>

从以上例子，可以看到通过NodePort方式已经实现了从集群外部端口进行访问，访问链接如下：http://192.168.56.11:30080/。实践中并不鼓励用户自定义使用节点的端口，因为容易和其他现存的Service冲突，建议留给系统自动配置。

####  3.2.3、Pod的会话保持

　　Service资源还支持Session affinity（粘性会话）机制，可以将来自同一个客户端的请求始终转发至同一个后端的Pod对象，这意味着它会影响调度算法的流量分发功用，进而降低其负载均衡的效果。因此，当客户端访问Pod中的应用程序时，如果有基于客户端身份保存某些私有信息，并基于这些私有信息追踪用户的活动等一类的需求时，那么应该启用session affinity机制。

　　Service affinity的效果仅仅在一段时间内生效，默认值为10800秒，超出时长，客户端再次访问会重新调度。该机制仅能基于客户端IP地址识别客户端身份，它会将经由同一个NAT服务器进行原地址转换的所有客户端识别为同一个客户端，由此可知，其调度的效果并不理想。Service 资源 通过. spec. sessionAffinity 和. spec. sessionAffinityConfig 两个字段配置粘性会话。 spec. sessionAffinity 字段用于定义要使用的粘性会话的类型，它仅支持使用“ None” 和“ ClientIP” 两种属性值。如下：

```
[root@k8s-master mainfests]# kubectl explain svc.spec.sessionAffinity
KIND:     Service
VERSION:  v1

FIELD:    sessionAffinity <string>

DESCRIPTION:
     Supports "ClientIP" and "None". Used to maintain session affinity. Enable
     client IP based session affinity. Must be ClientIP or None. Defaults to
     None. More info:
     https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies
```

sessionAffinity支持ClientIP和None 两种方式，默认是None（随机调度） ClientIP是来自于同一个客户端的请求调度到同一个pod中

```
[root@k8s-master mainfests]# vim myapp-svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: default
spec:
  selector:
    app: myapp
    release: canary
  sessionAffinity: ClientIP
  type: NodePort
  ports: 
  - port: 80
    targetPort: 80
    nodePort: 30080
[root@k8s-master mainfests]# kubectl apply -f myapp-svc.yaml 
service/myapp configured
[root@k8s-master mainfests]# kubectl describe svc myapp
Name:                     myapp
Namespace:                default
Labels:                   <none>
Annotations:              kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"myapp","namespace":"default"},"spec":{"ports":[{"nodePort":30080,"port":80,"ta...
Selector:                 app=myapp,release=canary
Type:                     NodePort
IP:                       10.101.245.119
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30080/TCP
Endpoints:                10.244.1.18:80,10.244.1.19:80,10.244.2.15:80 + 2 more...
Session Affinity:         ClientIP
External Traffic Policy:  Cluster
Events:                   <none>
[root@k8s-master mainfests]# while true;do curl http://192.168.56.11:30080/hostname.html;sleep 1;done
myapp-deploy-69b47bc96d-hwbzt
myapp-deploy-69b47bc96d-hwbzt
myapp-deploy-69b47bc96d-hwbzt
myapp-deploy-69b47bc96d-hwbzt
myapp-deploy-69b47bc96d-hwbzt
myapp-deploy-69b47bc96d-hwbzt
myapp-deploy-69b47bc96d-hwbzt
myapp-deploy-69b47bc96d-hwbzt
```

也可以使用打补丁的方式进行修改yaml内的内容，如下：

```
kubectl patch svc myapp -p '{"spec":{"sessionAffinity":"ClusterIP"}}'  #session保持，同一ip访问同一个pod

kubectl patch svc myapp -p '{"spec":{"sessionAffinity":"None"}}'    #取消session 
```

## 四、Headless Service 

有时不需要或不想要负载均衡，以及单独的 Service IP。 遇到这种情况，可以通过指定 Cluster IP（`spec.clusterIP`）的值为 `"None"` 来创建 `Headless` Service。

这个选项允许开发人员自由寻找他们自己的方式，从而降低与 Kubernetes 系统的耦合性。 应用仍然可以使用一种自注册的模式和适配器，对其它需要发现机制的系统能够很容易地基于这个 API 来构建。

对这类 `Service` 并不会分配 Cluster IP，kube-proxy 不会处理它们，而且平台也不会为它们进行负载均衡和路由。 DNS 如何实现自动配置，依赖于 `Service` 是否定义了 selector。

```
（1）编写headless service配置清单
[root@k8s-master mainfests]# cp myapp-svc.yaml myapp-svc-headless.yaml 
[root@k8s-master mainfests]# vim myapp-svc-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-headless
  namespace: default
spec:
  selector:
    app: myapp
    release: canary
  clusterIP: "None"　　#headless的clusterIP值为None
  ports: 
  - port: 80
    targetPort: 80

（2）创建headless service 
[root@k8s-master mainfests]# kubectl apply -f myapp-svc-headless.yaml 
service/myapp-headless created
[root@k8s-master mainfests]# kubectl get svc
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes       ClusterIP   10.96.0.1        <none>        443/TCP        36d
myapp            NodePort    10.101.245.119   <none>        80:30080/TCP   1h
myapp-headless   ClusterIP   None             <none>        80/TCP         5s
redis            ClusterIP   10.107.238.182   <none>        6379/TCP       2h

（3）使用coredns进行解析验证
[root@k8s-master mainfests]# dig -t A myapp-headless.default.svc.cluster.local. @10.96.0.10

; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7 <<>> -t A myapp-headless.default.svc.cluster.local. @10.96.0.10
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 62028
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 5, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;myapp-headless.default.svc.cluster.local. IN A

;; ANSWER SECTION:
myapp-headless.default.svc.cluster.local. 5 IN A 10.244.1.18
myapp-headless.default.svc.cluster.local. 5 IN A 10.244.1.19
myapp-headless.default.svc.cluster.local. 5 IN A 10.244.2.15
myapp-headless.default.svc.cluster.local. 5 IN A 10.244.2.16
myapp-headless.default.svc.cluster.local. 5 IN A 10.244.2.17

;; Query time: 4 msec
;; SERVER: 10.96.0.10#53(10.96.0.10)
;; WHEN: Thu Sep 27 04:27:15 EDT 2018
;; MSG SIZE  rcvd: 349

[root@k8s-master mainfests]# kubectl get svc -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP   36d

[root@k8s-master mainfests]# kubectl get pods -o wide -l app=myapp
NAME                            READY     STATUS    RESTARTS   AGE       IP            NODE
myapp-deploy-69b47bc96d-4hxxw   1/1       Running   0          1h        10.244.1.18   k8s-node01
myapp-deploy-69b47bc96d-95bc4   1/1       Running   0          1h        10.244.2.16   k8s-node02
myapp-deploy-69b47bc96d-hwbzt   1/1       Running   0          1h        10.244.1.19   k8s-node01
myapp-deploy-69b47bc96d-pjv74   1/1       Running   0          1h        10.244.2.15   k8s-node02
myapp-deploy-69b47bc96d-rf7bs   1/1       Running   0          1h        10.244.2.17   k8s-node02

（4）对比含有ClusterIP的service解析
[root@k8s-master mainfests]# dig -t A myapp.default.svc.cluster.local. @10.96.0.10

; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7 <<>> -t A myapp.default.svc.cluster.local. @10.96.0.10
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 50445
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;myapp.default.svc.cluster.local. IN    A

;; ANSWER SECTION:
myapp.default.svc.cluster.local. 5 IN    A    10.101.245.119

;; Query time: 1 msec
;; SERVER: 10.96.0.10#53(10.96.0.10)
;; WHEN: Thu Sep 27 04:31:16 EDT 2018
;; MSG SIZE  rcvd: 107

[root@k8s-master mainfests]# kubectl get svc
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes       ClusterIP   10.96.0.1        <none>        443/TCP        36d
myapp            NodePort    10.101.245.119   <none>        80:30080/TCP   1h
myapp-headless   ClusterIP   None             <none>        80/TCP         11m
redis            ClusterIP   10.107.238.182   <none>        6379/TCP       2h
```

从以上的演示可以看到对比普通的service和headless service，headless service做dns解析是直接解析到pod的，而servcie是解析到ClusterIP的，那么headless有什么用呢？？？这将在statefulset中应用到，这里暂时仅仅做了解什么是headless service和创建方法。





# Ingress和Ingress Controller

 

service完成对资源分类，ingress通过service得知后端主机，ingress通过ingress controller注入并保存配置文件，ingress发现后端资源发生改变，及时注入到pod中。

 

## **一、什么是Ingress？**

我们可以了解到Kubernetes暴露服务的方式目前只有三种:NodePort、LoadBalancer和 ExtemalName ；而我们需要将集群内服务提供外界访问就会产生以下几个问题：



**1、Pod 漂移问题**

Kubernetes 具有强大的副本控制能力，能保证在任意副本（Pod）挂掉时自动从其他机器启动一个新的，还可以动态扩容等，通俗地说，这个 Pod 可能在任何时刻出现在任何节点上，也可能在任何时刻死在任何节点上；那么自然随着 Pod 的创建和销毁，Pod IP 肯定会动态变化；那么如何把这个动态的 Pod IP 暴露出去？这里借助于 Kubernetes 的 Service 机制，Service 可以以标签的形式选定一组带有指定标签的Pod，并监控和自动负载他们的 Pod IP，那么我们向外暴露只暴露Service IP 就行了；这就是 NodePort 

![图片 1](/Users/jinhuaiwang/Documents/k8s-onenote/basic-word/picture/图片 1.png)

 

**2、端口管理问题**

采用 NodePort 方式暴露服务的问题是，服务一旦多起来，NodePort 在每个节点上开启的端口会相当庞大，而且难以维护；这时，我们可以能否使用一个Nginx直接对内进行转发呢？众所周知的是，Pod与Pod之间是可以互相通信的，而Pod是可以共享宿主机的网络名称空间的，也就是说当在共享网络名称空间时，Pod上所监听的就是Node上的端口。那么这又该如何实现呢？简单的实现就是使用 DaemonSet 在每个 Node 上监听 80，然后写好规则，因为Nginx外面绑定了宿主机 80 端口（就像 NodePort），本身又在集群内，那么向后直接转发到相应 Service IP 就行了，



![图片 2](/Users/jinhuaiwang/Documents/k8s-onenote/basic-word/picture/图片 2.png)

 

**3、域名分配及动态更新问题**

 从上面的方法，采用 Nginx-Pod 已经解决了问题，但是其实这里面有一个很大缺陷：当每次有新服务加入又该如何修改 Nginx 配置呢？我们知道使用Nginx可以通过虚拟主机域名进行区分不同的服务，而每个服务通过upstream进行定义不同的负载均衡池，再加上location进行负载均衡的反向代理，在日常使用中只需要修改nginx.conf即可实现，那在K8S中又该如何实现这种方式的调度呢？

假设后端的服务初始服务只有ecshop，后面增加了bbs和member服务，那么又该如何将这2个服务加入到Nginx-Pod进行调度呢？不能每次手动改或者Rolling Update 前端 Nginx Pod 吧！！此时 Ingress 出现了，如果不算上面的Nginx，Ingress 包含两大组件：Ingress Controller 和 Ingress。

![图片 3](/Users/jinhuaiwang/Documents/k8s-onenote/basic-word/picture/图片 3.png)

 

Ingress 简单的理解就是你原来需要改 Nginx 配置，然后配置各种域名对应哪个 Service，现在把这个动作抽象出来，变成一个 Ingress 对象，你可以用 yaml 创建，每次不要去改 Nginx 了，直接改 yaml 然后创建/更新就行了；那么问题来了：”Nginx 该怎么处理？”

 

Ingress Controller 就是解决 “Nginx 的处理方式”；Ingress Controller 通过与 Kubernetes API 交互，动态的去感知集群中 Ingress 规则变化，然后读取它，按照它自己模板生成一段 Nginx 配置，再写到 Nginx Pod 里，最后 reload 一下。

 

Ingress controller的作用就是实时感知Ingress路由规则集合的变化，再与Api Server交互，获取Service、Pod在集群中的 IP等信息，然后发送给反向代理web服务器，刷新其路由配置信息，这就是它的服务发现机制。工作流程如下图：

<img src="https://img2018.cnblogs.com/blog/1349539/201809/1349539-20180930094200688-1480474925.png" alt="img" style="zoom:67%;" />

 

ingress工作原理：

客户端请求到外部的负载均衡器externalLB把请求调度到一个NodePort类型的Service(ingress-nginx)上，然后nodePort类型的Service(ingress-nginx)又把它调度到内部的叫做ingress Controller的Pod上，ingress Ctroller根据ingress中的定义(主机名还是URL)，每一组主机名或者URL都对应后端的Pod资源，并且用Service分组(service>site1/<service>site2只是用来做分组用的)

 

注意：Ingress Controller不同于Deployment 控制器的是，Ingress控制器不直接运行为kube-controller-manager的一部分，它仅仅是Kubernetes集群的一个附件，类似于CoreDNS，需要在集群上单独部署。

 

## **二、如何创建Ingress资源**

 

Ingress资源时基于HTTP虚拟主机或URL的转发规则，需要强调的是，这是一条转发规则。它在资源配置清单中的spec字段中嵌套了rules、backend和tls等字段进行定义。如下示例中定义了一个Ingress资源，其包含了一个转发规则：将发往myapp.magedu.com的请求，代理给一个名字为myapp的Service资源。

```
apiVersion: extensions/v1beta1   
kind: Ingress    
metadata:      
  name: ingress-myapp  
  namespace: default   
  annotations:     
    kubernetes.io/ingress.class: "nginx"
spec:   
  rules: 
  - host: myapp.magedu.com  
   http:
     paths:    
     - path:    
       backend:  
         serviceName: myapp
         servicePort: 80 
```

Ingress 中的spec字段是Ingress资源的核心组成部分，主要包含以下3个字段：

* rules：用于定义当前Ingress资源的转发规则列表；由rules定义规则，或没有匹配到规则时，所有的流量会转发到由backend定义的默认后端。
* backend：默认的后端用于服务那些没有匹配到任何规则的请求；定义Ingress资源时，必须要定义backend或rules两者之一，该字段用于让负载均衡器指定一个全局默认的后端。
* tls：TLS配置，目前仅支持通过默认端口443提供服务，如果要配置指定的列表成员指向不同的主机，则需要通过SNI TLS扩展机制来支持该功能。

backend对象的定义由2个必要的字段组成：serviceName和servicePort，分别用于指定流量转发的后端目标Service资源名称和端口。

rules对象由一系列的配置的Ingress资源的host规则组成，这些host规则用于将一个主机上的某个URL映射到相关后端Service对象，其定义格式如下：

```
spec:
  rules:
  - hosts: <string>
  http:
    paths:
    - path:
      backend:
        serviceName: <string>
        servicePort: <string>
```

需要注意的是，.spec.rules.host属性值，目前暂不支持使用IP地址定义，也不支持IP:Port的格式，该字段留空，代表着通配所有主机名。

tls对象由2个内嵌的字段组成，仅在定义TLS主机的转发规则上使用。

* hosts包含于使用的TLS证书之内的主机名称字符串列表，因此，此处使用的主机名必须匹配 tlsSecret 中的名称。

* secretName：用于引用SSL会话的secret对象名称，在基于SNI实现多主机路由的场景中，此字段为可选。

 

## **三、Ingress资源类型**

 

Ingress的资源类型有以下4种：

* 1、单Service资源型Ingress
* 2、基于URL路径进行流量转发
* 3、基于主机名称的虚拟主机
* 4、TLS类型的Ingress资源

1、单Service资源型Ingress

暴露单个服务的方法有多种，如NodePort、LoadBanlancer等等，当然也可以使用Ingress来进行暴露单个服务，只需要为Ingress指定default backend即可，如下示例：

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
spec:
  backend:
    serviceName: my-svc
    servicePort: 80
```

Ingress控制器会为其分配一个IP地址接入请求流量，并将其转发至后端my-svc

 

## **四、Ingress Nginx部署**

 

使用Ingress功能步骤：

* 1、安装部署ingress controller Pod
* 2、部署ingress-nginx service
* 3、部署后端服务
* 4、部署ingress

从前面的描述我们知道，Ingress 可以使用 yaml 的方式进行创建，从而得知 Ingress 也是标准的 K8S 资源，其定义的方式，也可以使用 explain 进行查看：

 

**1、部署Ingress controller**

（1）在github上下载yaml文件，并创建部署

github ingress-nginx项目：https://github.com/kubernetes/ingress-nginx

ingress安装指南：https://kubernetes.github.io/ingress-nginx/deploy/

```
[root@master ~]# mkdir ingress-nginx && cd ingress-nginx
[root@master ingress-nginx]# wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
```

（2）创建ingress controller的pod

```
[root@master ingress-nginx]# kubectl apply -f mandatory.yaml
namespace/ingress-nginx created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
deployment.apps/nginx-ingress-controller created
```

此处可能会遇到一个问题，新版本的Kubernetes在安装部署中，需要从k8s.grc.io仓库中拉取所需镜像文件，但由于国内网络防火墙问题导致无法正常拉取。docker.io仓库对google的容器做了镜像，可以通过下列命令下拉取相关镜像：

```
[root@node01 ~]# docker pull mirrorgooglecontainers/defaultbackend-amd64:1.5
1.5: Pulling from mirrorgooglecontainers/defaultbackend-amd64
9ecb1e82bb4a: Pull complete 
Digest: sha256:d08e129315e2dd093abfc16283cee19eabc18ae6b7cb8c2e26cc26888c6fc56a
Status: Downloaded newer image for mirrorgooglecontainers/defaultbackend-amd64:1.5
 
[root@node01 ~]# docker tag mirrorgooglecontainers/defaultbackend-amd64:1.5 k8s.gcr.io/defaultbackend-amd64:1.5
[root@node01 ~]# docker image ls
REPOSITORY                  TAG         IMAGE ID      CREATED       SIZE
mirrorgooglecontainers/defaultbackend-amd64  1.5         b5af743e5984    34 hours ago    5.13MB
k8s.gcr.io/defaultbackend-amd64        1.5         b5af743e5984    34 hours ago    5.13MB

验证pod的状态
[root@master ingress-nginx]# kubectl get pod -n ingress-nginx -w 
NAME                   READY   STATUS  RESTARTS   AGE 
default-http-backend-7db7c45b69-gjrnl   1/1    Running   0     35s 
nginx-ingress-controller-6bd7c597cb-6pchv 1/1    Running   0     34s
```

**2、部署ingress-nginx service**

通过ingress-controller对外提供服务，现在还需要手动给ingress-controller建立一个service，接收集群外部流量。下载地址：

https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/service-nodeport.yaml

```
[root@master ingress-nginx]# kubectl apply -f service-nodeport.yaml
service/ingress-nginx created

# 查询svc
[root@master ingress-nginx]# kubectl get svc -n ingress-nginx
NAME      TYPE    CLUSTER-IP    EXTERNAL-IP  PORT(S)           AGE
ingress-nginx  NodePort  10.109.244.123  <none>    80:30080/TCP,443:30443/TCP  21s

# 查询svc的详细信息
[root@master ~]# kubectl describe svc ingress-nginx -n ingress-nginx
Name:             ingress-nginx
Namespace:        ingress-nginx
Labels:          app.kubernetes.io/name=ingress-nginx
              app.kubernetes.io/part-of=ingress-nginx
Annotations:       kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app.kubernetes.io/name":"ingress-nginx","app.kubernetes.io/part-of":"ingres...
Selector:         app.kubernetes.io/name=ingress-nginx,app.kubernetes.io/part-of=ingress-nginx
Type:           NodePort
IP:            10.111.143.90
Port:           http 80/TCP
TargetPort:        80/TCP
NodePort:         http 30080/TCP
Endpoints:        10.244.1.104:80
Port:           https 443/TCP
TargetPort:        443/TCP
NodePort:         https 30443/TCP
Endpoints:        10.244.1.104:443
Session Affinity:     None
External Traffic Policy: Cluster
Events:          <none>
```

**3、创建Ingress，代理到后端nginx服务**

3.1 准备后端pod和service

（1）编写yaml文件，并创建

创建3个nginx服务的pod，并创建一个service绑定

```
[root@master ingress]# vim deploy-damo.yaml
#创建service为myapp
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: default
spec:
  selector:
    app: myapp
    release: canary
  ports:
  - name: http
    targetPort: 80
    port: 80
---
#创建后端服务的pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-backend-pod
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      release: canary
  template:
    metadata:
      labels:
        app: myapp
        release: canary
    spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v2
        ports:
        - name: http
          containerPort: 80
[root@master ingress]# kubectl apply -f deploy-demo.yaml 
service/myapp created
deployment.apps/myapp-deploy created
```

（2）查看新建的后端服务

```
[root@master ~]# kubectl get svc
NAME     TYPE    CLUSTER-IP    EXTERNAL-IP  PORT(S)  AGE
kubernetes  ClusterIP  10.96.0.1    <none>    443/TCP  146d
myapp    ClusterIP  10.103.137.126  <none>    80/TCP  6s
[root@master ~]# kubectl get pods
NAME              READY   STATUS  RESTARTS  AGE
myapp-deploy-67f6f6b4dc-2vzjn  1/1    Running  0     14s
myapp-deploy-67f6f6b4dc-c7f76  1/1    Running  0     14s
myapp-deploy-67f6f6b4dc-x79hc  1/1    Running  0     14s
[root@master ~]# kubectl describe svc myapp
Name:       myapp
Namespace:     default
Labels:      <none>
Annotations:    kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"myapp","namespace":"default"},"spec":{"ports":[{"name":"http","port":80,"targe...
Selector:     app=myapp,release=canary
Type:       ClusterIP
IP:        10.103.137.126
Port:       http 80/TCP
TargetPort:    80/TCP
Endpoints:     10.244.1.102:80,10.244.1.103:80,10.244.2.109:80
Session Affinity: None
Events:      <none>
```


**4、部署ingress，绑定后端myapp服务**

（1）编写yaml文件，并创建

```
[root@master ingress]# vim ingress-myapp.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-myapp
  namespace: default
spec:
  rules:
  - host: myapp.magedu.com
    http:
      paths:
      - path:
         backend:
           serviceName: myapp #注此处必须要和后端pod的service的名称一致，否则会报503错误
           servicePort: 80  #注此处必须要和后端pod的service的端口一致，否则会报503错误
[root@master ingress]# kubectl apply -f ingress-myapp.yaml
ingress.extensions/ingress-myapp created
```


（2）查看ingress-myapp的详细信息

```
[root@master ~]# kubectl get ingress
NAME      HOSTS       ADDRESS  PORTS   AGE
ingress-myapp  myapp.magedu.com       80    140d
[root@master ~]# kubectl describe ingress ingress-myapp
Name:       ingress-myapp
Namespace:    default
Address:     
Default backend: default-http-backend:80 (<none>)
Rules:
 Host       Path Backends
----       ---- --------
 myapp.magedu.com 
            myapp:80 (<none>)
Annotations:
 kubectl.kubernetes.io/last-applied-configuration: {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{},"name":"ingress-myapp","namespace":"default"},"spec":{"rules":[{"host":"myapp.magedu.com","http":{"paths":[{"backend":{"serviceName":"myapp","servicePort":80},"path":null}]}}]}}

Events:
 Type  Reason Age  From           Message
----  ------ ---- ----           -------
 Normal CREATE 37s  nginx-ingress-controller Ingress default/ingress-myapp
```

（3）进入nginx-ingress-controller进行查看是否注入了nginx的配置

```
[root@master ingress]# kubectl exec -n ingress-nginx -it nginx-ingress-controller-6bd7c597cb-6pchv -- /bin/bash
www-data@nginx-ingress-controller-6bd7c597cb-6pchv:/etc/nginx$ cat nginx.conf
......
  ## start server myapp.magedu.com
  server {
    server_name myapp.magedu.com ;
    listen 80;
    set $proxy_upstream_name "-";
    location / {
      set $namespace   "default";
      set $ingress_name  "ingress-myapp";
      set $service_name  "myapp";
      set $service_port  "80";
      set $location_path "/";
      rewrite_by_lua_block {
        balancer.rewrite()
      }
      log_by_lua_block {
        balancer.log()
        monitor.call()
      }
......
```

**（4）在集群外，查询服务验证**

①可以先修改一下主机的hosts，因为不是公网域名

192.168.130.103 myapp.magedu.com

② 访问业务成功

![img](https://img2018.cnblogs.com/blog/1349539/201809/1349539-20180930120604221-1512400790.png)

 

## **五、增加tomcat服务**

 

4.1 编写tomcat的配置清单文件

```
[root@master ingress]# vim tomcat-deploy.yaml
apiVersion: v1
kind: Service
metadata:
  name: tomcat
  namespace: default
spec:
  selector:
    app: tomcat
    release: canary
  ports:
  - name: http
    targetPort: 8080
    port: 8080
  - name: ajp
    targetPort: 8009
    port: 8009
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deploy
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tomcat
      release: canary
  template:
    metadata:
      labels:
        app: tomcat
        release: canary
    spec:
      containers:
      - name: tomcat
        image: tomcat:8.5.34-jre8-alpine   
        #此镜像在dockerhub上进行下载，需要查看版本是否有变化，hub.docker.com
        ports:
        - name: http
          containerPort: 8080
          name: ajp
          containerPort: 8009
[root@master ingress]# kubectl apply -f tomcat-deploy.yaml
service/tomcat created
deployment.apps/tomcat-deploy created
```

（2）查询验证

```
[root@master ~]# kubectl get pods
NAME              READY   STATUS  RESTARTS  AGE
tomcat-deploy-97d6458c5-hrmrw  1/1    Running  0     1m
tomcat-deploy-97d6458c5-ngxxx  1/1    Running  0     1m
tomcat-deploy-97d6458c5-xchgn  1/1    Running  0     1m
[root@master ~]# kubectl get svc
NAME     TYPE    CLUSTER-IP    EXTERNAL-IP  PORT(S)       AGE
kubernetes  ClusterIP  10.96.0.1    <none>    443/TCP       146d
tomcat    ClusterIP  10.98.193.252  <none>    8080/TCP,8009/TCP  1m
```

4.2 创建tomcat的ingress规则，绑定后端tomcat服务

```
[root@master ingress]# vim ingress-tomcat.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: tomcat
  namespace: default
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: tomcat.magedu.com    #主机域名
    http:
      paths:
      - path:
        backend:
          serviceName: tomcat
          servicePort: 8080
[root@master ingress]# kubectl apply -f ingress-tomcat.yaml
ingress.extensions/ingress-tomcat created
```

（2）查看ingress具体信息

```
[root@master ~]# kubectl get ingress
NAME       HOSTS       ADDRESS  PORTS   AGE
ingress-myapp  myapp.magedu.com       80    17m
ingress-tomcat  tomcat.magedu.com       80    6s
[root@master ~]# kubectl describe ingress ingress-tomcat
Name:       ingress-tomcat
Namespace:    default
Address:     
Default backend: default-http-backend:80 (<none>)
Rules:
 Host       Path Backends
----       ---- --------
 tomcat.magedu.com 
             tomcat:8080 (<none>)
Annotations:

 kubectl.kubernetes.io/last-applied-configuration: {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{},"name":"ingress-tomcat","namespace":"default"},"spec":{"rules":[{"host":"tomcat.magedu.com","http":{"paths":[{"backend":{"serviceName":"tomcat","servicePort":8080},"path":null}]}}]}}

Events:
 Type  Reason Age  From           Message
----  ------ ---- ----           -------
 Normal CREATE 17s  nginx-ingress-controller Ingress default/ingress-tomcat
```


（3）在集群外，查询服务验证

① 可以先修改一下主机的hosts，因为不是公网域名

192.168.130.103 tomcat.magedu.com

② 测试访问：tomcat.mageud.com:30080
![1349539-20180930155902614-1149294054-2](/Users/jinhuaiwang/Documents/k8s-onenote/basic-word/picture/1349539-20180930155902614-1149294054-2.png)

（4）总结

从前面的部署过程中，可以再次进行总结部署的流程如下：

①下载Ingress-controller相关的YAML文件，并给Ingress-controller创建独立的名称空间；

②部署后端的服务，如myapp，并通过service进行暴露；

③部署Ingress-controller的service，以实现接入集群外部流量；

④部署Ingress，进行定义规则，使Ingress-controller和后端服务的Pod组进行关联。

本次部署后的说明图如下：
![img](https://img2018.cnblogs.com/blog/1349539/201809/1349539-20180930160336958-1687092087.png)



## 六、构建TLS站点

（1）准备证书

```
[root@k8s-master ingress]# openssl genrsa -out tls.key 2048 
Generating RSA private key, 2048 bit long modulus
.......+++
.......................+++
e is 65537 (0x10001)

[root@k8s-master ingress]# openssl req -new -x509 -key tls.key -out tls.crt -subj /C=CN/ST=Beijing/L=Beijing/O=DevOps/CN=tomcat.magedu.com
```

（2）生成secret

```
[root@k8s-master ingress]# kubectl create secret tls tomcat-ingress-secret --cert=tls.crt --key=tls.key
secret/tomcat-ingress-secret created
[root@k8s-master ingress]# kubectl get secret
NAME                    TYPE                                  DATA      AGE
default-token-j5pf5     kubernetes.io/service-account-token   3         39d
tomcat-ingress-secret   kubernetes.io/tls                     2         9s
[root@k8s-master ingress]# kubectl describe secret tomcat-ingress-secret
Name:         tomcat-ingress-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/tls

Data
====
tls.crt:  1294 bytes
tls.key:  1679 bytes
```

（3）创建ingress

```
[root@k8s-master ingress]# kubectl explain ingress.spec
[root@k8s-master ingress]# kubectl explain ingress.spec.tls
[root@k8s-master ingress]# cp ingress-tomcat.yaml ingress-tomcat-tls.yaml
[root@k8s-master ingress]# vim ingress-tomcat-tls.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-tomcat-tls
  namespace: default
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:
  - hosts:
    - tomcat.magedu.com
    secretName: tomcat-ingress-secret
  rules:
  - host: tomcat.magedu.com
    http:
      paths:
      - path:
        backend:
          serviceName: tomcat
          servicePort: 8080

[root@k8s-master ingress]# kubectl apply -f ingress-tomcat-tls.yaml 
ingress.extensions/ingress-tomcat-tls created
[root@k8s-master ingress]# kubectl get ingress
NAME                 HOSTS               ADDRESS   PORTS     AGE
ingress-myapp        myapp.magedu.com              80        4h
ingress-tomcat-tls   tomcat.magedu.com             80, 443   5s
tomcat               tomcat.magedu.com             80        1h
[root@k8s-master ingress]# kubectl describe ingress ingress-tomcat-tls
Name:             ingress-tomcat-tls
Namespace:        default
Address:          
Default backend:  default-http-backend:80 (<none>)
TLS:
  tomcat-ingress-secret terminates tomcat.magedu.com
Rules:
  Host               Path  Backends
  ----               ----  --------
  tomcat.magedu.com  
                        tomcat:8080 (<none>)
Annotations:
  kubectl.kubernetes.io/last-applied-configuration:  {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{"kubernetes.io/ingress.class":"nginx"},"name":"ingress-tomcat-tls","namespace":"default"},"spec":{"rules":[{"host":"tomcat.magedu.com","http":{"paths":[{"backend":{"serviceName":"tomcat","servicePort":8080},"path":null}]}}],"tls":[{"hosts":["tomcat.magedu.com"],"secretName":"tomcat-ingress-secret"}]}}

  kubernetes.io/ingress.class:  nginx
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  CREATE  20s   nginx-ingress-controller  Ingress default/ingress-tomcat-tls
```

（4）访问测试：[https://tomcat.magedu.com:30443](https://tomcat.magedu.com:30443/)

![img](https://img2018.cnblogs.com/blog/1349539/201809/1349539-20180930174510553-912620418.png)





# 存储卷

## 一、存储卷的概念和类型

为了保证数据的持久性，必须保证数据在外部存储在`docker`容器中，为了实现数据的持久性存储，在宿主机和容器内做映射，可以保证在容器的生命周期结束，数据依旧可以实现持久性存储。但是在`k8s`中，由于`pod`分布在各个不同的节点之上，并不能实现不同节点之间持久性数据的共享，并且，在节点故障时，可能会导致数据的永久性丢失。为此，`k8s`就引入了外部存储卷的功能。
k8s的存储卷类型：

```
[root@k8s-master ~]# kubectl explain pod.spec.volumes #查看k8s支持的存储类型
KIND:     Pod
VERSION:  v1

常用分类：
emptyDir（临时目录）:Pod删除，数据也会被清除，这种存储成为emptyDir，用于数据的临时存储。
hostPath(宿主机目录映射):
本地的SAN(iSCSI,FC)、NAS(nfs,cifs,http)存储
分布式存储（glusterfs，rbd，cephfs）
云存储（EBS，Azure Disk）
```

persistentVolumeClaim -->PVC(存储卷创建申请)
当你需要创建一个存储卷时，只需要进行申请对应的存储空间即可使用，这就是PVC。其关联关系如图：

![img](https://img2018.cnblogs.com/blog/1349539/201810/1349539-20181009150238161-243800675.png)

上图解析：在Pod上定义一个PVC，该PVC要关联到当前名称空间的PVC资源，该PVC只是一个申请，PVC需要和PV进行关联。PV属于存储上的一部分存储空间。但是该方案存在的问题是，我们无法知道用户是什么时候去创建Pod，也不知道创建Pod时定义多大的PVC，那么如何实现按需创建呢？？？

不需要PV层，把所有存储空间抽象出来，这一个抽象层称为存储类，当用户创建PVC需要用到PV时，可以向存储类申请对应的存储空间，存储类会按照需求创建对应的存储空间，这就是PV的动态供给，如图：

![img](https://img2018.cnblogs.com/blog/1349539/201810/1349539-20181009150251762-2029671134.png)

那么PV的动态供给，其重点是在存储类的定义，其分类大概是对存储的性能进行分类的，如图：金存储类、银存储类、铜存储类等。

![img](https://img2018.cnblogs.com/blog/1349539/201810/1349539-20181009150328995-1548599425.png)总结：
k8s要使用存储卷，需要2步：
1、在pod定义volume，并指明关联到哪个存储设备
2、在容器使用volume mount进行挂载

## 二、emptyDir存储卷演示

> 一个emptyDir 第一次创建是在一个pod被指定到具体node的时候，并且会一直存在在pod的生命周期当中，正如它的名字一样，它初始化是一个空的目录，pod中的容器都可以读写这个目录，这个目录可以被挂在到各个容器相同或者不相同的的路径下。当一个pod因为任何原因被移除的时候，这些数据会被永久删除。注意：一个容器崩溃了不会导致数据的丢失，因为容器的崩溃并不移除pod.

> emptyDir 磁盘的作用：
> （1）普通空间，基于磁盘的数据存储
> （2）作为从崩溃中恢复的备份点
> （3）存储那些那些需要长久保存的数据，例web服务中的数据
> 默认的，emptyDir 磁盘会存储在主机所使用的媒介上，可能是SSD，或者网络硬盘，这主要取决于你的环境。当然，我们也可以将emptyDir.medium的值设置为Memory来告诉Kubernetes 来挂在一个基于内存的目录tmpfs，因为
> tmpfs速度会比硬盘块度了，但是，当主机重启的时候所有的数据都会丢失。

```
[root@k8s-master ~]# kubectl explain pods.spec.volumes.emptyDir  #查看emptyDir存储定义
[root@k8s-master ~]# kubectl explain pods.spec.containers.volumeMounts  #查看容器挂载方式
[root@k8s-master ~]# cd mainfests && mkdir volumes && cd volumes
[root@k8s-master volumes]# cp ../pod-demo.yaml ./
[root@k8s-master volumes]# mv pod-demo.yaml pod-vol-demo.yaml
[root@k8s-master volumes]# vim pod-vol-demo.yaml   #创建emptyDir的清单
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
  namespace: default
  labels:
    app: myapp
    tier: frontend
  annotations:
    magedu.com/create-by:"cluster admin"
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
    imagePullPolicy: IfNotPresent
    ports:
    - name: http
      containerPort: 80
    volumeMounts:    #在容器内定义挂载存储名称和挂载路径
    - name: html
      mountPath: /usr/share/nginx/html/
  - name: busybox
    image: busybox:latest
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: html
      mountPath: /data/    #在容器内定义挂载存储名称和挂载路径
    command: ['/bin/sh','-c','while true;do echo $(date) >> /data/index.html;sleep 2;done']
  volumes:  #定义存储卷
  - name: html    #定义存储卷名称  
    emptyDir: {}  #定义存储卷类型
[root@k8s-master volumes]# kubectl apply -f pod-vol-demo.yaml 
pod/pod-vol-demo created 
[root@k8s-master volumes]# kubectl get pods
NAME                                 READY     STATUS    RESTARTS   AGE
pod-vol-demo                         2/2       Running   0          27s
[root@k8s-master volumes]# kubectl get pods -o wide
NAME                      READY     STATUS    RESTARTS   AGE       IP            NODE
......
pod-vol-demo              2/2       Running   0          16s       10.244.2.34   k8s-node02
......

在上面，我们定义了2个容器，其中一个容器是输入日期到index.html中，然后验证访问nginx的html是否可以获取日期。以验证两个容器之间挂载的emptyDir实现共享。如下访问验证:
[root@k8s-master volumes]# curl 10.244.2.34  #访问验证
Tue Oct 9 03:56:53 UTC 2018
Tue Oct 9 03:56:55 UTC 2018
Tue Oct 9 03:56:57 UTC 2018
Tue Oct 9 03:56:59 UTC 2018
Tue Oct 9 03:57:01 UTC 2018
Tue Oct 9 03:57:03 UTC 2018
Tue Oct 9 03:57:05 UTC 2018
Tue Oct 9 03:57:07 UTC 2018
Tue Oct 9 03:57:09 UTC 2018
Tue Oct 9 03:57:11 UTC 2018
Tue Oct 9 03:57:13 UTC 2018
Tue Oct 9 03:57:15 UTC 2018
```

## 三、hostPath存储卷演示

hostPath宿主机路径，就是把pod所在的宿主机之上的脱离pod中的容器名称空间的之外的宿主机的文件系统的某一目录和pod建立关联关系，在pod删除时，存储数据不会丢失。

```
（1）查看hostPath存储类型定义
[root@k8s-master volumes]# kubectl explain pods.spec.volumes.hostPath  
KIND:     Pod
VERSION:  v1

RESOURCE: hostPath <Object>

DESCRIPTION:
     HostPath represents a pre-existing file or directory on the host machine
     that is directly exposed to the container. This is generally used for
     system agents or other privileged things that are allowed to see the host
     machine. Most containers will NOT need this. More info:
     https://kubernetes.io/docs/concepts/storage/volumes#hostpath

     Represents a host path mapped into a pod. Host path volumes do not support
     ownership management or SELinux relabeling.

FIELDS:
   path	<string> -required-  #指定宿主机的路径
     Path of the directory on the host. If the path is a symlink, it will follow
     the link to the real path. More info:
     https://kubernetes.io/docs/concepts/storage/volumes#hostpath

   type	<string>
     Type for HostPath Volume Defaults to "" More info:
     https://kubernetes.io/docs/concepts/storage/volumes#hostpath

type：
DirectoryOrCreate  宿主机上不存在创建此目录  
Directory 必须存在挂载目录  
FileOrCreate 宿主机上不存在挂载文件就创建  
File 必须存在文件  

（2）清单定义
[root@k8s-master volumes]# vim pod-hostpath-vol.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-vol-hostpath
  namespace: default
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  volumes:
    - name: html
      hostPath:
        path: /data/pod/volume1
        type: DirectoryOrCreate

（3）在node节点上创建挂载目录
[root@k8s-node01 ~]# mkdir -p /data/pod/volume1
[root@k8s-node01 ~]# vim /data/pod/volume1/index.html
node01.magedu.com
[root@k8s-node02 ~]# mkdir -p /data/pod/volume1
[root@k8s-node02 ~]# vim /data/pod/volume1/index.html
node02.magedu.com
[root@k8s-master volumes]# kubectl apply -f pod-hostpath-vol.yaml 
pod/pod-vol-hostpath created

（4）访问测试
[root@k8s-master volumes]# kubectl get pods -o wide
NAME                                 READY     STATUS    RESTARTS   AGE       IP            NODE
......
pod-vol-hostpath                     1/1       Running   0          37s       10.244.2.35   k8s-node02
......
[root@k8s-master volumes]# curl 10.244.2.35
node02.magedu.com
[root@k8s-master volumes]# kubectl delete -f pod-hostpath-vol.yaml  #删除pod，再重建，验证是否依旧可以访问原来的内容
[root@k8s-master volumes]# kubectl apply -f pod-hostpath-vol.yaml 
pod/pod-vol-hostpath created
[root@k8s-master volumes]# curl  10.244.2.37 
node02.magedu.com
```

hostPath可以实现持久存储，但是在node节点故障时，也会导致数据的丢失

## 四、nfs共享存储卷演示

nfs使的我们可以挂在已经存在的共享到的我们的Pod中，和emptyDir不同的是，emptyDir会被删除当我们的Pod被删除的时候，但是nfs不会被删除，仅仅是解除挂在状态而已，这就意味着NFS能够允许我们提前对数据进行处理，而且这些数据可以在Pod之间相互传递.并且，nfs可以同时被多个pod挂在并进行读写

注意：必须先报纸NFS服务器正常运行在我们进行挂在nfs的时候

```
（1）在stor01节点上安装nfs，并配置nfs服务
[root@stor01 ~]# yum install -y nfs-utils  ==》192.168.56.14
[root@stor01 ~]# mkdir /data/volumes -pv
[root@stor01 ~]# vim /etc/exports
/data/volumes 192.168.56.0/24(rw,no_root_squash)
[root@stor01 ~]# systemctl start nfs
[root@stor01 ~]# showmount -e
Export list for stor01:
/data/volumes 192.168.56.0/24

（2）在node01和node02节点上安装nfs-utils，并测试挂载
[root@k8s-node01 ~]# yum install -y nfs-utils
[root@k8s-node02 ~]# yum install -y nfs-utils
[root@k8s-node02 ~]# mount -t nfs stor01:/data/volumes /mnt
[root@k8s-node02 ~]# mount
......
stor01:/data/volumes on /mnt type nfs4 (rw,relatime,vers=4.1,rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,port=0,timeo=600,retrans=2,sec=sys,clientaddr=192.168.56.13,local_lock=none,addr=192.168.56.14)
[root@k8s-node02 ~]# umount /mnt/

（3）创建nfs存储卷的使用清单
[root@k8s-master volumes]# cp pod-hostpath-vol.yaml pod-nfs-vol.yaml
[root@k8s-master volumes]# vim pod-nfs-vol.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-vol-nfs
  namespace: default
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  volumes:
    - name: html
      nfs:
        path: /data/volumes
        server: stor01
[root@k8s-master volumes]# kubectl apply -f pod-nfs-vol.yaml 
pod/pod-vol-nfs created
[root@k8s-master volumes]# kubectl get pods -o wide
NAME                     READY     STATUS    RESTARTS   AGE       IP            NODE
pod-vol-nfs              1/1       Running   0          21s       10.244.2.38   k8s-node02

（3）在nfs服务器上创建index.html
[root@stor01 ~]# cd /data/volumes
[root@stor01 volumes ~]# vim index.html
<h1> nfs stor01</h1>
[root@k8s-master volumes]# curl 10.244.2.38
<h1> nfs stor01</h1>
[root@k8s-master volumes]# kubectl delete -f pod-nfs-vol.yaml   #删除nfs相关pod，再重新创建，可以得到数据的持久化存储
pod "pod-vol-nfs" deleted
[root@k8s-master volumes]# kubectl apply -f pod-nfs-vol.yaml 
```

## 五、PVC和PV的概念

前面提到kubernetes提供那么多存储接口，但是首先kubernetes的各个Node节点能管理这些存储，但是各种存储参数也需要专业的存储工程师才能了解，由此我们的kubernetes管理变的更加复杂的。由此kubernetes提出了PV和PVC的概念，这样开发人员和使用者就不需要关注后端存储是什么，使用什么参数等问题。如下图：

<img src="https://img2018.cnblogs.com/blog/1349539/201810/1349539-20181010111022629-145101656.png" alt="img" style="zoom:70%;" />

PersistentVolume（PV）是集群中已由管理员配置的一段网络存储。 集群中的资源就像一个节点是一个集群资源。 PV是诸如卷之类的卷插件，但是具有独立于使用PV的任何单个pod的生命周期。 该API对象捕获存储的实现细节，即NFS，iSCSI或云提供商特定的存储系统。

<img src="https://img2018.cnblogs.com/blog/1349539/201810/1349539-20181010111035655-819686452.png" alt="img" align="center" style="zoom:100%;" />



PersistentVolumeClaim（PVC）是用户存储的请求。PVC的使用逻辑：在pod中定义一个存储卷（该存储卷类型为PVC），定义的时候直接指定大小，pvc必须与对应的pv建立关系，pvc会根据定义去pv申请，而pv是由存储空间创建出来的。pv和pvc是kubernetes抽象出来的一种存储资源。

虽然PersistentVolumeClaims允许用户使用抽象存储资源，但是常见的需求是，用户需要根据不同的需求去创建PV，用于不同的场景。而此时需要集群管理员提供不同需求的PV，而不仅仅是PV的大小和访问模式，但又不需要用户了解这些卷的实现细节。 对于这样的需求，此时可以采用StorageClass资源。这个在前面就已经提到过此方案。

PV是集群中的资源。 PVC是对这些资源的请求，也是对资源的索赔检查。 PV和PVC之间的相互作用遵循这个生命周期：

```
Provisioning（配置）---> Binding（绑定）--->Using（使用）---> Releasing（释放） ---> Recycling（回收）
```

#### Provisioning

> 这里有两种PV的提供方式:静态或者动态
>
> 静态-->直接固定存储空间：
>     集群管理员创建一些 PV。它们携带可供集群用户使用的真实存储的详细信息。 它们存在于Kubernetes API中，可用于消费。

> 动态-->通过存储类进行动态创建存储空间：
>     当管理员创建的静态 PV 都不匹配用户的 PVC 时，集群可能会尝试动态地为 PVC 配置卷。此配置基于 StorageClasses：PVC 必须请求存储类，并且管理员必须已创建并配置该类才能进行动态配置。 要求该类的声明有效地为自己禁用动态配置。

#### Binding

> 在动态配置的情况下，用户创建或已经创建了具有特定数量的存储请求和特定访问模式的PersistentVolumeClaim。 主机中的控制回路监视新的PVC，找到匹配的PV（如果可能），并将 PVC 和 PV 绑定在一起。 如果为新的PVC动态配置PV，则循环将始终将该PV绑定到PVC。 否则，用户总是至少得到他们要求的内容，但是卷可能超出了要求。 一旦绑定，PersistentVolumeClaim绑定是排他的，不管用于绑定它们的模式。

> 如果匹配的卷不存在，PVC将保持无限期。 随着匹配卷变得可用，PVC将被绑定。 例如，提供许多50Gi PV的集群将不匹配要求100Gi的PVC。 当集群中添加100Gi PV时，可以绑定PVC。

#### Using

> Pod使用PVC作为卷。 集群检查声明以找到绑定的卷并挂载该卷的卷。 对于支持多种访问模式的卷，用户在将其声明用作pod中的卷时指定所需的模式。

> 一旦用户有声明并且该声明被绑定，绑定的PV属于用户，只要他们需要它。 用户通过在其Pod的卷块中包含PersistentVolumeClaim来安排Pods并访问其声明的PV。

#### Releasing

> 当用户完成卷时，他们可以从允许资源回收的API中删除PVC对象。 当声明被删除时，卷被认为是“释放的”，但是它还不能用于另一个声明。 以前的索赔人的数据仍然保留在必须根据政策处理的卷上.

#### Reclaiming

> PersistentVolume的回收策略告诉集群在释放其声明后，该卷应该如何处理。 目前，卷可以是保留，回收或删除。 保留可以手动回收资源。 对于那些支持它的卷插件，删除将从Kubernetes中删除PersistentVolume对象，以及删除外部基础架构（如AWS EBS，GCE PD，Azure Disk或Cinder卷）中关联的存储资产。 动态配置的卷始终被删除

#### Recycling

> 如果受适当的卷插件支持，回收将对卷执行基本的擦除（rm -rf / thevolume / *），并使其再次可用于新的声明。

## 六、NFS使用PV和PVC

实验图如下：

<img src="https://img2018.cnblogs.com/blog/1349539/201810/1349539-20181010110937451-1076717702.png" alt="img" style="zoom:67%;" />

```
[root@k8s-master ~]# kubectl explain pv    #查看pv的定义方式
FIELDS:
	apiVersion
	kind
	metadata
	spec
[root@k8s-master ~]# kubectl explain pv.spec    #查看pv定义的规格
spec:
  nfs（定义存储类型）
    path（定义挂载卷路径）
    server（定义服务器名称）
  accessModes（定义访问模型，有以下三种访问模型，以列表的方式存在，也就是说可以定义多个访问模式）
    ReadWriteOnce（RWO）  单节点读写
	ReadOnlyMany（ROX）  多节点只读
	ReadWriteMany（RWX）  多节点读写
  capacity（定义PV空间的大小）
    storage（指定大小）

[root@k8s-master volumes]# kubectl explain pvc   #查看PVC的定义方式
KIND:     PersistentVolumeClaim
VERSION:  v1
FIELDS:
   apiVersion	<string>
   kind	<string>  
   metadata	<Object>
   spec	<Object>
[root@k8s-master volumes]# kubectl explain pvc.spec
spec:
  accessModes（定义访问模式，必须是PV的访问模式的子集）
  resources（定义申请资源的大小）
    requests:
      storage: 
```

### 1、配置nfs存储

```
[root@stor01 volumes]# mkdir v{1,2,3,4,5}
[root@stor01 volumes]# vim /etc/exports
/data/volumes/v1 192.168.56.0/24(rw,no_root_squash)
/data/volumes/v2 192.168.56.0/24(rw,no_root_squash)
/data/volumes/v3 192.168.56.0/24(rw,no_root_squash)
/data/volumes/v4 192.168.56.0/24(rw,no_root_squash)
/data/volumes/v5 192.168.56.0/24(rw,no_root_squash)
[root@stor01 volumes]# exportfs -arv
exporting 192.168.56.0/24:/data/volumes/v5
exporting 192.168.56.0/24:/data/volumes/v4
exporting 192.168.56.0/24:/data/volumes/v3
exporting 192.168.56.0/24:/data/volumes/v2
exporting 192.168.56.0/24:/data/volumes/v1
[root@stor01 volumes]# showmount -e
Export list for stor01:
/data/volumes/v5 192.168.56.0/24
/data/volumes/v4 192.168.56.0/24
/data/volumes/v3 192.168.56.0/24
/data/volumes/v2 192.168.56.0/24
/data/volumes/v1 192.168.56.0/24
```

### 2、定义PV

> 这里定义5个PV，并且定义挂载的路径以及访问模式，还有PV划分的大小。

```
[root@k8s-master volumes]# kubectl explain pv
[root@k8s-master volumes]# kubectl explain pv.spec.nfs
[root@k8s-master volumes]# vim pv-demo.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv001
  labels:
    name: pv001
spec:
  nfs:
    path: /data/volumes/v1
    server: stor01
  accessModes: ["ReadWriteMany","ReadWriteOnce"]
  capacity:
    storage: 1Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv002
  labels:
    name: pv002
spec:
  nfs:
    path: /data/volumes/v2
    server: stor01
  accessModes: ["ReadWriteOnce"]
  capacity:
    storage: 2Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv003
  labels:
    name: pv003
spec:
  nfs:
    path: /data/volumes/v3
    server: stor01
  accessModes: ["ReadWriteMany","ReadWriteOnce"]
  capacity:
    storage: 2Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv004
  labels:
    name: pv004
spec:
  nfs:
    path: /data/volumes/v4
    server: stor01
  accessModes: ["ReadWriteMany","ReadWriteOnce"]
  capacity:
    storage: 4Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv005
  labels:
    name: pv005
spec:
  nfs:
    path: /data/volumes/v5
    server: stor01
  accessModes: ["ReadWriteMany","ReadWriteOnce"]
  capacity:
    storage: 5Gi
[root@k8s-master volumes]# kubectl apply -f pv-demo.yaml 
persistentvolume/pv001 created
persistentvolume/pv002 created
persistentvolume/pv003 created
persistentvolume/pv004 created
persistentvolume/pv005 created
[root@k8s-master volumes]# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
pv001     1Gi        RWO,RWX        Retain           Available                                      7s
pv002     2Gi        RWO            Retain           Available                                      7s
pv003     2Gi        RWO,RWX        Retain           Available                                      7s
pv004     4Gi        RWO,RWX        Retain           Available                                      7s
pv005     5Gi        RWO,RWX        Retain           Available                                      7s
```

### 3、定义PVC

> 这里定义了pvc的访问模式为多路读写，该访问模式必须在前面pv定义的访问模式之中。定义PVC申请的大小为2Gi，此时PVC会自动去匹配多路读写且大小为2Gi的PV，匹配成功获取PVC的状态即为Bound

```
[root@k8s-master volumes ~]# vim pod-vol-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
  namespace: default
spec:
  accessModes: ["ReadWriteMany"]
  resources:
    requests:
      storage: 2Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-vol-pvc
  namespace: default
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  volumes:
    - name: html
      persistentVolumeClaim:
        claimName: mypvc
[root@k8s-master volumes]# kubectl apply -f pod-vol-pvc.yaml 
persistentvolumeclaim/mypvc created
pod/pod-vol-pvc created
[root@k8s-master volumes]# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM           STORAGECLASS   REASON    AGE
pv001     1Gi        RWO,RWX        Retain           Available                                            19m
pv002     2Gi        RWO            Retain           Available                                            19m
pv003     2Gi        RWO,RWX        Retain           Bound       default/mypvc                            19m
pv004     4Gi        RWO,RWX        Retain           Available                                            19m
pv005     5Gi        RWO,RWX        Retain           Available                                            19m
[root@k8s-master volumes]# kubectl get pvc
NAME      STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mypvc     Bound     pv003     2Gi        RWO,RWX                       22s
```

### 4、测试访问

> 在存储服务器上创建index.html，并写入数据，通过访问Pod进行查看，可以获取到相应的页面。

```
[root@stor01 volumes]# cd v3/
[root@stor01 v3]# echo "welcome to use pv3" > index.html
[root@k8s-master volumes]# kubectl get pods -o wide
pod-vol-pvc             1/1       Running   0          3m        10.244.2.39   k8s-node02
[root@k8s-master volumes]# curl  10.244.2.39
welcome to use pv3
```

## 七、StorageClass

在pv和pvc使用过程中存在的问题，在pvc申请存储空间时，未必就有现成的pv符合pvc申请的需求，上面nfs在做pvc可以成功的因素是因为我们做了指定的需求处理。那么当PVC申请的存储空间不一定有满足PVC要求的PV事，又该如何处理呢？？？为此，Kubernetes为管理员提供了描述存储"class（类）"的方法（StorageClass）。举个例子，在存储系统中划分一个1TB的存储空间提供给Kubernetes使用，当用户需要一个10G的PVC时，会立即通过restful发送请求，从而让存储空间创建一个10G的image，之后在我们的集群中定义成10G的PV供给给当前的PVC作为挂载使用。在此之前我们的存储系统必须支持restful接口，比如ceph分布式存储，而glusterfs则需要借助第三方接口完成这样的请求。如图：

<img src="https://img2018.cnblogs.com/blog/1349539/201810/1349539-20181010160310479-848300996.png" alt="img" style="zoom: 67%;" />



```
[root@k8s-master ~]# kubectl explain storageclass  #storageclass也是k8s上的资源
KIND:     StorageClass
VERSION:  storage.k8s.io/v1
FIELDS:
   allowVolumeExpansion	<boolean>     
   allowedTopologies	<[]Object>   
   apiVersion	<string>   
   kind	<string>     
   metadata	<Object>     
   mountOptions	<[]string>    #挂载选项
   parameters	<map[string]string>  #参数，取决于分配器，可以接受不同的参数。 例如，参数 type 的值 io1 和参数 iopsPerGB 特定于 EBS PV。当参数被省略时，会使用默认值。  
   provisioner	<string> -required-  #存储分配器，用来决定使用哪个卷插件分配 PV。该字段必须指定。
   reclaimPolicy	<string>   #回收策略，可以是 Delete 或者 Retain。如果 StorageClass 对象被创建时没有指定 reclaimPolicy ，它将默认为 Delete。 
   volumeBindingMode	<string>  #卷的绑定模式
```

> StorageClass 中包含 provisioner、parameters 和 reclaimPolicy 字段，当 class 需要动态分配 PersistentVolume 时会使用到。由于StorageClass需要一个独立的存储系统，此处就不再演示。从其他资料查看定义StorageClass的方式如下：

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
mountOptions:
  - debug
```



# Secret和configMap

在日常单机甚至集群状态下，我们需要对一个应用进行配置，只需要修改其配置文件即可。那么在容器中又该如何提供配置 信息呢？？？例如，为Nginx配置一个指定的server_name或worker进程数，为Tomcat的JVM配置其堆内存大小。传统的实践过程中通常有以下几种方式：

> - 启动容器时，通过命令传递参数
> - 将定义好的配置文件通过镜像文件进行写入
> - 通过环境变量的方式传递配置数据
> - 挂载Docker卷传送配置文件
>   而在Kubernetes系统之中也存在这样的组件，就是特殊的存储卷类型。其并不是提供pod存储空间，而是给管理员或用户提供从集群外部向Pod内部的应用注入配置信息的方式。这两种特殊类型的存储卷分别是：configMap和secret

* Secret：用于向Pod传递敏感信息，比如密码，私钥，证书文件等，这些信息如果在容器中定义容易泄露，Secret资源可以让用户将这些信息存储在急群众，然后通过Pod进行挂载，实现敏感数据和系统解耦的效果。

* ConfigMap：主要用于向Pod注入非敏感数据，使用时，用户将数据直接存储在ConfigMap对象当中，然后Pod通过使用ConfigMap卷进行引用，实现容器的配置文件集中定义和管理。

```
[root@k8s-master volumes]# kubectl explain pods.spec.volumes
......
   configMap	<Object>
     ConfigMap represents a configMap that should populate this volume

   secret	<Object>
     Secret represents a secret that should populate this volume. More info:
     https://kubernetes.io/docs/concepts/storage/volumes#secret
```

## 一、Secret解析

Secret对象存储数据的方式是以键值方式存储数据，在Pod资源进行调用Secret的方式是通过环境变量或者存储卷的方式进行访问数据，解决了密码、token、密钥等敏感数据的配置问题，而不需要把这些敏感数据暴露到镜像或者Pod Spec中。另外，Secret对象的数据存储和打印格式为Base64编码的字符串，因此用户在创建Secret对象时，也需要提供该类型的编码格式的数据。在容器中以环境变量或存储卷的方式访问时，会自动解码为明文格式。需要注意的是，如果是在Master节点上，Secret对象以非加密的格式存储在etcd中，所以需要对etcd的管理和权限进行严格控制。

> Secret有4种类型：

> - Service Account ：用来访问Kubernetes API，由Kubernetes自动创建，并且会自动挂载到Pod的/run/secrets/kubernetes.io/serviceaccount目录中；
> - Opaque ：base64编码格式的Secret，用来存储密码、密钥、信息、证书等，类型标识符为generic；
> - kubernetes.io/dockerconfigjson ：用来存储私有docker registry的认证信息，类型标识为docker-registry。
> - kubernetes.io/tls：用于为SSL通信模式存储证书和私钥文件，命令式创建类型标识为tls。

### 创建 Secret的2种方式

**命令式创建**

> - 1、通过 --from-literal：

```
[root@k8s-master ~]# kubectl create secret -h
Create a secret using specified subcommand.

Available Commands:
  docker-registry Create a secret for use with a Docker registry
  generic         Create a secret from a local file, directory or literal value
  tls             Create a TLS secret

Usage:
  kubectl create secret [flags] [options]

Use "kubectl <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands).

每个 --from-literal 对应一个信息条目。
[root@k8s-master ~]# kubectl create secret generic mysecret --from-literal=username=admin --from-literal=password=123456
secret/mysecret created
[root@k8s-master ~]# kubectl get secret
NAME                    TYPE                                  DATA      AGE
mysecret                Opaque                                2         6s
```

> - 2、通过 --from-file：
>   每个文件内容对应一个信息条目。

```
[root@k8s-master ~]# echo -n admin > ./username
[root@k8s-master ~]# echo -n 123456 > ./password
[root@k8s-master ~]# kubectl create secret generic mysecret --from-file=./username --from-file=./password 
secret/mysecret created
[root@k8s-master ~]# kubectl get secret
NAME                    TYPE                                  DATA      AGE
mysecret                Opaque                                2         6s
```

> - 3、通过 --from-env-file：
>   文件 env.txt 中每行 Key=Value 对应一个信息条目。

```
[root@k8s-master ~]# cat << EOF > env.txt
> username=admin
> password=123456
> EOF
[root@k8s-master ~]# kubectl create secret generic mysecret --from-env-file=env.txt 
secret/mysecret created
[root@k8s-master ~]# kubectl get secret
NAME                    TYPE                                  DATA      AGE
mysecret                Opaque                                2         10s
```

**清单式创建**

> - 通过 YAML 配置文件：

```
#事先完成敏感数据的Base64编码
[root@k8s-master ~]# echo -n admin |base64
YWRtaW4=
[root@k8s-master ~]# echo -n 123456 |base64
MTIzNDU2

[root@k8s-master ~]# vim secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
data:
  username: YWRtaW4=
  password: MTIzNDU2
[root@k8s-master ~]# kubectl apply -f secret.yaml 
secret/mysecret created
[root@k8s-master ~]# kubectl get secret  #查看存在的 secret，显示有2条数据
NAME                    TYPE                                  DATA      AGE
mysecret                Opaque                                2         8s
[root@k8s-master ~]# kubectl describe secret mysecret  #查看数据的 Key
Name:         mysecret
Namespace:    default
Labels:       <none>
Annotations:  
Type:         Opaque

Data
====
username:  5 bytes
password:  6 bytes
[root@k8s-master ~]# kubectl edit secret mysecret  #查看具体的value，可以使用该命令
apiVersion: v1
data:
  password: MTIzNDU2
  username: YWRtaW4=
kind: Secret
metadata:
......
[root@k8s-master ~]# echo -n MTIzNDU2 |base64 --decode  #通过 base64 将 Value 反编码：
123456
[root@k8s-master ~]# echo -n YWRtaW4= |base64 --decode
admin
```

### 如何使用Secret？？

> Pod 可以通过 Volume 或者环境变量的方式使用 Secret

```
[root@k8s-master volumes]# vim pod-secret-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret
spec:
  containers:
  - name: pod-secret
    image: busybox
    args:
      - /bin/sh
      - -c
      - sleep 10;touch /tmp/healthy;sleep 30000
    volumeMounts:   #将 foo mount 到容器路径 /etc/foo，可指定读写权限为 readOnly。
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:    #定义 volume foo，来源为 secret mysecret。
  - name: foo
    secret:
      secretName: mysecret
[root@k8s-master volumes]# kubectl apply -f pod-secret-demo.yaml 
pod/pod-secret created
[root@k8s-master volumes]# kubectl get pods
pod-secret                           1/1       Running   0          1m
[root@k8s-master volumes]# kubectl exec -it pod-secret sh
/ # ls /etc/foo/
password  username
/ # cat /etc/foo/username 
admin/ # 
/ # cat /etc/foo/password 
123456/ # 
```

> 可以看到，Kubernetes 会在指定的路径 /etc/foo 下为每条敏感数据创建一个文件，文件名就是数据条目的 Key，这里是 /etc/foo/username 和 /etc/foo/password，Value 则以明文存放在文件中。
> 也可以自定义存放数据的文件名，比如将配置文件改为：

```
[root@k8s-master volumes]# cat pod-secret-demo.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret
spec:
  containers:
  - name: pod-secret
    image: busybox
    args:
      - /bin/sh
      - -c
      - sleep 10;touch /tmp/healthy;sleep 30000
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      items:    #自定义存放数据的文件名
      - key: username
        path: my-secret/my-username
      - key: password
        path: my-secret/my-password
[root@k8s-master volumes]# kubectl delete pods pod-secret
pod "pod-secret" deleted
[root@k8s-master volumes]# kubectl apply -f pod-secret-demo.yaml 
pod/pod-secret created
[root@k8s-master volumes]# kubectl exec -it pod-secret sh
/ # cat /etc/foo/my-secret/my-username 
admin
/ # cat /etc/foo/my-secret/my-password 
123456
```

> 这时数据将分别存放在 /etc/foo/my-secret/my-username 和 /etc/foo/my-secret/my-password 中。

> 以 Volume 方式使用的 Secret 支持动态更新：Secret 更新后，容器中的数据也会更新。

> 将 password 更新为 abcdef，base64 编码为 YWJjZGVm

```
[root@k8s-master ~]# vim secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
data:
  username: YWRtaW4=
  password: YWJjZGVm
[root@k8s-master ~]# kubectl apply -f secret.yaml 
secret/mysecret configured
/ # cat /etc/foo/my-secret/my-password 
abcdef
```

> 通过 Volume 使用 Secret，容器必须从文件读取数据，会稍显麻烦，Kubernetes 还支持通过环境变量使用 Secret。

```
[root@k8s-master volumes]# cp pod-secret-demo.yaml pod-secret-env-demo.yaml
[root@k8s-master volumes]# vim pod-secret-env-demo.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-env
spec:
  containers:
  - name: pod-secret-env
    image: busybox
    args:
      - /bin/sh
      - -c
      - sleep 10;touch /tmp/healthy;sleep 30000
    env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
[root@k8s-master volumes]# kubectl apply -f pod-secret-env-demo.yaml 
pod/pod-secret-env created

[root@k8s-master volumes]# kubectl exec -it pod-secret-env sh
/ # echo $SECRET_USERNAME
admin
/ # echo $SECRET_PASSWORD
abcdef
```

通过环境变量 SECRET_USERNAME 和 SECRET_PASSWORD 成功读取到 Secret 的数据。
需要注意的是，环境变量读取 Secret 很方便，但无法支撑 Secret 动态更新。
Secret 可以为 Pod 提供密码、Token、私钥等敏感数据；对于一些非敏感数据，比如应用的配置信息，则可以用 ConfigMap。

## 二、ConifgMap解析

configmap是让配置文件从镜像中解耦，让镜像的可移植性和可复制性。许多应用程序会从配置文件、命令行参数或环境变量中读取配置信息。这些配置信息需要与docker image解耦，你总不能每修改一个配置就重做一个image吧？ConfigMap API给我们提供了向容器中注入配置信息的机制，ConfigMap可以被用来保存单个属性，也可以用来保存整个配置文件或者JSON二进制大对象。

ConfigMap API资源用来保存key-value pair配置数据，这个数据可以在pods里使用，或者被用来为像controller一样的系统组件存储配置数据。虽然ConfigMap跟Secrets类似，但是ConfigMap更方便的处理不含敏感信息的字符串。 注意：ConfigMaps不是属性配置文件的替代品。ConfigMaps只是作为多个properties文件的引用。可以把它理解为Linux系统中的/etc目录，专门用来存储配置文件的目录。下面举个例子，使用ConfigMap配置来创建Kuberntes Volumes，ConfigMap中的每个data项都会成为一个新文件。

```
[root@k8s-master volumes]# kubectl explain cm
KIND:     ConfigMap
VERSION:  v1
FIELDS:
   apiVersion	<string>
   binaryData	<map[string]string>
   data	<map[string]string>
   kind	<string>
   metadata	<Object>
```

### ConfigMap创建方式

与 Secret 一样，ConfigMap 也支持四种创建方式：

1、通过 --from-literal：
每个 --from-literal 对应一个信息条目。

```
[root@k8s-master volumes]# kubectl create configmap nginx-config --from-literal=nginx_port=80 --from-literal=server_name=myapp.magedu.com
configmap/nginx-config created
[root@k8s-master volumes]# kubectl get cm
NAME           DATA      AGE
nginx-config   2         6s
[root@k8s-master volumes]# kubectl describe cm nginx-config
Name:         nginx-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
server_name:
----
myapp.magedu.com
nginx_port:
----
80
Events:  <none>
```

2、通过 --from-file：
每个文件内容对应一个信息条目。

```
[root@k8s-master mainfests]# mkdir configmap && cd configmap
[root@k8s-master configmap]# vim www.conf
server {
	server_name myapp.magedu.com;
	listen 80;
	root /data/web/html;
}
[root@k8s-master configmap]# kubectl create configmap nginx-www --from-file=./www.conf 
configmap/nginx-www created
[root@k8s-master configmap]# kubectl get cm
NAME           DATA      AGE
nginx-config   2         3m
nginx-www      1         4s
[root@k8s-master configmap]# kubectl get cm nginx-www -o yaml
apiVersion: v1
data:
  www.conf: "server {\n\tserver_name myapp.magedu.com;\n\tlisten 80;\n\troot /data/web/html;\n}\n"
kind: ConfigMap
metadata:
  creationTimestamp: 2018-10-10T08:50:06Z
  name: nginx-www
  namespace: default
  resourceVersion: "389929"
  selfLink: /api/v1/namespaces/default/configmaps/nginx-www
  uid: 7c3dfc35-cc69-11e8-801a-000c2972dc1f
```

### 如何使用configMap？

1、环境变量方式注入到pod

```
[root@k8s-master configmap]# vim pod-configmap.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-cm-1
  namespace: default
  labels: 
    app: myapp
    tier: frontend
  annotations:
    magedu.com/created-by: "cluster admin"
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
    ports:
    - name: http
      containerPort: 80 
    env:
    - name: NGINX_SERVER_PORT
      valueFrom:
        configMapKeyRef:
          name: nginx-config
          key: nginx_port
    - name: NGINX_SERVER_NAME
      valueFrom:
        configMapKeyRef:
          name: nginx-config
          key: server_name
[root@k8s-master configmap]# kubectl apply -f pod-configmap.yaml 
pod/pod-cm-1 created
[root@k8s-master configmap]# kubectl exec -it pod-cm-1 -- /bin/sh
/ # echo $NGINX_SERVER_PORT
80
/ # echo $NGINX_SERVER_NAME
myapp.magedu.com
```

修改端口，可以发现使用环境变化注入pod中的端口不会根据配置的更改而变化

```
[root@k8s-master volumes]#  kubectl edit cm nginx-config
configmap/nginx-config edited
/ # echo $NGINX_SERVER_PORT
80
```

2、存储卷方式挂载configmap：Volume 形式的 ConfigMap 也支持动态更新

```
[root@k8s-master configmap ~]# vim pod-configmap-2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-cm-2
  namespace: default
  labels: 
    app: myapp
    tier: frontend
  annotations:
    magedu.com/created-by: "cluster admin"
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
    ports:
    - name: http
      containerPort: 80 
    volumeMounts:
    - name: nginxconf
      mountPath: /etc/nginx/config.d/
      readOnly: true
  volumes:
  - name: nginxconf
    configMap:
      name: nginx-config
[root@k8s-master configmap ~]# kubectl apply -f pod-configmap-2.yaml
pod/pod-cm-2 created
[root@k8s-master configmap ~]# kubectl get pods
[root@k8s-master configmap ~]# kubectl exec -it pod-cm-2 -- /bin/sh
/ # cd /etc/nginx/config.d
/ # cat nginx_port
80
/ # cat server_name 
myapp.magedu.com

[root@k8s-master configmap ~]# kubectl edit cm nginx-config  #修改端口，再在容器中查看端口是否变化。
apiVersion: v1
data:
  nginx_port: "800"
  ......
  
/ # cat nginx_port
800
[root@k8s-master configmap ~]# kubectl delete -f pod-configmap2.yaml
```

3、以nginx-www配置nginx

```
[root@k8s-master configmap ~]# vim pod-configmap3.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-cm-3
  namespace: default
  labels: 
    app: myapp
    tier: frontend
  annotations:
    magedu.com/created-by: "cluster admin"
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
    ports:
    - name: http
      containerPort: 80 
    volumeMounts:
    - name: nginxconf
      mountPath: /etc/nginx/conf.d/
      readOnly: true
  volumes:
  - name: nginxconf
    configMap:
      name: nginx-www
[root@k8s-master configmap ~]# kubectl apply -f pod-configmap3.yaml
pod/pod-cm-3 created
[root@k8s-master configmap ~]# kubectl get pods
[root@k8s-master configmap]# kubectl exec -it pod-cm-3 -- /bin/sh
/ # cd /etc/nginx/conf.d/
/etc/nginx/conf.d # ls
www.conf
/etc/nginx/conf.d # cat www.conf 
server {
	server_name myapp.magedu.com;
	listen 80;
	root /data/web/html;
}
```

至此，K8S的存储卷到此结束!



# Statefulset控制器

## 一、statefulset简介

从前面的学习我们知道使用Deployment创建的pod是无状态的，当挂载了Volume之后，如果该pod挂了，Replication Controller会再启动一个pod来保证可用性，但是由于pod是无状态的，pod挂了就会和之前的Volume的关系断开，新创建的Pod无法找到之前的Pod。但是对于用户而言，他们对底层的Pod挂了是没有感知的，但是当Pod挂了之后就无法再使用之前挂载的存储卷。

为了解决这一问题，就引入了StatefulSet用于保留Pod的状态信息。

StatefulSet是为了解决有状态服务的问题（对应Deployments和ReplicaSets是为无状态服务而设计），其应用场景包括：

- 1、稳定的持久化存储，即Pod重新调度后还是能访问到相同的持久化数据，基于PVC来实现
- 2、稳定的网络标志，即Pod重新调度后其PodName和HostName不变，基于Headless Service（即没有Cluster IP的Service）来实现
- 3、有序部署，有序扩展，即Pod是有顺序的，在部署或者扩展的时候要依据定义的顺序依次依次进行（即从0到N-1，在下一个Pod运行之前所有之前的Pod必须都是Running和Ready状态），基于init containers来实现
- 4、有序收缩，有序删除（即从N-1到0）
- 5、有序的滚动更新

从上面的应用场景可以发现，StatefulSet由以下几个部分组成：

* Headless Service（无头服务）用于为Pod资源标识符生成可解析的DNS记录。
* volumeClaimTemplates （存储卷申请模板）基于静态或动态PV供给方式为Pod资源提供专有的固定存储。
* StatefulSet，用于管控Pod资源。

## 二、为什么要有headless？

在deployment中，每一个pod是没有名称，是随机字符串，是无序的。而statefulset中是要求有序的，每一个pod的名称必须是固定的。当节点挂了，重建之后的标识符是不变的，每一个节点的节点名称是不能改变的。pod名称是作为pod识别的唯一标识符，必须保证其标识符的稳定并且唯一。
    为了实现标识符的稳定，这时候就需要一个headless service 解析直达到pod，还需要给pod配置一个唯一的名称。

## 三、为什么要有volumeClainTemplate？

大部分有状态副本集都会用到持久存储，比如分布式系统来说，由于数据是不一样的，每个节点都需要自己专用的存储节点。而在deployment中pod模板中创建的存储卷是一个共享的存储卷，多个pod使用同一个存储卷，而statefulset定义中的每一个pod都不能使用同一个存储卷，由此基于pod模板创建pod是不适应的，这就需要引入volumeClainTemplate，当在使用statefulset创建pod时，会自动生成一个PVC，从而请求绑定一个PV，从而有自己专用的存储卷。Pod名称、PVC和PV关系图如下：

<img src="https://img2018.cnblogs.com/blog/1349539/201903/1349539-20190307152625497-1629030475.png" alt="img" style="zoom:67%;" />



## 四、statefulSet使用演示

> 在创建StatefulSet之前需要准备的东西，值得注意的是创建顺序非常关键，创建顺序如下：
> 1、Volume
> 2、Persistent Volume
> 3、Persistent Volume Claim
> 4、Service
> 5、StatefulSet
> Volume可以有很多种类型，比如nfs、glusterfs等，我们这里使用的ceph RBD来创建。

### （1）查看statefulset的定义

```
[root@k8s-master ~]# kubectl explain statefulset
KIND:     StatefulSet
VERSION:  apps/v1

DESCRIPTION:
     StatefulSet represents a set of pods with consistent identities. Identities
     are defined as: - Network: A single stable DNS and hostname. - Storage: As
     many VolumeClaims as requested. The StatefulSet guarantees that a given
     network identity will always map to the same storage identity.

FIELDS:
   apiVersion	<string>
   kind	<string>
   metadata	<Object>
   spec	<Object>
   status	<Object>
[root@k8s-master ~]# kubectl explain statefulset.spec
KIND:     StatefulSet
VERSION:  apps/v1

RESOURCE: spec <Object>

DESCRIPTION:
     Spec defines the desired identities of pods in this set.

     A StatefulSetSpec is the specification of a StatefulSet.

FIELDS:
   podManagementPolicy	<string>  #Pod管理策略
   replicas	<integer>    #副本数量
   revisionHistoryLimit	<integer>   #历史版本限制
   selector	<Object> -required-    #选择器，必选项
   serviceName	<string> -required-  #服务名称，必选项
   template	<Object> -required-    #模板，必选项
   updateStrategy	<Object>       #更新策略
   volumeClaimTemplates	<[]Object>   #存储卷申请模板，列表对象形式
```

### （2）清单定义StatefulSet

如上所述，一个完整的StatefulSet控制器由一个Headless Service、一个StatefulSet和一个volumeClaimTemplate组成。如下资源清单中的定义：

```
[root@k8s-master mainfests]# vim stateful-demo.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
  labels:
    app: myapp-svc
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: myapp-pod
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: myapp
spec:
  serviceName: myapp-svc
  replicas: 3
  selector:
    matchLabels:
      app: myapp-pod
  template:
    metadata:
      labels:
        app: myapp-pod
    spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: myappdata
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: myappdata
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 2Gi
```

解析：由于StatefulSet资源依赖于一个实现存在的Headless类型的Service资源，所以需要先定义一个名为myapp-svc的Headless Service资源，用于为关联到每个Pod资源创建DNS资源记录。接着定义了一个名为myapp的StatefulSet资源，它通过Pod模板创建了3个Pod资源副本，并基于volumeClaimTemplates向前面创建的PV进行了请求大小为2Gi的专用存储卷。

### （3）删除前期的操作

```
[root@k8s-master mainfests]# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM           STORAGECLASS   REASON    AGE
pv001     1Gi        RWO,RWX        Retain           Available                                            23h
pv002     2Gi        RWO            Retain           Available                                            23h
pv003     2Gi        RWO,RWX        Retain           Bound       default/mypvc                            23h
pv004     4Gi        RWO,RWX        Retain           Available                                            23h
pv005     5Gi        RWO,RWX        Retain           Available                                            23h
[root@k8s-master mainfests]# kubectl delete pods pod-vol-pvc
pod "pod-vol-pvc" deleted
[root@k8s-master mainfests]# kubectl delete pods/pod-cm-3 pods/pod-secret-env pods/pod-vol-hostpath
pod "pod-cm-3" deleted
pod "pod-secret-env" deleted
pod "pod-vol-hostpath" deleted

[root@k8s-master mainfests]# kubectl delete deploy/myapp-backend-pod deploy/tomcat-deploy
deployment.extensions "myapp-backend-pod" deleted
deployment.extensions "tomcat-deploy" deleted

[root@k8s-master mainfests]# kubectl delete pods pod-vol-pvc
pod "pod-vol-pvc" deleted
[root@k8s-master mainfests]# kubectl delete pods/pod-cm-3 pods/pod-secret-env pods/pod-vol-hostpath
pod "pod-cm-3" deleted
pod "pod-secret-env" deleted
pod "pod-vol-hostpath" deleted

[root@k8s-master mainfests]# kubectl delete deploy/myapp-backend-pod deploy/tomcat-deploy
deployment.extensions "myapp-backend-pod" deleted
deployment.extensions "tomcat-deploy" deleted

persistentvolumeclaim "mypvc" deleted
[root@k8s-master mainfests]# kubectl delete pv --all
persistentvolume "pv001" deleted
persistentvolume "pv002" deleted
persistentvolume "pv003" deleted
persistentvolume "pv004" deleted
persistentvolume "pv005" deleted
```

### （4）修改pv的大小为2Gi

```
[root@k8s-master ~]# cd mainfests/volumes
[root@k8s-master volumes]# vim pv-demo.yaml 
[root@k8s-master volumes]# kubectl apply -f pv-demo.yaml 
persistentvolume/pv001 created
persistentvolume/pv002 created
persistentvolume/pv003 created
persistentvolume/pv004 created
persistentvolume/pv005 created
[root@k8s-master volumes]# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
pv001     1Gi        RWO,RWX        Retain           Available                                      5s
pv002     2Gi        RWO            Retain           Available                                      5s
pv003     2Gi        RWO,RWX        Retain           Available                                      5s
pv004     2Gi        RWO,RWX        Retain           Available                                      5s
pv005     2Gi        RWO,RWX        Retain           Available                                      5s
```

### （5）创建statefulset

```
[root@k8s-master mainfests]# kubectl apply -f stateful-demo.yaml 
service/myapp-svc created
statefulset.apps/myapp created
[root@k8s-master mainfests]# kubectl get svc  #查看创建的无头服务myapp-svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP             50d
myapp-svc    ClusterIP   None             <none>        80/TCP              38s
[root@k8s-master mainfests]# kubectl get sts    #查看statefulset
NAME      DESIRED   CURRENT   AGE
myapp     3         3         55s
[root@k8s-master mainfests]# kubectl get pvc    #查看pvc绑定
NAME                STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
myappdata-myapp-0   Bound     pv002     2Gi        RWO                           1m
myappdata-myapp-1   Bound     pv003     2Gi        RWO,RWX                       1m
myappdata-myapp-2   Bound     pv004     2Gi        RWO,RWX                       1m
[root@k8s-master mainfests]# kubectl get pv    #查看pv绑定
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                       STORAGECLASS   REASON    AGE
pv001     1Gi        RWO,RWX        Retain           Available                                                        6m
pv002     2Gi        RWO            Retain           Bound       default/myappdata-myapp-0                            6m
pv003     2Gi        RWO,RWX        Retain           Bound       default/myappdata-myapp-1                            6m
pv004     2Gi        RWO,RWX        Retain           Bound       default/myappdata-myapp-2                            6m
pv005     2Gi        RWO,RWX        Retain           Available                                                        6m

[root@k8s-master mainfests]# kubectl get pods   #查看Pod信息
NAME                     READY     STATUS    RESTARTS   AGE
myapp-0                  1/1       Running   0          2m
myapp-1                  1/1       Running   0          2m
myapp-2                  1/1       Running   0          2m
pod-vol-demo             2/2       Running   0          1d
redis-5b5d6fbbbd-q8ppz   1/1       Running   1          2d
```

当删除的时候是从myapp-2开始进行删除的，关闭是逆向关闭

```
[root@k8s-master mainfests]# kubectl delete -f stateful-demo.yaml 
service "myapp-svc" deleted
statefulset.apps "myapp" deleted

[root@k8s-master ~]# kubectl get pods -w
NAME                     READY     STATUS    RESTARTS   AGE
filebeat-ds-hxgdx        1/1       Running   1          33d
filebeat-ds-s466l        1/1       Running   2          33d
myapp-0                  1/1       Running   0          3m
myapp-1                  1/1       Running   0          3m
myapp-2                  1/1       Running   0          3m
pod-vol-demo             2/2       Running   0          1d
redis-5b5d6fbbbd-q8ppz   1/1       Running   1          2d
myapp-0   1/1       Terminating   0         3m
myapp-2   1/1       Terminating   0         3m
myapp-1   1/1       Terminating   0         3m
myapp-1   0/1       Terminating   0         3m
myapp-0   0/1       Terminating   0         3m
myapp-2   0/1       Terminating   0         3m
myapp-1   0/1       Terminating   0         3m
myapp-1   0/1       Terminating   0         3m
myapp-0   0/1       Terminating   0         4m
myapp-0   0/1       Terminating   0         4m
myapp-2   0/1       Terminating   0         3m
myapp-2   0/1       Terminating   0         3m

此时PVC依旧存在的，再重新创建pod时，依旧会重新去绑定原来的pvc
[root@k8s-master mainfests]# kubectl apply -f stateful-demo.yaml 
service/myapp-svc created
statefulset.apps/myapp created

[root@k8s-master mainfests]# kubectl get pvc
NAME                STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
myappdata-myapp-0   Bound     pv002     2Gi        RWO                           5m
myappdata-myapp-1   Bound     pv003     2Gi        RWO,RWX                       5m
myappdata-myapp-2   Bound     pv004     2Gi        RWO,RWX                       5m
[root@k8s-master mainfests]# kubectl delete -f stateful-demo.yaml 
service "myapp-svc" deleted
statefulset.apps "myapp" deleted

[root@k8s-master ~]# kubectl get pods -w
NAME                     READY     STATUS    RESTARTS   AGE
filebeat-ds-hxgdx        1/1       Running   1          33d
filebeat-ds-s466l        1/1       Running   2          33d
myapp-0                  1/1       Running   0          3m
myapp-1                  1/1       Running   0          3m
myapp-2                  1/1       Running   0          3m
pod-vol-demo             2/2       Running   0          1d
redis-5b5d6fbbbd-q8ppz   1/1       Running   1          2d
myapp-0   1/1       Terminating   0         3m
myapp-2   1/1       Terminating   0         3m
myapp-1   1/1       Terminating   0         3m
myapp-1   0/1       Terminating   0         3m
myapp-0   0/1       Terminating   0         3m
myapp-2   0/1       Terminating   0         3m
myapp-1   0/1       Terminating   0         3m
myapp-1   0/1       Terminating   0         3m
myapp-0   0/1       Terminating   0         4m
myapp-0   0/1       Terminating   0         4m
myapp-2   0/1       Terminating   0         3m
myapp-2   0/1       Terminating   0         3m

此时PVC依旧存在的，再重新创建pod时，依旧会重新去绑定原来的pvc
[root@k8s-master mainfests]# kubectl apply -f stateful-demo.yaml 
service/myapp-svc created
statefulset.apps/myapp created

[root@k8s-master mainfests]# kubectl get pvc
NAME                STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
myappdata-myapp-0   Bound     pv002     2Gi        RWO                           5m
myappdata-myapp-1   Bound     pv003     2Gi        RWO,RWX                       5m
myappdata-myapp-2   Bound     pv004     2Gi        RWO,RWX                       5m
```

## 五、策略

### 1、滚动更新

RollingUpdate 更新策略在 StatefulSet 中实现 Pod 的自动滚动更新。 当StatefulSet的 .spec.updateStrategy.type 设置为 RollingUpdate 时，默认为：RollingUpdate。StatefulSet 控制器将在 StatefulSet 中删除并重新创建每个 Pod。 它将以与 Pod 终止相同的顺序进行（从最大的序数到最小的序数），每次更新一个 Pod。 在更新其前身之前，它将等待正在更新的 Pod 状态变成正在运行并就绪。如下操作的滚动更新是有2-0的顺序更新。

```
[root@k8s-master mainfests]# vim stateful-demo.yaml  #修改image版本为v2
.....
image: ikubernetes/myapp:v2
....
[root@k8s-master mainfests]# kubectl apply -f stateful-demo.yaml 
service/myapp-svc unchanged
statefulset.apps/myapp configured
[root@k8s-master ~]# kubectl get pods -w   #查看滚动更新的过程
NAME                     READY     STATUS    RESTARTS   AGE
myapp-0                  1/1       Running   0          36m
myapp-1                  1/1       Running   0          36m
myapp-2                  1/1       Running   0          36m

myapp-2   1/1       Terminating   0         36m
myapp-2   0/1       Terminating   0         36m
myapp-2   0/1       Terminating   0         36m
myapp-2   0/1       Terminating   0         36m
myapp-2   0/1       Pending   0         0s
myapp-2   0/1       Pending   0         0s
myapp-2   0/1       ContainerCreating   0         0s
myapp-2   1/1       Running   0         2s
myapp-1   1/1       Terminating   0         36m
myapp-1   0/1       Terminating   0         36m
myapp-1   0/1       Terminating   0         36m
myapp-1   0/1       Terminating   0         36m
myapp-1   0/1       Pending   0         0s
myapp-1   0/1       Pending   0         0s
myapp-1   0/1       ContainerCreating   0         0s
myapp-1   1/1       Running   0         1s
myapp-0   1/1       Terminating   0         37m
myapp-0   0/1       Terminating   0         37m
myapp-0   0/1       Terminating   0         37m
myapp-0   0/1       Terminating   0         37m
```

在创建的每一个Pod中，每一个pod自己的名称都是可以被解析的，如下：

```
[root@k8s-master ~]# kubectl get pods -o wide
NAME                     READY     STATUS    RESTARTS   AGE       IP            NODE
myapp-0                  1/1       Running   0          8m        10.244.1.62   k8s-node01
myapp-1                  1/1       Running   0          8m        10.244.2.49   k8s-node02
myapp-2                  1/1       Running   0          8m        10.244.1.61   k8s-node01

[root@k8s-master mainfests]# kubectl exec -it myapp-0 -- /bin/sh
/ # nslookup myapp-0.myapp-svc.default.svc.cluster.local
nslookup: can't resolve '(null)': Name does not resolve

Name:      myapp-0.myapp-svc.default.svc.cluster.local
Address 1: 10.244.1.62 myapp-0.myapp-svc.default.svc.cluster.local
/ # nslookup myapp-1.myapp-svc.default.svc.cluster.local
nslookup: can't resolve '(null)': Name does not resolve

Name:      myapp-1.myapp-svc.default.svc.cluster.local
Address 1: 10.244.2.49 myapp-1.myapp-svc.default.svc.cluster.local
/ # nslookup myapp-2.myapp-svc.default.svc.cluster.local
nslookup: can't resolve '(null)': Name does not resolve

Name:      myapp-2.myapp-svc.default.svc.cluster.local
Address 1: 10.244.1.61 myapp-2.myapp-svc.default.svc.cluster.local

从上面的解析，我们可以看到在容器当中可以通过对Pod的名称进行解析到ip。其解析的域名格式如下：
pod_name.service_name.ns_name.svc.cluster.local
eg: myapp-0.myapp.default.svc.cluster.local
```

### 2、扩展伸缩

```
[root@k8s-master mainfests]# kubectl scale sts myapp --replicas=4  #扩容副本增加到4个
statefulset.apps/myapp scaled
[root@k8s-master ~]# kubectl get pods -w  #动态查看扩容
NAME                     READY     STATUS    RESTARTS   AGE
myapp-0                  1/1       Running   0          23m
myapp-1                  1/1       Running   0          23m
myapp-2                  1/1       Running   0          23m

myapp-3   0/1       Pending   0         0s
myapp-3   0/1       Pending   0         0s
myapp-3   0/1       ContainerCreating   0         0s
myapp-3   1/1       Running   0         1s
[root@k8s-master mainfests]# kubectl get pv  #查看pv绑定
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                       STORAGECLASS   REASON    AGE
pv001     1Gi        RWO,RWX        Retain           Available                                                        1h
pv002     2Gi        RWO            Retain           Bound       default/myappdata-myapp-0                            1h
pv003     2Gi        RWO,RWX        Retain           Bound       default/myappdata-myapp-1                            1h
pv004     2Gi        RWO,RWX        Retain           Bound       default/myappdata-myapp-2                            1h
pv005     2Gi        RWO,RWX        Retain           Bound       default/myappdata-myapp-3                            1h

[root@k8s-master mainfests]# kubectl patch sts myapp -p '{"spec":{"replicas":2}}'  #打补丁方式缩容
statefulset.apps/myapp patched
[root@k8s-master ~]# kubectl get pods -w  #动态查看缩容
NAME                     READY     STATUS    RESTARTS   AGE
myapp-0                  1/1       Running   0          25m
myapp-1                  1/1       Running   0          25m
myapp-2                  1/1       Running   0          25m
myapp-3                  1/1       Running   0          1m
myapp-3   1/1       Terminating   0         2m
myapp-3   0/1       Terminating   0         2m
myapp-3   0/1       Terminating   0         2m
myapp-3   0/1       Terminating   0         2m
myapp-2   1/1       Terminating   0         26m
myapp-2   0/1       Terminating   0         26m
myapp-2   0/1       Terminating   0         27m
myapp-2   0/1       Terminating   0         27m
```

### 3、更新策略和版本升级

修改更新策略，以partition方式进行更新，更新值为2，只有myapp编号大于等于2的才会进行更新。类似于金丝雀部署方式。

```
[root@k8s-master mainfests]# kubectl patch sts myapp -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'
statefulset.apps/myapp patched
[root@k8s-master ~]# kubectl get sts myapp
NAME      DESIRED   CURRENT   AGE
myapp     4         4         1h
[root@k8s-master ~]# kubectl describe sts myapp
Name:               myapp
Namespace:          default
CreationTimestamp:  Wed, 10 Oct 2018 21:58:24 -0400
Selector:           app=myapp-pod
Labels:             <none>
Annotations:        kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"apps/v1","kind":"StatefulSet","metadata":{"annotations":{},"name":"myapp","namespace":"default"},"spec":{"replicas":3,"selector":{"match...
Replicas:           4 desired | 4 total
Update Strategy:    RollingUpdate
  Partition:        2
......
```

版本升级，将image的版本升级为v3，升级后对比myapp-2和myapp-1的image版本是不同的。这样就实现了金丝雀发布的效果。

```
[root@k8s-master mainfests]# kubectl set image sts/myapp myapp=ikubernetes/myapp:v3
statefulset.apps/myapp image updated
[root@k8s-master ~]# kubectl get sts -o wide
NAME      DESIRED   CURRENT   AGE       CONTAINERS   IMAGES
myapp     4         4         1h        myapp        ikubernetes/myapp:v3
[root@k8s-master ~]# kubectl get pods myapp-2 -o yaml |grep image
  - image: ikubernetes/myapp:v3
    imagePullPolicy: IfNotPresent
    image: ikubernetes/myapp:v3
    imageID: docker-pullable://ikubernetes/myapp@sha256:b8d74db2515d3c1391c78c5768272b9344428035ef6d72158fd9f6c4239b2c69

[root@k8s-master ~]# kubectl get pods myapp-1 -o yaml |grep image
  - image: ikubernetes/myapp:v2
    imagePullPolicy: IfNotPresent
    image: ikubernetes/myapp:v2
    imageID: docker-pullable://ikubernetes/myapp@sha256:85a2b81a62f09a414ea33b74fb8aa686ed9b168294b26b4c819df0be0712d358
```

将剩余的Pod也更新版本，只需要将更新策略的partition值改为0即可，如下：

```
[root@k8s-master mainfests]#  kubectl patch sts myapp -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
statefulset.apps/myapp patched

[root@k8s-master ~]# kubectl get pods -w
NAME                     READY     STATUS    RESTARTS   AGE
myapp-0                  1/1       Running   0          58m
myapp-1                  1/1       Running   0          58m
myapp-2                  1/1       Running   0          13m
myapp-3                  1/1       Running   0          13m
myapp-1   1/1       Terminating   0         58m
myapp-1   0/1       Terminating   0         58m
myapp-1   0/1       Terminating   0         58m
myapp-1   0/1       Terminating   0         58m
myapp-1   0/1       Pending   0         0s
myapp-1   0/1       Pending   0         0s
myapp-1   0/1       ContainerCreating   0         0s
myapp-1   1/1       Running   0         2s
myapp-0   1/1       Terminating   0         58m
myapp-0   0/1       Terminating   0         58m
myapp-0   0/1       Terminating   0         58m
myapp-0   0/1       Terminating   0         58m
myapp-0   0/1       Pending   0         0s
myapp-0   0/1       Pending   0         0s
myapp-0   0/1       ContainerCreating   0         0s
myapp-0   1/1       Running   0         2s
```



# 认证、授权和准入控制

API Server作为Kubernetes网关，是访问和管理资源对象的唯一入口，其各种集群组件访问资源都需要经过网关才能进行正常访问和管理。每一次的访问请求都需要进行合法性的检验，其中包括身份验证、操作权限验证以及操作规范验证等，需要通过一系列验证通过之后才能访问或者存储数据到etcd当中。如下图：

<img src="https://img2018.cnblogs.com/blog/1349539/201903/1349539-20190307160125484-1365856991.png" alt="img" style="zoom:67%;" />

 

## 一、ServiceAccount

Service account是为了方便Pod里面的进程调用Kubernetes API或其他外部服务而设计的。它与User account不同

- User account是为人设计的，而service account则是为Pod中的进程调用Kubernetes API而设计；

- User account是跨namespace的，而service account则是仅局限它所在的namespace；

- 每个namespace都会自动创建一个default service account

- Token controller检测service account的创建，并为它们创建[secret](https://www.kubernetes.org.cn/secret)

- 开启ServiceAccount Admission Controller后

  ​		每个Pod在创建后都会自动设置spec.serviceAccount为default（除非指定了其他ServiceAccout）

  ​		验证Pod引用的service account已经存在，否则拒绝创建

  ​		如果Pod没有指定ImagePullSecrets，则把service account的ImagePullSecrets加到Pod中

  ​		每个container启动后都会挂载该service account的token和ca.crt到/var/run/secrets/kubernetes.io/serviceaccount/

 当创建 pod 的时候，如果没有指定一个 service account，系统会自动在与该pod 相同的 namespace 下为其指派一个default service account。而pod和apiserver之间进行通信的账号，称为serviceAccountName。如下：

```
[root@k8s-master ~]# kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
filebeat-ds-hxgdx        1/1       Running   1          34d
filebeat-ds-s466l        1/1       Running   2          34d
myapp-0                  1/1       Running   0          3h
myapp-1                  1/1       Running   0          3h
myapp-2                  1/1       Running   0          4h
myapp-3                  1/1       Running   0          4h
pod-vol-demo             2/2       Running   0          2d
redis-5b5d6fbbbd-q8ppz   1/1       Running   1          2d
[root@k8s-master ~]# kubectl get pods/myapp-0 -o yaml |grep "serviceAccountName"
  serviceAccountName: default
[root@k8s-master ~]# kubectl describe pods myapp-0
Name:               myapp-0
Namespace:          default
......
Volumes:
  ......
  default-token-j5pf5:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-j5pf5
    Optional:    false
```

从上面可以看到每个Pod无论定义与否都会有个存储卷，这个存储卷为default-token-*** token令牌，这就是pod和serviceaccount认证信息。通过secret进行定义，由于认证信息属于敏感信息，所以需要保存在secret资源当中，并以存储卷的方式挂载到Pod当中。从而让Pod内运行的应用通过对应的secret中的信息来连接apiserver，并完成认证。每个 namespace 中都有一个默认的叫做 default 的 service account 资源。进行查看名称空间内的secret，也可以看到对应的default-token。让当前名称空间中所有的pod在连接apiserver时可以使用的预制认证信息，从而保证pod之间的通信。

```
[root@k8s-master ~]# kubectl get sa
NAME      SECRETS   AGE
default   1         50d
[root@k8s-master ~]# kubectl get sa -n ingress-nginx  #前期创建的ingress-nginx名称空间也存在这样的serviceaccount
NAME                           SECRETS   AGE
default                        1         11d
nginx-ingress-serviceaccount   1         11d
[root@k8s-master ~]# kubectl get secret
NAME                    TYPE                                  DATA      AGE
default-token-j5pf5     kubernetes.io/service-account-token   3         50d
mysecret                Opaque                                2         1d
tomcat-ingress-secret   kubernetes.io/tls                     2         10d
[root@k8s-master ~]# kubectl get secret -n ingress-nginx
NAME                                       TYPE                                  DATA      AGE
default-token-zl49j                        kubernetes.io/service-account-token   3         11d
nginx-ingress-serviceaccount-token-mcsf4   kubernetes.io/service-account-token   3         11d
```

而默认的service account 仅仅只能获取当前Pod自身的相关属性，无法观察到其他名称空间Pod的相关属性信息。如果想要扩展Pod，假设有一个Pod需要用于管理其他Pod或者是其他资源对象，是无法通过自身的名称空间的serviceaccount进行获取其他Pod的相关属性信息的，此时就需要进行手动创建一个serviceaccount，并在创建Pod时进行定义。那么serviceaccount该如何进行定义呢？？？实际上，service accout也属于一个k8s资源，如下查看service account的定义方式：

```
[root@k8s-master ~]# kubectl explain sa
KIND:     ServiceAccount
VERSION:  v1

DESCRIPTION:
     ServiceAccount binds together: * a name, understood by users, and perhaps
     by peripheral systems, for an identity * a principal that can be
     authenticated and authorized * a set of secrets

FIELDS:
   apiVersion    <string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#resources

   automountServiceAccountToken    <boolean>
     AutomountServiceAccountToken indicates whether pods running as this service
     account should have an API token automatically mounted. Can be overridden
     at the pod level.

   imagePullSecrets    <[]Object>
     ImagePullSecrets is a list of references to secrets in the same namespace
     to use for pulling any images in pods that reference this ServiceAccount.
     ImagePullSecrets are distinct from Secrets because Secrets can be mounted
     in the pod, but ImagePullSecrets are only accessed by the kubelet. More
     info:
     https://kubernetes.io/docs/concepts/containers/images/#specifying-imagepullsecrets-on-a-pod

   kind    <string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#types-kinds

   metadata    <Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#metadata

   secrets    <[]Object>
     Secrets is the list of secrets allowed to be used by pods running using
     this ServiceAccount. More info:
     https://kubernetes.io/docs/concepts/configuration/secret
```

### service account的创建

```
[root@k8s-master mainfests]# kubectl create serviceaccount mysa -o yaml --dry-run #不执行查看定义方式
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: null
  name: mysa

[root@k8s-master mainfests]# kubectl create serviceaccount mysa -o yaml --dry-run > serviceaccount.yaml  #直接导出为yaml定义文件，可以节省敲键盘的时间
[root@k8s-master mainfests]# kubectl apply -f serviceaccount.yaml 
serviceaccount/mysa created
[root@k8s-master mainfests]# kubectl get serviceaccount/mysa -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ServiceAccount","metadata":{"annotations":{},"creationTimestamp":null,"name":"mysa","namespace":"default"}}
  creationTimestamp: 2018-10-11T08:12:25Z
  name: mysa
  namespace: default
  resourceVersion: "432865"
  selfLink: /api/v1/namespaces/default/serviceaccounts/mysa
  uid: 62fc7782-cd2d-11e8-801a-000c2972dc1f
secrets:
- name: mysa-token-h2mgk
```

看到有一个 token 已经被自动创建，并被 service account 引用。设置非默认的 service account，只需要在 pod 的`spec.serviceAccountName` 字段中将name设置为您想要用的 service account 名字即可。在 pod 创建之初 service account 就必须已经存在，否则创建将被拒绝。需要注意的是不能更新已创建的 pod 的 service account。

###  serviceaccount的自定义使用

这里在default名称空间创建了一个sa为admin，可以看到已经自动生成了一个**Tokens：admin-token-7k5nr。**

```
[root@k8s-master mainfests]# kubectl create serviceaccount admin
serviceaccount/admin created
[root@k8s-master mainfests]# kubectl get sa
NAME      SECRETS   AGE
admin     1         3s
default   1         50d
[root@k8s-master mainfests]# kubectl describe sa/admin
Name:                admin
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   admin-token-7k5nr
Tokens:              admin-token-7k5nr
Events:              <none>
[root@k8s-master mainfests]# kubectl get secret
NAME                    TYPE                                  DATA      AGE
admin-token-7k5nr       kubernetes.io/service-account-token   3         31s
default-token-j5pf5     kubernetes.io/service-account-token   3         50d
mysecret                Opaque                                2         1d
tomcat-ingress-secret   kubernetes.io/tls                     2         10d
[root@k8s-master mainfests]# vim pod-sa-demo.yaml　　#Pod中引用新建的serviceaccount
apiVersion: v1
kind: Pod
metadata:
  name: pod-sa-demo
  namespace: default
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
    ports:
    - name: http
      containerPort: 80
  serviceAccountName: admin
[root@k8s-master mainfests]# kubectl apply -f pod-sa-demo.yaml 
pod/pod-sa-demo created
[root@k8s-master mainfests]# kubectl describe pods pod-sa-demo
......
Volumes:
  admin-token-7k5nr:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  admin-token-7k5nr
    Optional:    false
......
```

 在K8S集群当中，每一个用户对资源的访问都是需要通过apiserver进行通信认证才能进行访问的，那么在此机制当中，对资源的访问可以是token，也可以是通过配置文件的方式进行保存和使用认证信息，可以通过kubectl config进行查看配置，如下：

```
[root@k8s-master mainfests]# kubectl config view
apiVersion: v1
clusters:  #集群列表
- cluster:
    certificate-authority-data: REDACTED
    server: https://192.168.56.11:6443
  name: kubernetes
contexts:  #上下文列表
- context: #定义哪个集群被哪个用户访问
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes  #当前上下文
kind: Config
preferences: {}
users:   #用户列表
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

在上面的配置文件当中，定义了集群、上下文以及用户。其中Config也是K8S的标准资源之一，在该配置文件当中定义了一个集群列表，指定的集群可以有多个；用户列表也可以有多个，指明集群中的用户；而在上下文列表当中，是进行定义可以使用哪个用户对哪个集群进行访问，以及当前使用的上下文是什么。如图：定义了用户kubernetes-admin可以对kubernetes该集群的访问，用户kubernetes-user1对Clluster1集群的访问

<img src="https://img2018.cnblogs.com/blog/1349539/201810/1349539-20181013114713444-798210079.png" alt="img" style="zoom:67%;" />

###  自建证书和账号进行访问apiserver演示

```
（1）生成证书
[root@k8s-master pki]# (umask 077;openssl genrsa -out magedu.key 2048)
Generating RSA private key, 2048 bit long modulus
............................................................................................+++
...................................................................................+++
e is 65537 (0x10001)
[root@k8s-master pki]# ll magedu.key 
-rw------- 1 root root 1675 Oct 12 23:52 magedu.key

（2）使用ca.crt进行签署
[root@k8s-master pki]# openssl req -new -key magedu.key -out magedu.csr -subj "/CN=magedu"  证书签署请求

[root@k8s-master pki]# openssl x509 -req -in magedu.csr -CA ./ca.crt -CAkey ./ca.key -CAcreateserial -out magedu.crt -days 365  #证书签署
Signature ok
subject=/CN=magedu
Getting CA Private Key
[root@k8s-master pki]# openssl x509 -in magedu.crt -text -noout

（3）添加到用户认证
[root@k8s-master pki]# kubectl config set-credentials magedu --client-certificate=./magedu.crt --client-key=./magedu.key --embed-certs=true
User "magedu" set.

[root@k8s-master pki]# kubectl config set-context magedu@kubernetes --cluster=kubernetes --user=magedu
Context "magedu@kubernetes" created.

[root@k8s-master pki]# kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: REDACTED
    server: https://192.168.56.11:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
- context:
    cluster: kubernetes
    user: magedu
  name: magedu@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: magedu
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED

[root@k8s-master pki]# kubectl config use-context magedu@kubernetes
Switched to context "magedu@kubernetes".
[root@k8s-master pki]# kubectl get pods
No resources found.
Error from server (Forbidden): pods is forbidden: User "magedu" cannot list pods in the namespace "default"
```

从上面的演示，当切换成magedu用户进行访问集群时，由于magedu该账户没有管理集群的权限，所以在获取pods资源信息时，会提示Forrbidden。那么下面就再来了解一下怎么对账户进行授权！！！

## 二、RBAC--基于角色的访问控制

 Kubernetes的授权是基于插件形式的，其常用的授权插件有以下几种：

- Node（节点认证）
- ABAC(基于属性的访问控制)
- RBAC（基于角色的访问控制）
- Webhook（基于http回调机制的访问控制）

 让一个用户（Users）扮演一个角色（Role），角色拥有权限，从而让用户拥有这样的权限，随后在授权机制当中，只需要将权限授予某个角色，此时用户将获取对应角色的权限，从而实现角色的访问控制。如图：

<img src="https://img2018.cnblogs.com/blog/1349539/201810/1349539-20181013104659832-1317864140.png" alt="img" style="zoom:80%;" />

 

基于角色的访问控制（Role-Based Access Control, 即”RBAC”）使用”rbac.authorization.k8s.io” API Group实现授权决策，允许管理员通过Kubernetes API动态配置策略。

在k8s的授权机制当中，采用RBAC的方式进行授权，其工作逻辑是　　把对对象的操作权限定义到一个角色当中，再将用户绑定到该角色，从而使用户得到对应角色的权限。此种方式仅作用于名称空间当中，这是什么意思呢？当User1绑定到Role角色当中，User1就获取了对该NamespaceA的操作权限，但是对NamespaceB是没有权限进行操作的，如get，list等操作。
另外，k8s为此还有一种集群级别的授权机制，就是定义一个集群角色（ClusterRole），对集群内的所有资源都有可操作的权限，从而将User2，User3通过ClusterRoleBinding到ClusterRole，从而使User2、User3拥有集群的操作权限。Role、RoleBinding、ClusterRole和ClusterRoleBinding的关系如下图：

<img src="https://img2018.cnblogs.com/blog/1349539/201810/1349539-20181011164935788-596677596.png" alt="img" style="zoom:67%;" />

但是这里有2种绑定ClusterRoleBinding、RoleBinding。也可以使用RoleBinding去绑定ClusterRole。
当使用这种方式进行绑定时，用户仅能获取当前名称空间的所有权限。为什么这么绕呢？？举例有10个名称空间，每个名称空间都需要一个管理员，而该管理员的权限都是一致的。那么此时需要去定义这样的管理员，使用RoleBinding就需要创建10个Role，这样显得更加繁重。为此当使用RoleBinding去绑定一个ClusterRole时，该User仅仅拥有对当前名称空间的集群操作权限，换句话说，此时只需要创建一个ClusterRole就解决了以上的需求。

 **这里要注意的是：RoleBinding仅仅对当前名称空间有对应的权限。**

**在RBAC API中，一个角色包含了一套表示一组权限的规则。 权限以纯粹的累加形式累积（没有”否定”的规则）。 角色可以由命名空间（namespace）内的`Role`对象定义，而整个Kubernetes集群范围内有效的角色则通过`ClusterRole`对象实现。**

## 三、Kubernetes RBAC演示

### **1、User --> Rolebinding --> Role**

####  **（1）角色的创建**

**一个`Role`对象只能用于授予对某一单一命名空间中资源的访问权限**

```
[root@k8s-master ~]# kubectl create role -h   #查看角色创建帮助
Create a role with single rule.

Examples:
  # Create a Role named "pod-reader" that allows user to perform "get", "watch" and "list" on pods
  kubectl create role pod-reader --verb=get --verb=list --verb=watch --resource=pods
  
  # Create a Role named "pod-reader" with ResourceName specified
  kubectl create role pod-reader --verb=get --resource=pods --resource-name=readablepod --resource-name=anotherpod
  
  # Create a Role named "foo" with API Group specified
  kubectl create role foo --verb=get,list,watch --resource=rs.extensions
  
  # Create a Role named "foo" with SubResource specified
  kubectl create role foo --verb=get,list,watch --resource=pods,pods/status

Options:
      --allow-missing-template-keys=true: If true, ignore any errors in templates when a field or map key is missing in
the template. Only applies to golang and jsonpath output formats.
      --dry-run=false: If true, only print the object that would be sent, without sending it.
  -o, --output='': Output format. One of:
json|yaml|name|go-template|go-template-file|templatefile|template|jsonpath|jsonpath-file.
      --resource=[]: Resource that the rule applies to
      --resource-name=[]: Resource in the white list that the rule applies to, repeat this flag for multiple items
      --save-config=false: If true, the configuration of current object will be saved in its annotation. Otherwise, the
annotation will be unchanged. This flag is useful when you want to perform kubectl apply on this object in the future.
      --template='': Template string or path to template file to use when -o=go-template, -o=go-template-file. The
template format is golang templates [http://golang.org/pkg/text/template/#pkg-overview].
      --validate=true: If true, use a schema to validate the input before sending it
      --verb=[]: Verb that applies to the resources contained in the rule

Usage:
  kubectl create role NAME --verb=verb --resource=resource.group/subresource [--resource-name=resourcename] [--dry-run]
[options]
使用kubectl create进行创建角色，指定角色名称，--verb指定权限，--resource指定资源或者资源组，--dry-run单跑模式并不会创建

Use "kubectl options" for a list of global command-line options (applies to all commands).

[root@k8s-master ~]# kubectl create role pods-reader --verb=get,list,watch --resource=pods --dry-run -o yaml #干跑模式查看role的定义

apiVersion: rbac.authorization.k8s.io/v1
kind: Role #资源类型
metadata:
  creationTimestamp: null
  name: pods-reader
rules:
- apiGroups:  #对那些api组内的资源进行操作
  - ""
  resources:  #对那些资源定义
  - pods
  verbs:      #操作权限定义
  - get
  - list
  - watch

[root@k8s-master ~]# cd mainfests/
[root@k8s-master mainfests]# kubectl create role pods-reader --verb=get,list,watch --resource=pods --dry-run -o yaml > role-demo.yaml

[root@k8s-master mainfests]# vim role-demo.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pods-reader
  namespace: default
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
  - watch

[root@k8s-master mainfests]# kubectl apply -f role-demo.yaml  #角色创建
role.rbac.authorization.k8s.io/pods-reader created
[root@k8s-master mainfests]# kubectl get role
NAME          AGE
pods-reader   3s
[root@k8s-master mainfests]# kubectl describe role pods-reader
Name:         pods-reader
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"rbac.authorization.k8s.io/v1","kind":"Role","metadata":{"annotations":{},"name":"pods-reader","namespace":"default"},"rules":[{"apiGroup...
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  pods       []                 []              [get list watch]  #此处已经定义了pods-reader这个角色对pods资源拥有get、list、watch的权限
```

#### **（2）角色的绑定**

**`RoleBinding`可以引用在同一命名空间内定义的`Role`对象。**

```
[root@k8s-master ~]# kubectl create rolebinding -h  #角色绑定创建帮助
Create a RoleBinding for a particular Role or ClusterRole.

Examples:
  # Create a RoleBinding for user1, user2, and group1 using the admin ClusterRole
  kubectl create rolebinding admin --clusterrole=admin --user=user1 --user=user2 --group=group1

Options:
      --allow-missing-template-keys=true: If true, ignore any errors in templates when a field or map key is missing in
the template. Only applies to golang and jsonpath output formats.
      --clusterrole='': ClusterRole this RoleBinding should reference
      --dry-run=false: If true, only print the object that would be sent, without sending it.
      --generator='rolebinding.rbac.authorization.k8s.io/v1alpha1': The name of the API generator to use.
      --group=[]: Groups to bind to the role
  -o, --output='': Output format. One of:
json|yaml|name|templatefile|template|go-template|go-template-file|jsonpath-file|jsonpath.
      --role='': Role this RoleBinding should reference
      --save-config=false: If true, the configuration of current object will be saved in its annotation. Otherwise, the
annotation will be unchanged. This flag is useful when you want to perform kubectl apply on this object in the future.
      --serviceaccount=[]: Service accounts to bind to the role, in the format <namespace>:<name>
      --template='': Template string or path to template file to use when -o=go-template, -o=go-template-file. The
template format is golang templates [http://golang.org/pkg/text/template/#pkg-overview].
      --validate=true: If true, use a schema to validate the input before sending it

Usage:
  kubectl create rolebinding NAME --clusterrole=NAME|--role=NAME [--user=username] [--group=groupname]
[--serviceaccount=namespace:serviceaccountname] [--dry-run] [options]
使用kubectl create进行创建角色绑定，指定角色绑定的名称，--role|--clusterrole指定绑定哪个角色，--user指定哪个用户

Use "kubectl options" for a list of global command-line options (applies to all commands).

[root@k8s-master mainfests]# kubectl create rolebinding magedu-read-pods --role=pods-reader --user=magedu --dry-run -o yaml > rolebinding-demo.yaml
[root@k8s-master mainfests]# cat rolebinding-demo.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: null
  name: magedu-read-pods
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pods-reader
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: magedu
[root@k8s-master mainfests]# kubectl apply -f rolebinding-demo.yaml  #创建角色绑定
rolebinding.rbac.authorization.k8s.io/magedu-read-pods created

[root@k8s-master mainfests]# kubectl describe rolebinding magedu-read-pods #查看角色绑定的信息，这里可以看到user：magedu绑定到了pods-reader这个角色上
Name:         magedu-read-pods
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"rbac.authorization.k8s.io/v1","kind":"RoleBinding","metadata":{"annotations":{},"creationTimestamp":null,"name":"magedu-read-pods","name...
Role:
  Kind:  Role
  Name:  pods-reader
Subjects:
  Kind  Name    Namespace
  ----  ----    ---------
  User  magedu  

 [root@k8s-master ~]# kubectl config use-context magedu@kubernetes #切换magedu这个用户，并使用get获取pods资源信息
Switched to context "magedu@kubernetes".
[root@k8s-master ~]# kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
filebeat-ds-hxgdx        1/1       Running   1          36d
filebeat-ds-s466l        1/1       Running   2          36d
myapp-0                  1/1       Running   0          2d
myapp-1                  1/1       Running   0          2d
myapp-2                  1/1       Running   0          2d
myapp-3                  1/1       Running   0          2d
pod-sa-demo              1/1       Running   0          1d
pod-vol-demo             2/2       Running   0          3d
redis-5b5d6fbbbd-q8ppz   1/1       Running   1          4d
[root@k8s-master ~]# kubectl get pods -n ingress-nginx  #测试获取ingress-nginx这个名称空间的pods信息
No resources found.
Error from server (Forbidden): pods is forbidden: User "magedu" cannot list pods in the namespace "ingress-nginx"
```

从上面的操作，可以总结出，role的定义和绑定，仅作用于当前名称空间，在获取ingress-nginx名称空间时，一样会出现Forbidden！！！

### 2、User --> Clusterrolebinding --> Clusterrole

####  **（1）clusterrole定义**

`ClusterRole`对象可以授予与`Role`对象相同的权限，但由于它们属于集群范围对象， 也可以使用它们授予对以下几种资源的访问权限：

- 集群范围资源（例如节点，即node）
- 非资源类型endpoint（例如”/healthz”）
- 跨所有命名空间的命名空间范围资源（例如pod，需要运行命令`kubectl get pods --all-namespaces`来查询集群中所有的pod）

```
[root@k8s-master mainfests]# kubectl config use-context kubernetes-admin@kubernetes  #切换会kubernetes-admin用户
Switched to context "kubernetes-admin@kubernetes".
[root@k8s-master mainfests]# kubectl create clusterrole cluster-read --verb=get,list,watch --resource=pods -o yaml > clusterrole-demo.yaml

[root@k8s-master mainfests]# vim clusterrole-demo.yaml #定义clusterrole和权限
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-read
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
[root@k8s-master mainfests]# kubectl apply -f clusterrole-demo.yaml  #创建clusterrole
clusterrole.rbac.authorization.k8s.io/cluster-read configured
```

这里我们需要切换回kubernetes-admin账户，是由于magedu账户不具备创建的权限，这也说明普通用户是无法进行创建K8S资源的，除非进行授权。如下，我们另开一个终端，将配置到一个普通用户ik8s上，使其使用magedu账户进行通信

```
[root@k8s-master ~]# useradd ik8s
[root@k8s-master ~]# cp -rp .kube/ /home/ik8s/
[root@k8s-master ~]# chown -R ik8s.ik8s /home/ik8s/
[root@k8s-master ~]# su - ik8s
[ik8s@k8s-master ~]$ kubectl config use-context magedu@kubernetes
Switched to context "magedu@kubernetes".
[ik8s@k8s-master ~]$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: REDACTED
    server: https://192.168.56.11:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
- context:
    cluster: kubernetes
    user: magedu
  name: magedu@kubernetes
current-context: magedu@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: magedu
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

#### **（2）clusterrolebinding定义**

```
[root@k8s-master mainfests]# kubectl get rolebinding  #获取角色绑定信息
NAME               AGE
magedu-read-pods   1h
[root@k8s-master mainfests]# kubectl delete rolebinding magedu-read-pods #删除前面的绑定
rolebinding.rbac.authorization.k8s.io "magedu-read-pods" deleted

[ik8s@k8s-master ~]$ kubectl get pods  #删除后，在ik8s普通用户上进行获取pods资源信息，就立马出现forbidden了
No resources found.
Error from server (Forbidden): pods is forbidden: User "magedu" cannot list pods in the namespace "default"

[root@k8s-master mainfests]# kubectl create clusterrolebinding magedu-read-all-pods --clusterrole=cluster-read --user=magedu --dry-run -o yaml > clusterrolebinding-demo.yaml
[root@k8s-master mainfests]# vim clusterrolebinding-demo.yaml  #创建角色绑定，将magedu绑定到clusterrole：magedu-read-all-pods上
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: magedu-read-all-pods
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-read
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: magedu
  
[root@k8s-master mainfests]# kubectl get clusterrole
NAME                                                                   AGE
admin                                                                  52d
cluster-admin                                                          52d
cluster-read                                                           13m
......

[root@k8s-master mainfests]# kubectl apply -f clusterrolebinding-demo.yaml 
clusterrolebinding.rbac.authorization.k8s.io/magedu-read-all-pods created
[root@k8s-master mainfests]# kubectl get clusterrolebinding
NAME                                                   AGE
......
magedu-read-all-pods                                   10s

[root@k8s-master mainfests]# kubectl describe clusterrolebinding magedu-read-all-pods
Name:         magedu-read-all-pods
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"rbac.authorization.k8s.io/v1beta1","kind":"ClusterRoleBinding","metadata":{"annotations":{},"name":"magedu-read-all-pods","namespace":""...
Role:
  Kind:  ClusterRole
  Name:  cluster-read
Subjects:
  Kind  Name    Namespace
  ----  ----    ---------
  User  magedu  

[ik8s@k8s-master ~]$ kubectl get pods  #角色绑定后在ik8s终端上进行获取pods信息，已经不会出现forbidden了
NAME                     READY     STATUS    RESTARTS   AGE
filebeat-ds-hxgdx        1/1       Running   1          36d
filebeat-ds-s466l        1/1       Running   2          36d
myapp-0                  1/1       Running   0          2d
myapp-1                  1/1       Running   0          2d
myapp-2                  1/1       Running   0          2d
myapp-3                  1/1       Running   0          2d
pod-sa-demo              1/1       Running   0          1d
pod-vol-demo             2/2       Running   0          4d
redis-5b5d6fbbbd-q8ppz   1/1       Running   1          4d
[ik8s@k8s-master ~]$ kubectl get pods -n ingress-nginx #更换名称空间进行查看也是可行的
NAME                                        READY     STATUS    RESTARTS   AGE
default-http-backend-7db7c45b69-nqxw9       1/1       Running   1          4d
nginx-ingress-controller-6bd7c597cb-9fzbw   1/1       Running   0          4d

[ik8s@k8s-master ~]$ kubectl delete pods pod-sa-demo  #但是进行删除pod就无法进行，因为在授权时是没有delete权限的
Error from server (Forbidden): pods "pod-sa-demo" is forbidden: User "magedu" cannot delete pods in the namespace "default"
```

**从上面的实验，我们可以知道对用户magedu进行集群角色绑定，用户magedu将会获取对集群内所有资源的对应权限。**

###  **3、User --> Rolebinding --> Clusterrole**

将maedu通过rolebinding到集群角色magedu-read-pods当中，此时，magedu仅作用于当前名称空间的所有pods资源的权限

```
[root@k8s-master mainfests]# kubectl delete clusterrolebinding magedu-read-all-pods
clusterrolebinding.rbac.authorization.k8s.io "magedu-read-all-pods" deleted

[root@k8s-master mainfests]# kubectl create rolebinding magedu-read-pods --clusterrole=cluster-read --user=magedu --dry-run -o yaml > rolebinding-clusterrole-demo.yaml
[root@k8s-master mainfests]# vim rolebinding-clusterrole-demo.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: magedu-read-pods
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-read
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: magedu

[root@k8s-master mainfests]# kubectl apply -f rolebinding-clusterrole-demo.yaml 
rolebinding.rbac.authorization.k8s.io/magedu-read-pods created

[ik8s@k8s-master ~]$ kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
filebeat-ds-hxgdx        1/1       Running   1          36d
filebeat-ds-s466l        1/1       Running   2          36d
myapp-0                  1/1       Running   0          2d
myapp-1                  1/1       Running   0          2d
myapp-2                  1/1       Running   0          2d
myapp-3                  1/1       Running   0          2d
pod-sa-demo              1/1       Running   0          1d
pod-vol-demo             2/2       Running   0          4d
redis-5b5d6fbbbd-q8ppz   1/1       Running   1          4d
[ik8s@k8s-master ~]$ kubectl get pods -n ingress-nginx
No resources found.
Error from server (Forbidden): pods is forbidden: User "magedu" cannot list pods in the namespace "ingress-nginx"
```

##  四、RBAC的三种授权访问

RBAC不仅仅可以对user进行访问权限的控制，还可以通过group和serviceaccount进行访问权限控制。当我们想对一组用户进行权限分配时，即可将这一组用户归并到一个组内，从而通过对group进行访问权限的分配，达到访问权限控制的效果。

从前面serviceaccount我们可以了解到，Pod可以通过 spec.serviceAccountName来定义其是以某个serviceaccount的身份进行运行，当我们通过RBAC对serviceaccount进行访问授权时，即可以实现Pod对其他资源的访问权限进行控制。也就是说，当我们对serviceaccount进行rolebinding或clusterrolebinding，会使创建Pod拥有对应角色的权限和apiserver进行通信。如图：

<img src="https://img2018.cnblogs.com/blog/1349539/201810/1349539-20181013171216120-114663466.png" alt="img" style="zoom:80%;" />

 



# dashboard认证访问

Dashboard:https://github.com/kubernetes/dashboard

## 一、Dashboard部署

由于需要用到k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0，这里有2种方式进行pull 镜像。docker search该镜像名称，直接pull，再重新进行tag；另外一种方式是通过谷歌容器镜像拉取。

```
[root@k8s-node01 ~]# docker pull siriuszg/kubernetes-dashboard-amd64  
[root@k8s-node01 ~]# docker tag siriuszg/kubernetes-dashboard-amd64:latest k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0

或者是
[root@k8s-node01 ~]# docker pull mirrorgooglecontainers/kubernetes-dashboard-amd64:v1.10.0
```

再看其部署的过程：

```
[root@k8s-master ~]# kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created

[root@k8s-master ~]# kubectl get pods -n kube-system
NAME                                   READY     STATUS    RESTARTS   AGE
coredns-78fcdf6894-nmcmz               1/1       Running   1          54d
coredns-78fcdf6894-p5pfm               1/1       Running   1          54d
etcd-k8s-master                        1/1       Running   2          54d
kube-apiserver-k8s-master              1/1       Running   9          54d
kube-controller-manager-k8s-master     1/1       Running   5          54d
kube-flannel-ds-n5c86                  1/1       Running   1          54d
kube-flannel-ds-nrcw2                  1/1       Running   1          52d
kube-flannel-ds-pgpr7                  1/1       Running   5          54d
kube-proxy-glzth                       1/1       Running   1          52d
kube-proxy-rxlt7                       1/1       Running   2          54d
kube-proxy-vxckf                       1/1       Running   4          54d
kube-scheduler-k8s-master              1/1       Running   3          54d
kubernetes-dashboard-767dc7d4d-n4clq   1/1       Running   0          3s

[root@k8s-master ~]# kubectl get svc -n kube-system
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
kube-dns               ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP   54d
kubernetes-dashboard   ClusterIP   10.105.204.4   <none>        443/TCP         30m

[root@k8s-master ~]# kubectl patch svc kubernetes-dashboard -p '{"spec":{"type":"NodePort"}}' -n kube-system  #以打补丁方式修改dasboard的访问方式
service/kubernetes-dashboard patched
[root@k8s-master ~]# kubectl get svc -n kube-system
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
kube-dns               ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP   54d
kubernetes-dashboard   NodePort    10.105.204.4   <none>        443:32645/TCP   31m
```

浏览器访问：https://192.168.56.12:32645，如图：**这里需要注意的是谷歌浏览器会禁止不安全证书访问，建议使用火狐浏览器，并且需要在高级选项中添加信任**

![img](https://img2018.cnblogs.com/blog/1349539/201810/1349539-20181015144554239-72262775.png)

在k8s中 dashboard可以有两种访问方式：kubeconfig（HTTPS）和token（http）

### 1、token认证

 （1）创建dashboard专用证书

```
[root@k8s-master pki]# (umask 077;openssl genrsa -out dashboard.key 2048)
Generating RSA private key, 2048 bit long modulus
.....................................................................................................+++
..........................+++
e is 65537 (0x10001)
```

（2）证书签署请求

```
[root@k8s-master pki]# openssl req -new -key dashboard.key -out dashboard.csr -subj "/O=magedu/CN=dashboard"
[root@k8s-master pki]# openssl x509 -req -in dashboard.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out dashboard.crt -days 365
Signature ok
subject=/O=magedu/CN=dashboard
Getting CA Private Key
```

（3）定义令牌方式仅能访问default名称空间

```
[root@k8s-master pki]# kubectl create secret generic dashboard-cert -n kube-system --from-file=./dashboard.crt --from-file=dashboard.key=./dashboard.key   #secret创建
secret/dashboard-cert created

[root@k8s-master pki]# kubectl get secret -n kube-system |grep dashboard
dashboard-cert                                   Opaque                                2         1m
kubernetes-dashboard-certs                       Opaque                                0         3h
kubernetes-dashboard-key-holder                  Opaque                                2         3h
kubernetes-dashboard-token-jpbgw                 kubernetes.io/service-account-token   3         3h

[root@k8s-master pki]# kubectl create serviceaccount def-ns-admin -n default  #创建serviceaccount
serviceaccount/def-ns-admin created

[root@k8s-master pki]# kubectl create rolebinding def-ns-admin --clusterrole=admin --serviceaccount=default:def-ns-admin  #service account账户绑定到集群角色admin
rolebinding.rbac.authorization.k8s.io/def-ns-admin created

[root@k8s-master pki]# kubectl describe secret def-ns-admin-token-k9fz9  #查看def-ns-admin这个serviceaccount的token
Name:         def-ns-admin-token-k9fz9
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=def-ns-admin
              kubernetes.io/service-account.uid=56ed901c-d042-11e8-801a-000c2972dc1f

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZi1ucy1hZG1pbi10b2tlbi1rOWZ6OSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJkZWYtbnMtYWRtaW4iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI1NmVkOTAxYy1kMDQyLTExZTgtODAxYS0wMDBjMjk3MmRjMWYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpkZWYtbnMtYWRtaW4ifQ.QfB5RR19nBv4-kFYyzW5-2n5Ksg-kON8lU18-COLBNfObQTDHs926m4k9f_5bto4YGncYi7sV_3oEec8ouW1FRjJWfY677L1IqIlwcuqc-g0DUo21zkjY_s3Lv3JSb_AfXUbZ7VTeWOhvwonqfK8uriGO1-XET-RBk1CE4Go1sL7X5qDgPjNO1g85D9IbIZG64VygplT6yZNc-b7tLNn_O49STthy6J0jdNk8lYxjy6UJohoTicy2XkZMHp8bNPBj9RqGqMSnnJxny5WO3vHxYAodKx7h6w-PtuON84lICnhiJ06RzsWjZfdeaQYg4gCZmd2J6Hdq0_K32n3l3kFLg
```

将该token复制后，填入验证，要知道的是，该token认证仅可以查看default名称空间的内容，如下图：

![img](https://img2018.cnblogs.com/blog/1349539/201810/1349539-20181015145441660-710436835.png)

###  2、kubeconfig认证 

（1）配置def-ns-admin的集群信息

```
[root@k8s-master pki]# kubectl config set-cluster kubernetes --certificate-authority=./ca.crt --server="https://192.168.56.11:6443" --embed-certs=true --kubeconfig=/root/def-ns-admin.conf
Cluster "kubernetes" set.
```

（2）使用token写入集群验证

```
[root@k8s-master ~]# kubectl config set-credentials -h  #认证的方式可以通过crt和key文件，也可以使用token进行配置，这里使用tonken

Usage:
  kubectl config set-credentials NAME [--client-certificate=path/to/certfile] [--client-key=path/to/keyfile]
[--token=bearer_token] [--username=basic_user] [--password=basic_password] [--auth-provider=provider_name]
[--auth-provider-arg=key=value] [options]

[root@k8s-master pki]# kubectl describe secret def-ns-admin-token-k9fz9
Name:         def-ns-admin-token-k9fz9
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=def-ns-admin
              kubernetes.io/service-account.uid=56ed901c-d042-11e8-801a-000c2972dc1f

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZi1ucy1hZG1pbi10b2tlbi1rOWZ6OSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJkZWYtbnMtYWRtaW4iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI1NmVkOTAxYy1kMDQyLTExZTgtODAxYS0wMDBjMjk3MmRjMWYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpkZWYtbnMtYWRtaW4ifQ.QfB5RR19nBv4-kFYyzW5-2n5Ksg-kON8lU18-COLBNfObQTDHs926m4k9f_5bto4YGncYi7sV_3oEec8ouW1FRjJWfY677L1IqIlwcuqc-g0DUo21zkjY_s3Lv3JSb_AfXUbZ7VTeWOhvwonqfK8uriGO1-XET-RBk1CE4Go1sL7X5qDgPjNO1g85D9IbIZG64VygplT6yZNc-b7tLNn_O49STthy6J0jdNk8lYxjy6UJohoTicy2XkZMHp8bNPBj9RqGqMSnnJxny5WO3vHxYAodKx7h6w-PtuON84lICnhiJ06RzsWjZfdeaQYg4gCZmd2J6Hdq0_K32n3l3kFLg

这里的token是base64编码，此处需要进行解码操作
[root@k8s-master ~]# kubectl get secret def-ns-admin-token-k9fz9 -o jsonpath={.data.token} |base64 -d
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZi1ucy1hZG1pbi10b2tlbi1rOWZ6OSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJkZWYtbnMtYWRtaW4iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI1NmVkOTAxYy1kMDQyLTExZTgtODAxYS0wMDBjMjk3MmRjMWYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpkZWYtbnMtYWRtaW4ifQ.QfB5RR19nBv4-kFYyzW5-2n5Ksg-kON8lU18-COLBNfObQTDHs926m4k9f_5bto4YGncYi7sV_3oEec8ouW1FRjJWfY677L1IqIlwcuqc-g0DUo21zkjY_s3Lv3JSb_AfXUbZ7VTeWOhvwonqfK8uriGO1-XET-RBk1CE4Go1sL7X5qDgPjNO1g85D9IbIZG64VygplT6yZNc-b7tLNn_O49STthy6J0jdNk8lYxjy6UJohoTicy2XkZMHp8bNPBj9RqGqMSnnJxny5WO3vHxYAodKx7h6w-PtuON84lICnhiJ06RzsWjZfdeaQYg4gCZmd2J6Hdq0_K32n3l3kFLg

配置token信息
[root@k8s-master ~]# kubectl config set-credentials def-ns-admin --token=eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZi1ucy1hZG1pbi10b2tlbi1rOWZ6OSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJkZWYtbnMtYWRtaW4iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI1NmVkOTAxYy1kMDQyLTExZTgtODAxYS0wMDBjMjk3MmRjMWYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpkZWYtbnMtYWRtaW4ifQ.QfB5RR19nBv4-kFYyzW5-2n5Ksg-kON8lU18-COLBNfObQTDHs926m4k9f_5bto4YGncYi7sV_3oEec8ouW1FRjJWfY677L1IqIlwcuqc-g0DUo21zkjY_s3Lv3JSb_AfXUbZ7VTeWOhvwonqfK8uriGO1-XET-RBk1CE4Go1sL7X5qDgPjNO1g85D9IbIZG64VygplT6yZNc-b7tLNn_O49STthy6J0jdNk8lYxjy6UJohoTicy2XkZMHp8bNPBj9RqGqMSnnJxny5WO3vHxYAodKx7h6w-PtuON84lICnhiJ06RzsWjZfdeaQYg4gCZmd2J6Hdq0_K32n3l3kFLg --kubeconfig=/root/def-ns-admin.conf 
User "def-ns-admin" set.
```

（3）配置上下文和当前上下文

```
[root@k8s-master ~]# kubectl config set-context def-ns-admin@kubernetes --cluster=kubernetes --user=def-ns-admin --kubeconfig=/root/def-ns-admin.conf 
Context "def-ns-admin@kubernetes" created.

[root@k8s-master ~]# kubectl config use-context def-ns-admin@kubernetes --kubeconfig=/root/def-ns-admin.conf 
Switched to context "def-ns-admin@kubernetes".

[root@k8s-master ~]# kubectl config view --kubeconfig=/root/def-ns-admin.conf 
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: REDACTED
    server: https://192.168.56.11:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: def-ns-admin
  name: def-ns-admin@kubernetes
current-context: def-ns-admin@kubernetes
kind: Config
preferences: {}
users:
- name: def-ns-admin
  user:
    token: eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZi1ucy1hZG1pbi10b2tlbi1rOWZ6OSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJkZWYtbnMtYWRtaW4iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI1NmVkOTAxYy1kMDQyLTExZTgtODAxYS0wMDBjMjk3MmRjMWYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpkZWYtbnMtYWRtaW4ifQ.QfB5RR19nBv4-kFYyzW5-2n5Ksg-kON8lU18-COLBNfObQTDHs926m4k9f_5bto4YGncYi7sV_3oEec8ouW1FRjJWfY677L1IqIlwcuqc-g0DUo21zkjY_s3Lv3JSb_AfXUbZ7VTeWOhvwonqfK8uriGO1-XET-RBk1CE4Go1sL7X5qDgPjNO1g85D9IbIZG64VygplT6yZNc-b7tLNn_O49STthy6J0jdNk8lYxjy6UJohoTicy2XkZMHp8bNPBj9RqGqMSnnJxny5WO3vHxYAodKx7h6w-PtuON84lICnhiJ06RzsWjZfdeaQYg4gCZmd2J6Hdq0_K32n3l3kFLg
```

将/root/def-ns-admin.conf文件发送到宿主机，浏览器访问时选择Kubeconfig认证，载入该配置文件，点击登陆，即可实现访问，如图： 

![img](https://img2018.cnblogs.com/blog/1349539/201810/1349539-20181015155039479-1039305805.png)

## 二、总结

**1、部署dashboard：**

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

**2、将Service改为Node Port方式进行访问：**

```
kubectl patch svc kubernetes-dashboard -p '{"spec":{"type":"NodePort"}}' -n kube-system
```

**3、访问认证：**

认证时的账号必须为ServiceAccount：其作用是被dashboard pod拿来由kubenetes进行认证；认证方式有2种：
**token：**

- （1）创建ServiceAccount，根据其管理目标，使用rolebinding或clusterbinding绑定至合理的role或clusterrole；
- （2）获取此ServiceAccount的secret，查看secret的详细信息，其中就有token；
- （3）复制token到认证页面即可登录。

**kubeconfig**：把ServiceAccount的token封装为kubeconfig文件

- （1）创建ServiceAccount，根据其管理目标，使用rolebinding或clusterbinding绑定至合理的role或clusterrole；
- （2）kubectl get secret |awk '/^ServiceAccount/{print $1}'
- KUBE_TOKEN=$(kubectl get secret SERVICEACCOUNT_SECRET_NAME -o jsonpath={.data.token} | base64 -d)
- （3）生成kubeconfig文件

```
kubectl config set-cluster
kubectl config set-credentials NAME --token=$KUBE_TOKEN
kubectl config set-context
kubectl config use-context
```



# 网络模型和网络策略

## 一、Kubernetes网络模型和CNI插件

 在Kubernetes中设计了一种网络模型，要求无论容器运行在集群中的哪个节点，所有容器都能通过一个扁平的网络平面进行通信，即在同一IP网络中。需要注意的是：在K8S集群中，IP地址分配是以Pod对象为单位，而非容器，同一Pod内的所有容器共享同一网络名称空间。

### 1.1、Docker网络模型

 了解Docker的友友们都应该清楚，Docker容器的原生网络模型主要有3种：Bridge（桥接）、Host（主机）、none。

- Bridge：借助虚拟网桥设备为容器建立网络连接。
- Host：设置容器直接共享当前节点主机的网络名称空间。
- none：多个容器共享同一个网络名称空间。

```
#使用以下命令查看docker原生的三种网络
[root@localhost ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
0efec019c899        bridge              bridge              local
40add8bb5f07        host                host                local
ad94f0b1cca6        none                null                local

#none网络，在该网络下的容器仅有lo网卡，属于封闭式网络，通常用于对安全性要求较高并且不需要联网的应用
[root@localhost ~]# docker run -it --network=none busybox
/ # ifconfig
lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

#host网络，共享宿主机的网络名称空间，容器网络配置和host一致，但是存在端口冲突的问题
[root@localhost ~]# docker run -it --network=host busybox
/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether 00:0c:29:69:a7:23 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.4/24 brd 192.168.1.255 scope global dynamic eth0
       valid_lft 84129sec preferred_lft 84129sec
    inet6 fe80::20c:29ff:fe69:a723/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue 
    link/ether 02:42:29:09:8f:dd brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:29ff:fe09:8fdd/64 scope link 
       valid_lft forever preferred_lft forever
/ # hostname
localhost

#bridge网络，Docker安装完成时会创建一个名为docker0的linux bridge，不指定网络时，创建的网络默认为桥接网络，都会桥接到docker0上。
[root@localhost ~]# brctl show
bridge name	bridge id		STP enabled	interfaces
docker0		8000.024229098fdd	no		

[root@localhost ~]# docker run -d nginx	#运行一个nginx容器
c760a1b6c9891c02c992972d10a99639d4816c4160d633f1c5076292855bbf2b

[root@localhost ~]# brctl show		
bridge name	bridge id		STP enabled	interfaces
docker0		8000.024229098fdd	no		veth3f1b114

一个新的网络接口veth3f1b114桥接到了docker0上，veth3f1b114就是新创建的容器的虚拟网卡。进入容器查看其网络配置：
[root@localhost ~]# docker exec -it c760a1b6c98 bash
root@c760a1b6c989:/# apt-get update
root@c760a1b6c989:/# apt-get iproute
root@c760a1b6c989:/# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
38: eth0@if39: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

 从上可以看到容器内有一个网卡`eth0@if39`，实际上`eth0@if39`和`veth3f1b114`是一对`veth pair`。`veth pair`是一种成对出现的特殊网络设备，可以想象它们由一根虚拟的网线进行连接的一对网卡，`eth0@if39`在容器中，`veth3f1b114`挂在网桥`docker0`上，最终的效果就是`eth0@if39`也挂在了`docker0上`。

 桥接式网络是目前较为流行和默认的解决方案。但是这种方案的弊端是无法跨主机通信的，仅能在宿主机本地进行，而解决该问题的方法就是NAT。所有接入到该桥接设备上的容器都会被NAT隐藏，它们发往Docker主机外部的所有流量都会经过源地址转换后发出，并且默认是无法直接接受节点之外的其他主机发来的请求。当需要接入Docker主机外部流量，就需要进行目标地址转换甚至端口转换将其暴露在外部网络当中。如下图：

![img](https://img2018.cnblogs.com/blog/1349539/201903/1349539-20190312153957757-632072688.png)

 容器内的属于私有地址，需要在左侧的主机上的eth0上进行源地址转换，而右侧的地址需要被访问，就需要将eth0的地址进行NAT转换。SNAT---->DNAT

 这样的通信方式会比较麻烦，从而需要借助第三方的网络插件实现这样的跨主机通信的网络策略。

### 1.2、Kubernetes网络模型

 我们知道的是，在K8S上的网络通信包含以下几类：

- 容器间的通信：同一个Pod内的多个容器间的通信，它们之间通过lo网卡进行通信。

- Pod之间的通信：通过Pod IP地址进行通信。

- Pod和Service之间的通信：Pod IP地址和Service IP进行通信，两者并不属于同一网络，实现方式是通过IPVS或iptables规则转发。

- Service和集群外部客户端的通信，实现方式：Ingress、NodePort、Loadbalance

  K8S网络的实现不是集群内部自己实现，而是依赖于第三方网络插件----CNI（Container Network Interface）

  flannel、calico、canel等是目前比较流行的第三方网络插件。

  这三种的网络插件需要实现Pod网络方案的方式通常有以下几种：

 虚拟网桥、多路复用（MacVLAN）、硬件交换（SR-IOV）

 无论是上面的哪种方式在容器当中实现，都需要大量的操作步骤，而K8S支持CNI插件进行编排网络，以实现Pod和集群网络管理功能的自动化。每次Pod被初始化或删除，kubelet都会调用默认的CNI插件去创建一个虚拟设备接口附加到相关的底层网络，为Pod去配置IP地址、路由信息并映射到Pod对象的网络名称空间。

 在配置Pod网络时，kubelet会在默认的/etc/cni/net.d/目录中去查找CNI JSON配置文件，然后通过type属性到/opt/cni/bin中查找相关的插件二进制文件，如下面的"portmap"。然后CNI插件调用IPAM插件（IP地址管理插件）来配置每个接口的IP地址：

```
[root@k8s-master ~]# cat /etc/cni/net.d/10-flannel.conflist 
{
  "name": "cbr0",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}

kubelet调用第三方插件，进行网络地址的分配
```

 CNI主要是定义容器网络模型规范，链接容器管理系统和网络插件，两者主要通过上面的JSON格式文件进行通信，实现容器的网络功能。CNI的主要核心是：在创建容器时，先创建好网络名称空间（netns），然后调用CNI插件为这个netns配置网络，最后在启动容器内的进程。

 常见的CNI网络插件包含以下几种：

- Flannel：为Kubernetes提供叠加网络的网络插件，基于TUN/TAP隧道技术，使用UDP封装IP报文进行创建叠 加网络，借助etcd维护网络的分配情况，缺点：无法支持网络策略访问控制。
- Calico：基于BGP的三层网络插件，也支持网络策略进而实现网络的访问控制；它在每台主机上都运行一个虚拟路由，利用Linux内核转发网络数据包，并借助iptables实现防火墙功能。实际上Calico最后的实现就是将每台主机都变成了一台路由器，将各个网络进行连接起来，实现跨主机通信的功能。
- Canal：由Flannel和Calico联合发布的一个统一网络插件，提供CNI网络插件，并支持网络策略实现。
- 其他的还包括Weave Net、Contiv、OpenContrail、Romana、NSX-T、kube-router等等。而Flannel和Calico是目前最流行的选择方案。

### 1.3、Flannel网络插件

 在各节点上的Docker主机在docker0上默认使用同一个子网，不同节点的容器都有可能会获取到相同的地址，那么在跨节点通信时就会出现地址冲突的问题。并且在多个节点上的docker0使用不同的子网，也会因为没有准确的路由信息导致无法准确送达报文。

 而为了解决这一问题，Flannel的解决办法是，预留一个使用网络，如10.244.0.0/16，然后自动为每个节点的Docker容器引擎分配一个子网，如10.244.1.0/24和10.244.2.0/24，并将分配信息保存在etcd持久存储。

 第二个问题的解决，Flannel是采用不同类型的后端网络模型进行处理。其后端的类型有以下几种：

- VxLAN：使用内核中的VxLAN模块进行封装报文。也是flannel推荐的方式，其报文格式如下：

![img](https://img2018.cnblogs.com/blog/1349539/201903/1349539-20190312153144413-1844597225.png)

- host-gw：即Host GateWay，通过在节点上创建目标容器地址的路由直接完成报文转发，要求各节点必须在同一个2层网络，对报文转发性能要求较高的场景使用。
- UDP：使用普通的UDP报文封装完成隧道转发。

### 1.4、VxLAN后端和direct routing

 VxLAN（Virtual extensible Local Area Network）虚拟可扩展局域网，采用MAC in UDP封装方式，具体的实现方式为：

- 1、将虚拟网络的数据帧添加到VxLAN首部，封装在物理网络的UDP报文中
- 2、以传统网络的通信方式传送该UDP报文
- 3、到达目的主机后，去掉物理网络报文的头部信息以及VxLAN首部，并交付给目的终端

跨节点的Pod之间的通信就是以上的一个过程，整个过程中通信双方对物理网络是没有感知的。如下网络图：

![img](https://img2018.cnblogs.com/blog/1349539/201903/1349539-20190312153127876-1058842025.png)

 VxLAN的部署可以直接在官方上找到其YAML文件，如下：

```
[root@k8s-master:~# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds-amd64 created
daemonset.extensions/kube-flannel-ds-arm64 created
daemonset.extensions/kube-flannel-ds-arm created
daemonset.extensions/kube-flannel-ds-ppc64le created
daemonset.extensions/kube-flannel-ds-s390x created

#输出如下结果表示flannel可以正常运行了
[root@k8s-master ~]# kubectl get daemonset -n kube-system
NAME              DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR                   AGE
kube-flannel-ds   3         3         3         3            3           beta.kubernetes.io/arch=amd64   202d
kube-proxy        3         3         3         3            3           beta.kubernetes.io/arch=amd64   202d
```

运行正常后，flanneld会在宿主机的/etc/cni/net.d目录下生成自已的配置文件，kubelet将会调用它。

网络插件运行成功后，Node状态才Ready。

```
[root@k8s-master ~]# kubectl get node
NAME         STATUS    ROLES     AGE       VERSION
k8s-master   Ready     master    202d      v1.11.2
k8s-node01   Ready     <none>    202d      v1.11.2
k8s-node02   Ready     <none>    201d      v1.11.2
```

flannel运行后，在各Node宿主机多了一个网络接口：

```
#master节点的flannel.1网络接口，其网段为：10.244.0.0
[root@k8s-master ~]# ifconfig flannel.1
flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.244.0.0  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::31:5dff:fe01:4bc0  prefixlen 64  scopeid 0x20<link>
        ether 02:31:5d:01:4b:c0  txqueuelen 0  (Ethernet)
        RX packets 1659239  bytes 151803796 (144.7 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2115439  bytes 6859187242 (6.3 GiB)
        TX errors 0  dropped 10 overruns 0  carrier 0  collisions 0

#node1节点的flannel.1网络接口，其网段为：10.244.1.0
[root@k8s-node01 ~]# ifconfig flannel.1
flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.244.1.0  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::2806:4ff:fe71:2253  prefixlen 64  scopeid 0x20<link>
        ether 2a:06:04:71:22:53  txqueuelen 0  (Ethernet)
        RX packets 136904  bytes 16191144 (15.4 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 180775  bytes 512637365 (488.8 MiB)
        TX errors 0  dropped 8 overruns 0  carrier 0  collisions 0

#node2节点的flannel.1网络接口，其网段为：10.244.2.0
[root@k8s-node02 ~]# ifconfig flannel.1
flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.244.2.0  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::58b7:7aff:fe8d:2d  prefixlen 64  scopeid 0x20<link>
        ether 5a:b7:7a:8d:00:2d  txqueuelen 0  (Ethernet)
        RX packets 9847824  bytes 12463823121 (11.6 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2108796  bytes 185073173 (176.4 MiB)
        TX errors 0  dropped 13 overruns 0  carrier 0  collisions 0
```

从上面的结果可以知道 ：

1. flannel默认就是VXLAN模式，即Overlay Network。
2. flanneld创建了一个flannel.1接口，它是专门用来封装隧道协议的，默认分给集群的Pod网段为10.244.0.0/16。
3. flannel给k8s-master节点配置的Pod网络为10.244.0.0段，给k8s-node01节点配置的Pod网络为10.244.1.0段，给k8s-node01节点配置的Pod网络为10.244.2.0段，如果有更多的节点，以此类推。

**举个实际例子**

```
#启动一个nginx容器，副本为3
[root@k8s-master ~]# kubectl run nginx --image=nginx:1.14 --port=80 --replicas=3
deployment.apps/nginx created

#查看Pod
[root@k8s-master ~]# kubectl get pods -o wide |grep nginx
nginx-5bd76bcc4f-8s64s                1/1       Running    0          2m        10.244.2.85    k8s-node02
nginx-5bd76bcc4f-mr6k5                1/1       Running    0          2m        10.244.1.146   k8s-node01
nginx-5bd76bcc4f-pb257                1/1       Running    0          2m        10.244.0.17    k8s-master
```

可以看到，3个Pod都分别运行在各个节点之上，其中master上的Pod的ip为：10.244.0.17，在master节点上查看网络接口可以发现在各个节点上多了一个虚拟接口cni0，其ip地址为10.244.0.1。它是由flanneld创建的一个虚拟网桥叫cni0，在Pod本地通信使用。 **这里需要注意的是，cni0虚拟网桥，仅作用于本地通信！！！！**

```
[root@k8s-master ~]# ifconfig cni0
cni0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.244.0.1  netmask 255.255.255.0  broadcast 0.0.0.0
        inet6 fe80::848a:beff:fe44:4959  prefixlen 64  scopeid 0x20<link>
        ether 0a:58:0a:f4:00:01  txqueuelen 1000  (Ethernet)
        RX packets 2772994  bytes 300522237 (286.6 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3180562  bytes 928687288 (885.6 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

flanneld为每个Pod创建一对veth虚拟设备，一端放在容器接口上，一端放在cni0桥上。 使用brctl查看该网桥：

```
#可以看到有一veth的网络接口桥接在cni0网桥上
[root@k8s-master ~]# brctl show cni0
bridge name	bridge id		STP enabled	interfaces
cni0		8000.0a580af40001	no		veth020fafae


#宿主机ping测试访问Pod ip
[root@k8s-master ~]# ping 10.244.0.17
PING 10.244.0.17 (10.244.0.17) 56(84) bytes of data.
64 bytes from 10.244.0.17: icmp_seq=1 ttl=64 time=0.291 ms
64 bytes from 10.244.0.17: icmp_seq=2 ttl=64 time=0.081 ms
^C
--- 10.244.0.17 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3000ms
rtt min/avg/max/mdev = 0.055/0.129/0.291/0.094 ms
```

在现有的Flannel VxLAN网络中，两台主机上的Pod间通信，也是正常的，如master节点上的Pod访问node01上的Pod：

```
[root@k8s-master ~]# kubectl exec -it nginx-5bd76bcc4f-pb257 -- /bin/bash
root@nginx-5bd76bcc4f-pb257:/# ping 10.244.1.146
PING 10.244.1.146 (10.244.1.146) 56(84) bytes of data.
64 bytes from 10.244.1.146: icmp_seq=1 ttl=62 time=1.44 ms
64 bytes from 10.244.1.146: icmp_seq=2 ttl=62 time=0.713 ms
64 bytes from 10.244.1.146: icmp_seq=3 ttl=62 time=0.713 ms
64 bytes from 10.244.1.146: icmp_seq=4 ttl=62 time=0.558 ms
^C
--- 10.244.1.146 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 0.558/0.858/1.448/0.346 ms
```

可以看到容器跨主机是可以正常通信的，那么容器的跨主机通信是如何实现的呢？？？？？master上查看路由表信息：

```
[root@k8s-master ~]# ip route
......
10.244.1.0/24 via 10.244.1.0 dev flannel.1 onlink 
10.244.2.0/24 via 10.244.2.0 dev flannel.1 onlink 
......
```

发送到`10.244.1.0/24`和`10.244.20/24`网段的数据报文发给本机的flannel.1接口，即进入二层隧道，然后对数据报文进行封装（封装VxLAN首部-->UDP首部-->IP首部-->以太网首部），到达目标Node节点后，由目标Node上的flannel.1进行解封装。使用`tcpdump`进行 抓一下包，如下：

```
#在宿主机和容器内都进行ping另外一台主机上的Pod ip并进行抓包
[root@k8s-master ~]# ping -c 10 10.244.1.146
[root@k8s-master ~]# kubectl exec -it nginx-5bd76bcc4f-pb257 -- /bin/bash
root@nginx-5bd76bcc4f-pb257:/# ping 10.244.1.146 

[root@k8s-master ~]# tcpdump -i flannel.1 -nn host 10.244.1.146
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on flannel.1, link-type EN10MB (Ethernet), capture size 262144 bytes

#宿主机ping后抓包情况如下：
22:22:35.737977 IP 10.244.0.0 > 10.244.1.146: ICMP echo request, id 29493, seq 1, length 64
22:22:35.738902 IP 10.244.1.146 > 10.244.0.0: ICMP echo reply, id 29493, seq 1, length 64
22:22:36.739042 IP 10.244.0.0 > 10.244.1.146: ICMP echo request, id 29493, seq 2, length 64
22:22:36.739789 IP 10.244.1.146 > 10.244.0.0: ICMP echo reply, id 29493, seq 2, length 64

#容器ping后抓包情况如下：
22:33:49.295137 IP 10.244.0.17 > 10.244.1.146: ICMP echo request, id 837, seq 1, length 64
22:33:49.295933 IP 10.244.1.146 > 10.244.0.17: ICMP echo reply, id 837, seq 1, length 64
22:33:50.296736 IP 10.244.0.17 > 10.244.1.146: ICMP echo request, id 837, seq 2, length 64
22:33:50.297222 IP 10.244.1.146 > 10.244.0.17: ICMP echo reply, id 837, seq 2, length 64
22:33:51.297556 IP 10.244.0.17 > 10.244.1.146: ICMP echo request, id 837, seq 3, length 64
22:33:51.298054 IP 10.244.1.146 > 10.244.0.17: ICMP echo reply, id 837, seq 3, length 64

可以看到报文都是经过flannel.1网络接口进入2层隧道进而转发
```

 VXLAN是Linux内核本身支持的一种网络虚拟化技术，是内核的一个模块，在内核态实现封装解封装，构建出覆盖网络，其实就是一个由各宿主机上的Flannel.1设备组成的虚拟二层网络。

 由于VXLAN由于额外的封包解包，导致其性能较差，所以Flannel就有了host-gw模式，即把宿主机当作网关，除了本地路由之外没有额外开销，性能和calico差不多，由于没有叠加来实现报文转发，这样会导致路由表庞大。因为一个节点对应一个网络，也就对应一条路由条目。

 host-gw虽然VXLAN网络性能要强很多。，但是种方式有个缺陷：要求各物理节点必须在同一个二层网络中。物理节点必须在同一网段中。这样会使得一个网段中的主机量会非常多，万一发一个广播报文就会产生干扰。在私有云场景下，宿主机不在同一网段是很常见的状态，所以就不能使用host-gw了。

 VXLAN还有另外一种功能，VXLAN也支持类似host-gw的玩法，如果两个节点在同一网段时使用host-gw通信，如果不在同一网段中，即 当前pod所在节点与目标pod所在节点中间有路由器，就使用VXLAN这种方式，使用叠加网络。 结合了Host-gw和VXLAN，这就是VXLAN的**Direct routing模式**

**Flannel VxLAN的Direct routing模式配置**

修改kube-flannel.yml文件，将flannel的configmap对象改为：

```
[root@k8s-master ~]# vim kube-flannel.yml 
......
 net-conf.json: |
    {
      "Network": "10.244.0.0/16",	#默认网段
      "Backend": {
        "Type": "vxlan",
        "Directrouting": true	#增加
      }
    }
......

[root@k8s-master ~]# kubectl apply -f kube-flannel.yml 
clusterrole.rbac.authorization.k8s.io/flannel configured
clusterrolebinding.rbac.authorization.k8s.io/flannel configured
serviceaccount/flannel unchanged
configmap/kube-flannel-cfg configured
daemonset.extensions/kube-flannel-ds-amd64 created
daemonset.extensions/kube-flannel-ds-arm64 created
daemonset.extensions/kube-flannel-ds-arm created
daemonset.extensions/kube-flannel-ds-ppc64le created
daemonset.extensions/kube-flannel-ds-s390x created

#查看路由信息
[root@k8s-master ~]# ip route
......
10.244.1.0/24 via 192.168.56.12 dev eth0 
10.244.2.0/24 via 192.168.56.13 dev eth0 
......
```

 从上面的结果可以看到，发往`10.244.1.0/24`和`10.244.1.0/24`的包都是直接经过`eth0`网络接口直接发出去的，这就是Directrouting。如果两个节点是跨网段的，则flannel自动降级为VxLAN模式。

 此时，在各 个 集群节点上执行“`iptables -nL”`命令 可以 看到， iptables filter 表 的 FORWARD 链 上 由其 生成 了 如下 两条 转发 规则， 它 显 式 放行 了`10. 244. 0. 0/ 16` 网络 进出 的 所有 报文， 用于 确保 由 物理 接口 接收 或 发送 的 目标 地址 或 源 地址 为`10. 244. 0. 0/ 16`网络 的 所有 报文 均能 够 正常 通行。 这些 是 Direct Routing 模式 得以 实现 的 必要条件：

```
target 		prot 	opt 	source 				destination 
ACCEPT 		all 	-- 	10. 244. 0. 0/ 16 		0. 0. 0. 0/ 0 
ACCEPT 		all 	-- 	0. 0. 0. 0/ 0 		    10. 244. 0. 0/ 16
```

 再在此之前创建的Pod和宿主机上进行ping测试，可以看到在flannel.1接口上已经抓不到包了，在eth0上可以用抓到ICMP的包，如下：

```
[root@k8s-master ~]# tcpdump -i flannel.1 -nn host 10.244.1.146
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on flannel.1, link-type EN10MB (Ethernet), capture size 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel

[root@k8s-master ~]# tcpdump -i eth0 -nn host 10.244.1.146
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
22:48:52.376393 IP 10.244.0.17 > 10.244.1.146: ICMP echo request, id 839, seq 1, length 64
22:48:52.376877 IP 10.244.1.146 > 10.244.0.17: ICMP echo reply, id 839, seq 1, length 64
22:48:53.377005 IP 10.244.0.17 > 10.244.1.146: ICMP echo request, id 839, seq 2, length 64
22:48:53.377621 IP 10.244.1.146 > 10.244.0.17: ICMP echo reply, id 839, seq 2, length 64

22:50:28.647490 IP 192.168.56.11 > 10.244.1.146: ICMP echo request, id 46141, seq 1, length 64
22:50:28.648320 IP 10.244.1.146 > 192.168.56.11: ICMP echo reply, id 46141, seq 1, length 64
22:50:29.648958 IP 192.168.56.11 > 10.244.1.146: ICMP echo request, id 46141, seq 2, length 64
22:50:29.649380 IP 10.244.1.146 > 192.168.56.11: ICMP echo reply, id 46141, seq 2, length 64
```

### 1.5、Host-gw后端

 Flannel除了上面2种数据传输的方式以外，还有一种是`host-gw`的方式，`host-gw`后端是通过添加必要的路由信息使用节点的二层网络直接发送Pod的通信报文。它的工作方式类似于Directrouting的功能，但是其并不具备VxLan的隧道转发能力。

 编辑kube-flannel的配置清单，将ConfigMap资源kube-flannel-cfg的data字段中网络配置进行修改，如下：

```
[root@k8s-master ~]# vim kube-flannel.yml 
......
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "host-gw"
      }
    }
......

[root@k8s-master ~]# kubectl apply -f kube-flannel.yml 
```

配置完成后，各节点会生成类似directrouting一样的 路由和iptables规则，用于实现二层转发Pod网络的通信报文，省去了隧道转发模式的额外开销。但是存在的问题点是，对于不在同一个二层网络的报文转发，`host-gw`是无法实现的。延续上面的例子，进行抓包查看：

master上的Pod：10.244.0.17向node01节点上的Pod：10.244.1.146发送ICMP报文

```
#查看路由表信息，可以看到其报文的发送方向都是和Directrouting是一样的
[root@k8s-master ~]# ip route
......
10.244.1.0/24 via 192.168.56.12 dev eth0 
10.244.2.0/24 via 192.168.56.13 dev eth0 
.....

#进行ping包测试
[root@k8s-master ~]# ping -c 2 10.244.1.146

#在eth0上进行抓包
[root@k8s-master ~]# tcpdump -i eth0 -nn host 10.244.1.146
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
23:11:05.556972 IP 192.168.56.11 > 10.244.1.146: ICMP echo request, id 59528, seq 1, length 64
23:11:05.557794 IP 10.244.1.146 > 192.168.56.11: ICMP echo reply, id 59528, seq 1, length 64
23:11:06.558231 IP 192.168.56.11 > 10.244.1.146: ICMP echo request, id 59528, seq 2, length 64
23:11:06.558610 IP 10.244.1.146 > 192.168.56.11: ICMP echo reply, id 59528, seq 2, length 64
```

 该模式下，报文转发的相关流程如下：

- 1、Pod（10.244.0.17）向Pod（10.244.1.146）发送报文，查看到报文中的目的地址为：10.244.1.146，并非本地网段，会直接发送到网关（192.168.56.11）；

- 2、网关发现该目标地址为10.244.1.146，要到达10.244.1.0/24网段，需要送达到node2 的物理网卡，node2接收以后发现该报文的目标地址属于本机上的另一个虚拟网卡，然后转发到相对应的Pod（10.244.1.146）

  工作模式流程图如下：

![img](https://img2018.cnblogs.com/blog/1349539/201903/1349539-20190313112934425-352124622.png)

 以上就是Flannel网络模型的三种工作模式，但是flannel自身并不具备为Pod网络实现网络策略和网络通信隔离的功能，为此只能借助于Calico联合统一的项目Calnal项目进行构建网络策略的功能。



## 二、网络策略

 网络策略（Network Policy ）是 Kubernetes 的一种资源。Network Policy 通过 Label 选择 Pod，并指定其他 Pod 或外界如何与这些 Pod 通信。

 Pod的网络流量包含流入（Ingress）和流出（Egress）两种方向。默认情况下，所有 Pod 是非隔离的，即任何来源的网络流量都能够访问 Pod，没有任何限制。当为 Pod 定义了 Network Policy，只有 Policy 允许的流量才能访问 Pod。

 Kubernetes的网络策略功能也是由第三方的网络插件实现的，因此，只有支持网络策略功能的网络插件才能进行配置网络策略，比如Calico、Canal、kube-router等等。

**PS：Kubernetes自1.8版本才支持Egress网络策略，在该版本之前仅支持Ingress网络策略。**

### 2.1、部署Canal提供网络策略功能

[calico官网](https://www.projectcalico.org/)

 Calico可以独立地为Kubernetes提供网络解决方案和网络策略，也可以和flannel相结合，由flannel提供网络解决方案，Calico仅用于提供网络策略，此时将Calico称为Canal。结合flannel工作时，Calico提供的默认配置清单式以flannel默认使用的10.244.0.0/16为Pod网络，因此在集群中kube-controller-manager启动时就需要通过--cluster-cidr选项进行设置使用该网络地址，并且---allocate-node-cidrs的值应设置为true。

```
#设置RBAC
[root@k8s-master ~]# kubectl apply -f https://docs.projectcalico.org/v3.2/getting-started/kubernetes/installation/hosted/canal/rbac.yaml

#部署Canal提供网络策略
[root@k8s-master ~]# kubectl apply -f https://docs.projectcalico.org/v3.2/getting-started/kubernetes/installation/hosted/canal/canal.yaml

[root@k8s-master ~]# kubectl get ds canal -n kube-system
NAME      DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
canal     3         3         0         3            0           beta.kubernetes.io/os=linux   2m

部署canal需要的镜像，建议先拉取镜像，避免耗死资源：
quay.io/calico/node:v3.2.6
quay.io/calico/cni:v3.2.6
quay.io/coreos/flannel:v0.9.1

[root@k8s-master ~]# kubectl get pods -n kube-system -o wide |grep canal
canal-2hqwt                            3/3       Running        0          1h        192.168.56.11   k8s-master
canal-c5pxr                            3/3       Running        0          1h        192.168.56.13   k8s-node02
canal-kr662                            3/3       Running        6          1h        192.168.56.12   k8s-node01
```

 `Canal`作为`DaemonSet`部署到每个节点，属于`kube-system`这个名称空间。需要注意的是，`Canal`只是直接使用了`Calico`和`flannel`项目，代码本身没有修改，`Canal`只是一种部署的模式，用于安装和配置项目。

### 2.2、配置网络策略

 在Kubernetes系统中，报文的流入和流出的核心组件是Pod资源，它们也是网络策略功能的主要应用对象。`NetworkPolicy`对象通过`podSelector`选择 一组Pod资源作为控制对象。`NetworkPolicy`是定义在一组Pod资源之上用于管理入站流量，或出站流量的一组规则，有可以是出入站规则一起生效，规则的生效模式通常由`spec.policyTypes`进行 定义。如下图：

![img](https://img2018.cnblogs.com/blog/1349539/201903/1349539-20190313170340204-1497786108.png)

 默认情况下，Pod对象的流量控制是为空的，报文可以自由出入。在附加网络策略之后，Pod对象会因为NetworkPolicy而被隔离，一旦名称空间中有任何NetworkPolicy对象匹配了某特定的Pod对象，则该Pod将拒绝NetworkPolicy规则中不允许的所有连接请求，但是那些未被匹配到的Pod对象依旧可以接受所有流量。

 就特定的Pod集合来说，入站和出站流量默认是放行状态，除非有规则可以进行匹配。还有一点需要注意的是，在`spec.policyTypes`中指定了生效的规则类型，但是在`networkpolicy.spec`字段中嵌套定义了没有任何规则的Ingress或Egress时，则表示拒绝入站或出站的一切流量。定义网络策略的基本格式如下：

```
apiVersion: networking.k8s.io/v1		#定义API版本
kind: NetworkPolicy					   #定义资源类型
metadata:
  name: allow-myapp-ingress			    #定义NetwokPolicy的名字
  namespace: default
spec:								  #NetworkPolicy规则定义
  podSelector: 						   #匹配拥有标签app:myapp的Pod资源
    matchLabels:
      app: myapp
  policyTypes ["Ingress"]			    #NetworkPolicy类型，可以是Ingress，Egress，或者两者共存
  ingress:							  #定义入站规则
  - from:
    - ipBlock:						  #定义可以访问的网段
        cidr: 10.244.0.0/16
        except:						  #排除的网段
        - 10.244.3.0/24
    - podSelector:					  #选定当前default名称空间，标签为app:myapp可以入站
        matchLabels:
          app: myapp
    ports:							 #开放的协议和端口定义
    - protocol: TCP
      port: 80
  
该网络策略就是将default名称空间中拥有标签"app=myapp"的Pod资源开放80/TCP端口给10.244.0.0/16网段，并排除10.244.3.0/24网段的访问，并且也开放给标签为app=myapp的所有Pod资源进行访问。  
```

为了看出Network Policy的效果，先部署一个httpd的应用。配置清单文件如下：

```
[root@k8s-master ~]# mkdir network-policy-demo
[root@k8s-master ~]# cd network-policy-demo/
[root@k8s-master network-policy-demo]# vim httpd.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: httpd
spec:
  replicas: 3
  template:
    metadata:
      labels:
        run: httpd
    spec:
      containers:
      - name: httpd
        image: httpd:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: httpd-svc
spec:
  type: NodePort
  selector:
    run: httpd
  ports:
  - protocol: TCP
    nodePort: 30000
    port: 8080
    targetPort: 80
```

创建三个副本，通过NodePort类型的Service对外方服务，部署应用：

```
[root@k8s-master network-policy-demo]# kubectl apply -f httpd.yaml 
deployment.apps/httpd unchanged
service/httpd-svc created
[root@k8s-master network-policy-demo]# kubectl get pods -o wide |grep httpd
httpd-75f655479d-882hz                1/1       Running    0          4m        10.244.0.2     k8s-master
httpd-75f655479d-h7lrr                1/1       Running    0          4m        10.244.2.2     k8s-node02
httpd-75f655479d-kzr5g                1/1       Running    0          4m        10.244.1.2     k8s-node01

[root@k8s-master network-policy-demo]# kubectl get svc httpd-svc
NAME        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
httpd-svc   NodePort   10.99.222.179   <none>        8080:30000/TCP   4m
```

当前没有定义任何Network Policy，验证应用的访问：

```
#启动一个busybox Pod，可以访问Service，也可以ping副本的Pod

[root@k8s-master ~]# kubectl run busybox --rm -it --image=busybox /bin/sh
If you don't see a command prompt, try pressing enter.
/ # wget httpd-svc:8080
Connecting to httpd-svc:8080 (10.99.222.179:8080)
index.html           100% |*********************************************************************************************|    45  0:00:00 ETA
/ # ping -c 2 10.244.1.2
PING 10.244.1.2 (10.244.1.2): 56 data bytes
64 bytes from 10.244.1.2: seq=0 ttl=63 time=0.507 ms
64 bytes from 10.244.1.2: seq=1 ttl=63 time=0.228 ms

--- 10.244.1.2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.228/0.367/0.507 ms


#集群节点也可以访问Sevice和ping通副本Pod
[root@k8s-node01 ~]# curl 10.99.222.179:8080
<html><body><h1>It works!</h1></body></html>
[root@k8s-node01 ~]# ping -c 2 10.244.2.2
PING 10.244.2.2 (10.244.2.2) 56(84) bytes of data.
64 bytes from 10.244.2.2: icmp_seq=1 ttl=63 time=0.931 ms
64 bytes from 10.244.2.2: icmp_seq=2 ttl=63 time=0.812 ms

--- 10.244.2.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.812/0.871/0.931/0.066 ms

#集群外部访问192.168.56.11:30000也是通的
[root@localhost ~]# curl 192.168.56.11:30000
<html><body><h1>It works!</h1></body></html>
```

那么下面再去设置不同的Network Policy来管控Pod的访问。

### 2.3、管控入站流量

NetworkPolicy资源属于名称空间级别，它的作用范围为其所属的名称空间。

**1、设置默认的Ingress策略**

用户可以创建一个NetworkPolicy来为名称空间设置一个默认的隔离策略，该策略选择所有的Pod对象，然后允许或拒绝任何到达这些Pod的入站流量，如下：

```
[root@k8s-master network-policy-demo]# vim policy-demo.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes: ["Ingress"]	
  #指明了Ingress生效规则，但不定义任何Ingress字段，因此不能匹配任何源端点，从而拒绝所有的入站流量

[root@k8s-master network-policy-demo]# kubectl apply -f policy-demo.yaml 
networkpolicy.networking.k8s.io/deny-all-ingress created
[root@k8s-master network-policy-demo]# kubectl get networkpolicy
NAME               POD-SELECTOR   AGE
deny-all-ingress   <none>         11s

#此时再去访问测试，是无法ping通，无法访问的
[root@k8s-master ~]# kubectl run busybox --rm -it --image=busybox /bin/sh
If you don't see a command prompt, try pressing enter.
/ # wget httpd-svc:8080
Connecting to httpd-svc:8080 (10.99.222.179:8080)
wget: can't connect to remote host (10.99.222.179): Connection timed out
```

如果要将默认策略设置为允许所有入站流量，只需要定义Ingress字段，并将这个字段设置为空，以匹配所有源端点，但本身不设定网络策略，就已经是默认允许所有入站流量访问的，下面给出一个定义的格式：

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
  ingress:
  - {}
```

实践中，通常将默认的网络策略设置为拒绝所有入站流量，然后再放行允许的源端点的入站流量。

**2、放行特定的入站流量**

```
[root@k8s-master network-policy-demo]# vim policy-demo.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-httpd
spec:
  podSelector: 
    matchLabels:
      run: httpd
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - ipBlock:
        cidr: 10.244.0.0/16
        except:
        - 10.244.2.0/24
        - 10.244.1.0/24
    - podSelector:
        matchLabels:
          access: "true"
    ports:
    - protocol: TCP
      port: 80

[root@k8s-master network-policy-demo]# kubectl apply -f policy-demo.yaml 
networkpolicy.networking.k8s.io/access-httpd created
[root@k8s-master network-policy-demo]# kubectl get networkpolicy
NAME           POD-SELECTOR   AGE
access-httpd   run=httpd      6s
```

验证NetworkPolicy的有效性：

```
#创建带有标签的busybox pod访问，是可以正常访问的，但是因为仅开放了TCP协议，所以PING是无法ping通的
[root@k8s-master ~]# kubectl run busybox --rm -it --labels="access=true" --image=busybox /bin/sh
If you don't see a command prompt, try pressing enter.
/ # wget httpd-svc:8080
Connecting to httpd-svc:8080 (10.99.222.179:8080)
index.html           100% |*********************************************************************************************|    45  0:00:00 ETA
/ # ping -c 3 10.244.0.2
PING 10.244.0.2 (10.244.0.2): 56 data bytes

--- 10.244.0.2 ping statistics ---
3 packets transmitted, 0 packets received, 100% packet loss
```

### 2.4、管控出站流量

通常，出站的流量默认策略应该是允许通过的，但是当有精细化需求，仅放行那些有对外请求需要的Pod对象的出站流量，也可以先为名称空间设置“禁止所有”的默认策略，再细化制定准许的策略。`networkpolicy.spec`中嵌套的Egress字段用来定义出站流量规则。

**1、设定默认Egress策略**

和Igress一样，只需要通过`policyTypes`字段指明生效的`Egress`类型规则，然后不去定义Egress字段，就不会去匹配到任何目标端点，从而拒绝所有的出站流量。

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
spec:
  podSelector: {}
  policyTypes: ["Egress"]
```

实践中，需要进行严格隔离的环境通常将默认的策略设置为拒绝所有出站流量，再去细化配置允许到达的目标端点的出站流量。

**2、放行特定的出站流量**

下面举个例子定义一个Egress规则，对标签`run=httpd`的Pod对象，到达标签为`access=true`的Pod对象的80端口的流量进行放行。

```
[root@k8s-master network-policy-demo]# vim egress-policy.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: httpd-egress
spec:
  podSelector: 
    matchLabels:
      run: httpd
  policyTypes: ["Egress"]
  egress:
  - to:
    - podSelector:
        matchLabels:
          access: "true"
    ports:
    - protocol: TCP
      port: 80


#NetworkPolicy检测，一个带有access=true标签，一个不带
[root@k8s-master ~]# kubectl run busybox --rm -it --labels="access=true" --image=busybox /bin/sh
If you don't see a command prompt, try pressing enter.
/ # wget httpd-svc:8080
Connecting to httpd-svc:8080 (10.99.222.179:8080)
index.html           100% |*********************************************************************************************|    45  0:00:00 ETA
/ # exit
Session ended, resume using 'kubectl attach busybox-686cb649b6-6j4qx -c busybox -i -t' command when the pod is running
deployment.apps "busybox" deleted

[root@k8s-master ~]# kubectl run busybox2 --rm -it --image=busybox /bin/sh
If you don't see a command prompt, try pressing enter.
/ # wget httpd-svc:8080
Connecting to httpd-svc:8080 (10.99.222.179:8080)
wget: can't connect to remote host (10.99.222.179): Connection timed out
```

从上面的检测结果可以看到，带有标签`access=true`的Pod才能访问到`httpd-svc`，说明上面配置的Network Policy已经生效。

### 2.5、隔离名称空间

 实践中，通常需要彼此隔离所有的名称空间，但是又需要允许它们可以和`kube-system`名称空间中的Pod资源进行流量交换，以实现监控和名称解析等各种管理功能。下面的配置清单示例在`default`名称空间定义相关规则，在出站和入站都默认均为拒绝的情况下，它用于放行名称空间内部的各Pod对象之间的通信，以及和`kube-system`名称空间内各Pod间的通信。

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-deny-all
  namespace: default
spec:
  policyTypes: ["Ingress","Egress"]
  podSelector: {}
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-allow
  namespace: default
spec:
  policyTypes: ["Ingress","Egress"]
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchExpressions:
        - key: name
          operator: In
          values: ["default","kube-system"]
  egress:
  - to:
    - namespaceSelector:
        matchExpressions:
        - key: name
          operator: In
          values: ["default","kube-system"]
```

需要注意的是，有一些额外的系统附件可能会单独部署到独有的名称空间中，比如将`prometheus`监控系统部署到prom名称空间等，这类具有管理功能的附件所在的名称空间和每一个特定的名称空间的出入流量也是需要被放行的。





# Pod资源调度

API Server在接受客户端提交Pod对象创建请求后，然后是通过调度器（kube-schedule）从集群中选择一个可用的最佳节点来创建并运行Pod。而这一个创建Pod对象，在调度的过程当中有3个阶段：节点预选、节点优选、节点选定，从而筛选出最佳的节点。如图：

![img](https://img2018.cnblogs.com/blog/1349539/201903/1349539-20190315110744164-1442880527.png)

- 节点预选：基于一系列的预选规则对每个节点进行检查，将那些不符合条件的节点过滤，从而完成节点的预选
- 节点优选：对预选出的节点进行优先级排序，以便选出最合适运行Pod对象的节点
- 节点选定：从优先级排序结果中挑选出优先级最高的节点运行Pod，当这类节点多于1个时，则进行随机选择

当我们有需求要将某些Pod资源运行在特定的节点上时，我们可以通过组合节点标签，以及Pod标签或标签选择器来匹配特定的预选策略并完成调度，如`MatchInterPodAfinity、MatchNodeSelector、PodToleratesNodeTaints`等预选策略，这些策略常用于为用户提供自定义Pod亲和性或反亲和性、节点亲和性以及基于污点及容忍度的调度机制。

## 1、常用的预选策略

预选策略实际上就是节点过滤器，例如节点标签必须能够匹配到Pod资源的标签选择器（MatchNodeSelector实现的规则），以及Pod容器的资源请求量不能大于节点上剩余的可分配资源（PodFitsResource规则）等等。执行预选操作，调度器会逐一根据规则进行筛选，如果预选没能选定一个合适的节点，此时Pod会一直处于Pending状态，直到有一个可用节点完成调度。其常用的预选策略如下：

- CheckNodeCondition：检查是否可以在节点报告磁盘、网络不可用或未准备好的情况下将Pod对象调度其上。
- HostName：如果Pod对象拥有spec.hostname属性，则检查节点名称字符串是否和该属性值匹配。
- PodFitsHostPorts：如果Pod对象定义了ports.hostPort属性，则检查Pod指定的端口是否已经被节点上的其他容器或服务占用。
- MatchNodeSelector：如果Pod对象定义了spec.nodeSelector属性，则检查节点标签是否和该属性匹配。
- NoDiskConflict：检查Pod对象请求的存储卷在该节点上可用。
- PodFitsResources：检查节点上的资源（CPU、内存）可用性是否满足Pod对象的运行需求。
- PodToleratesNodeTaints：如果Pod对象中定义了spec.tolerations属性，则需要检查该属性值是否可以接纳节点定义的污点（taints）。
- PodToleratesNodeNoExecuteTaints：如果Pod对象定义了spec.tolerations属性，检查该属性是否接纳节点的NoExecute类型的污点。
- CheckNodeLabelPresence：仅检查节点上指定的所有标签的存在性，要检查的标签以及其可否存在取决于用户的定义。
- CheckServiceAffinity：根据当前Pod对象所属的Service已有其他Pod对象所运行的节点调度，目前是将相同的Service的Pod对象放在同一个或同一类节点上。
- MaxEBSVolumeCount：检查节点上是否已挂载EBS存储卷数量是否超过了设置的最大值，默认值：39
- MaxGCEPDVolumeCount：检查节点上已挂载的GCE PD存储卷是否超过了设置的最大值，默认值：16
- MaxAzureDiskVolumeCount：检查节点上已挂载的Azure Disk存储卷数量是否超过了设置的最大值，默认值：16
- CheckVolumeBinding：检查节点上已绑定和未绑定的PVC是否满足Pod对象的存储卷需求。
- NoVolumeZoneConflct：在给定了区域限制的前提下，检查在该节点上部署Pod对象是否存在存储卷冲突。
- CheckNodeMemoryPressure：在给定了节点已经上报了存在内存资源压力过大的状态，则需要检查该Pod是否可以调度到该节点上。
- CheckNodePIDPressure：如果给定的节点已经报告了存在PID资源压力过大的状态，则需要检查该Pod是否可以调度到该节点上。
- CheckNodeDiskPressure：如果给定的节点存在磁盘资源压力过大，则检查该Pod对象是否可以调度到该节点上。
- MatchInterPodAffinity：检查给定的节点能否可以满足Pod对象的亲和性和反亲和性条件，用来实现Pod亲和性调度或反亲和性调度。

在上面的这些预选策略里面，CheckNodeLabelPressure和CheckServiceAffinity可以在预选过程中结合用户自定义调度逻辑，这些策略叫做可配置策略。其他不接受参数进行自定义配置的称为静态策略。

## 2、优选函数

预选策略筛选出一个节点列表就会进入优选阶段，在这个过程调度器会向每个通过预选的节点传递一系列的优选函数来计算其优先级分值，优先级分值介于0-10之间，其中0表示不适用，10表示最适合托管该Pod对象。

另外，调度器还支持给每个优选函数指定一个简单的值，表示权重，进行节点优先级分值计算时，它首先将每个优选函数的计算得分乘以权重，然后再将所有优选函数的得分相加，从而得出节点的最终优先级分值。权重可以让管理员定义优选函数倾向性的能力，其计算优先级的得分公式如下：

```
finalScoreNode = (weight1 * priorityFunc1) + (weight2 * priorityFunc2) + ......
```

下图是关于优选函数的列表图：

![img](https://img2018.cnblogs.com/blog/1349539/201903/1349539-20190315110812120-2048644331.png)

## 3、节点亲和调度

节点亲和性是用来确定Pod对象调度到哪一个节点的规则，这些规则基于节点上的自定义标签和Pod对象上指定的标签选择器进行定义。

定义节点亲和性规则有2种：硬亲和性（require）和软亲和性（preferred）

- 硬亲和性：实现的是强制性规则，是Pod调度时必须满足的规则，否则Pod对象的状态会一直是Pending
- 软亲和性：实现的是一种柔性调度限制，在Pod调度时可以尽量满足其规则，在无法满足规则时，可以调度到一个不匹配规则的节点之上。

定义节点亲和规则的两个要点：一是节点配置是否合乎需求的标签，而是Pod对象定义合理的标签选择器，这样才能够基于标签选择出期望的目标节点。

需要注意的是`preferredDuringSchedulingIgnoredDuringExecution`和`requiredDuringSchedulingIgnoredDuringExecution`名字中后半段字符串`IgnoredDuringExecution`表示的是，在Pod资源基于节点亲和性规则调度到某个节点之后，如果节点的标签发生了改变，调度器不会讲Pod对象从该节点上移除，因为该规则仅对新建的Pod对象有效。

### 3.1、节点硬亲和性

下面的配置清单中定义的Pod对象，使用节点硬亲和性和规则定义将当前Pod调度到标签为zone=foo的节点上：

```
apiVersion: v1
kind: Pod
metadata:
  name: with-require-nodeaffinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - {key: zone,operator: In,values: ["foo"]}
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1

#创建Pod对象
[root@k8s-master ~]# kubectl apply -f require-nodeAffinity-pod.yaml 
pod/with-require-nodeaffinity created

#由于集群中并没有节点含有节点标签为zone=foo，所以创建的Pod一直处于Pending状态
[root@k8s-master ~]# kubectl get pods with-require-nodeaffinity
NAME                        READY     STATUS    RESTARTS   AGE
with-require-nodeaffinity   0/1       Pending   0          35s

#查看Pending具体的原因
[root@k8s-master ~]# kubectl describe pods with-require-nodeaffinity
......
Events:
  Type     Reason            Age               From               Message
  ----     ------            ----              ----               -------
  Warning  FailedScheduling  3s (x21 over 1m)  default-scheduler  0/3 nodes are available: 3 node(s) didn't match node selector.

#给node01节点打上zone=foo的标签，可以看到成功调度到node01节点上
[root@k8s-master ~]# kubectl label node k8s-node01 zone=foo
node/k8s-node01 labeled
[root@k8s-master ~]# kubectl describe pods with-require-nodeaffinity
Events:
  Type     Reason            Age                From                 Message
  ----     ------            ----               ----                 -------
  Warning  FailedScheduling  58s (x25 over 2m)  default-scheduler    0/3 nodes are available: 3 node(s) didn't match node selector.
  Normal   Pulled            4s                 kubelet, k8s-node01  Container image "ikubernetes/myapp:v1" already present on machine
  Normal   Created           4s                 kubelet, k8s-node01  Created container
  Normal   Started           4s                 kubelet, k8s-node01  Started container

[root@k8s-master ~]# kubectl get pods with-require-nodeaffinity -o wide
NAME                        READY     STATUS    RESTARTS   AGE       IP            NODE
with-require-nodeaffinity   1/1       Running   0          6m        10.244.1.12   k8s-node01
```

在定义节点亲和性时，`requiredDuringSchedulingIgnoredDuringExecution`字段的值是一个对象列表，用于定义节点硬亲和性，它可以由一个或多个`nodeSelectorTerms`定义的对象组成，此时值需要满足其中一个`nodeSelectorTerms`即可。

而`nodeSelectorTerms`用来定义节点选择器的条目，它的值也是一个对象列表，由1个或多个`matchExpressions`对象定义的匹配规则组成，多个规则是逻辑与的关系，这就表示某个节点的标签必须要满足同一个`nodeSelectorTerms`下所有的`matchExpression`对象定义的规则才能够成功调度。如下：

```
#如下配置清单，必须存在满足标签zone=foo和ssd=true的节点才能够调度成功
apiVersion: v1
kind: Pod
metadata:
  name: with-require-nodeaffinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - {key: zone, operator: In, values: ["foo"]}
          - {key: ssd, operator: Exists, values: []}	#增加一个规则
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1

[root@k8s-master ~]# kubectl apply -f require-nodeAffinity-pod.yaml 
pod/with-require-nodeaffinity created
[root@k8s-master ~]# kubectl get pods with-require-nodeaffinity  
NAME                        READY     STATUS    RESTARTS   AGE
with-require-nodeaffinity   0/1       Pending   0          16s
[root@k8s-master ~]# kubectl label node k8s-node01 ssd=true
node/k8s-node01 labeled

[root@k8s-master ~]# kubectl get pods with-require-nodeaffinity  
NAME                        READY     STATUS    RESTARTS   AGE
with-require-nodeaffinity   1/1       Running   0          2m
```

在预选策略中，还可以通过节点资源的可用性去限制能够成功调度，如下配置清单：要求的资源为6核心CPU和20G内存，节点是无法满足该容器的资源需求，因此也会调度失败，Pod资源会处于Pending状态。

```
apiVersion: v1
kind: Pod
metadata:
  name: with-require-nodeaffinity
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
    resources:
      requests:
        cpu: 6
        memory: 20Gi
```

### 3.2、节点软亲和性

看下面一个配置清单：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deploy-with-node-affinity
spec:
  replicas: 5
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
      spec:
        affinity:
          nodeAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 60
              preference:
                matchExpressions:
                - {key: zone, operator: In, values: ["foo"]}
            - weight: 30
              preference:
                matchExpressions:
                - {key: ssd, operator: Exists, values: []}
        containers:
        - name: myapp
          image: ikubernetes/myapp:v1
          
#首先先给node02和master都打上标签，master标签为zone=foo，node02标签为ssd=true,这里node01是没有对应标签的
[root@k8s-master ~]# kubectl label node k8s-node02 ssd=true
node/k8s-node02 labeled
[root@k8s-master ~]# kubectl label node k8s-master zone=foo
node/k8s-master labeled

#进行创建
[root@k8s-master ~]# kubectl apply -f deploy-with-preferred-nodeAffinity.yaml 
deployment.apps/myapp-deploy-with-node-affinity2 created

#可以看到5个Pod分别分布在不同的节点上，node01上没有对应的标签也会调度上进行创建Pod，体现软亲和性
[root@k8s-master ~]# kubectl get pods -o wide |grep deploy-with
myapp-deploy-with-node-affinity2-75b8f65f87-2gqjv   1/1       Running    0          11s       10.244.2.4     k8s-node02
myapp-deploy-with-node-affinity2-75b8f65f87-7l2sg   1/1       Running    0          11s       10.244.0.4     k8s-master
myapp-deploy-with-node-affinity2-75b8f65f87-cdrxx   1/1       Running    0          11s       10.244.2.3     k8s-node02
myapp-deploy-with-node-affinity2-75b8f65f87-j77f6   1/1       Running    0          11s       10.244.1.36    k8s-node01
myapp-deploy-with-node-affinity2-75b8f65f87-wt6tq   1/1       Running    0          11s       10.244.0.3     k8s-master

#如果我们给node01打上zone=foo,ssd=true的标签，再去创建时，你会发现所有的Pod都调度在这个节点上。因为节点的软亲和性，会尽力满足Pod中定义的规则，如下：
[root@k8s-master ~]# kubectl label node k8s-node01 zone=foo
[root@k8s-master ~]# kubectl label node k8s-node01 ssd=true
[root@k8s-master ~]# kubectl get pods -o wide |grep deploy-with
myapp-deploy-with-node-affinity2-75b8f65f87-4lwsw   0/1       ContainerCreating   0          3s        <none>         k8s-node01
myapp-deploy-with-node-affinity2-75b8f65f87-dxbxf   1/1       Running             0          3s        10.244.1.31    k8s-node01
myapp-deploy-with-node-affinity2-75b8f65f87-lnhgm   0/1       ContainerCreating   0          3s        <none>         k8s-node01
myapp-deploy-with-node-affinity2-75b8f65f87-snxbc   0/1       ContainerCreating   0          3s        <none>         k8s-node01
myapp-deploy-with-node-affinity2-75b8f65f87-zx8ck   1/1       Running             0          3s        10.244.1.33    k8s-node01
```

上面的实验结果显示，当2个标签没有都存在一个node节点上时，Pod对象会被分散在集群中的三个节点上进行创建并运行，之所以如此，是因为使用了 节点软亲和性的预选方式，所有的节点都能够通过`MatchNodeSelector`预选策略的筛选。当我们将2个标签都集合在node01上时，所有Pod对象都会运行在node01之上。

## 4、Pod资源亲和调度

在出于高效通信的需求，有时需要将一些Pod调度到相近甚至是同一区域位置（比如同一节点、机房、区域）等等，比如业务的前端Pod和后端Pod，此时这些Pod对象之间的关系可以叫做亲和性。

同时出于安全性的考虑，也会把一些Pod之间进行隔离，此时这些Pod对象之间的关系叫做反亲和性（anti-affinity）。

调度器把第一个Pod放到任意位置，然后和该Pod有亲和或反亲和关系的Pod根据该动态完成位置编排，这就是Pod亲和性和反亲和性调度的作用。Pod的亲和性定义也存在硬亲和性和软亲和性的区别，其约束的意义和节点亲和性类似。

Pod的亲和性调度要求各相关的Pod对象运行在同一位置，而反亲和性则要求它们不能运行在同一位置。这里的位置实际上取决于节点的位置拓扑，拓扑的方式不同，Pod是否在同一位置的判定结果也会有所不同。

如果基于各个节点的`kubernetes.io/hostname`标签作为评判标准，那么会根据节点的`hostname`去判定是否在同一位置区域。

### 4.1、Pod硬亲和度

Pod强制约束的亲和性调度也是使用`requiredDuringSchedulingIgnoredDuringExecution`进行定义的。Pod亲和性是用来描述一个Pod对象和现有的Pod对象运行的位置存在某种依赖关系，所以如果要测试Pod亲和性约束，需要存在一个被依赖的Pod对象，下面创建一个带有`app=tomcat`的Deployment资源部署一个Pod对象：

```
[root@k8s-master ~]# kubectl run tomcat -l app=tomcat --image=tomcat:alpine
deployment.apps/tomcat created
[root@k8s-master ~]# kubectl get pods -l app=tomcat -o wide
NAME                      READY     STATUS    RESTARTS   AGE       IP            NODE
tomcat-75fd5cc757-w9qdb   1/1       Running   0          5m        10.244.1.37   k8s-node01
```

从上面我们可以看到新创建的`tomcat`pod对象被调度在k8s-node01上，再写一个配置清单定义一个Pod对象，通过`labelSelector`定义的标签选择器挑选对应的Pod对象。

```
[root@k8s-master ~]# vim required-podAffinity-pod1.yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity-1
spec:
  affinity:
    podAffninity:
      requiredDuringSchedulingIngnoreDuringExecution:
      - labelSelector:
          matchExpression:
          - {key: app , operator: In , values: ["tomcat"]}
        topologyKey: kubernetes.io/hostname
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
```

`kubernetes.io/hostname`标签是Kubernetes集群节点的内建标签，它的值为当前节点的主机名，对于各个节点来说都是不同的。所以新建的Pod对象要被部署到和`tomcat`所在的同一个节点上。

```
[root@k8s-master ~]# kubectl apply -f required-podAffinity-pod1.yaml 
pod/with-pod-affinity-1 created

[root@k8s-master ~]# kubectl get pods -o wide
NAME                      READY     STATUS    RESTARTS   AGE         IP              NODE
with-pod-affinity-1       1/1       Running    0          31s       10.244.1.38    k8s-node01
tomcat-75fd5cc757-w9qdb   1/1       Running    0          5m        10.244.1.37    k8s-node01
```

基于单一节点的Pod亲和性相对来说使用的情况会比较少，通常使用的是基于同一地区、区域、机架等拓扑位置约束。比如部署应用程序（myapp）和数据库（db）服务相关的Pod时，这两种Pod应该部署在同一区域上，可以加速通信的速度。

### 4.2、Pod软亲和度

同理，有硬亲和度即有软亲和度，Pod也支持使用`preferredDuringSchedulingIgnoredDuringExecuttion`属性进行定义Pod的软亲和性，调度器会尽力满足亲和约束的调度，在满足不了约束条件时，也允许将该Pod调度到其他节点上运行。比如下面这一配置清单：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-with-preferred-pod-affinity
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      name: myapp
      labels:
        app: myapp
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
         - weight: 80
           podAffinityTerm:
             labelSelector:
               matchExpressions:
               - {key: app, operator: In , values: ["cache"]}
             topologyKey: zone
         - weight: 20
           podAffinityTerm:
             labelSelector:
               matchExpressions:
               - {key: app, operator: In, values: ["db"]}
             topologyKey: zone
       containers:
       - name: myapp
         image: ikubernetes/mapp:v1
```

上述的清单配置当中，pod的软亲和调度需要将Pod调度到标签为`app=cache`并在区域zone当中，或者调度到`app=db`标签节点上的，但是我们的节点上并没有类似的标签，所以调度器会根据软亲和调度进行随机调度到`k8s-node01`节点之上。如下：

```
[root@k8s-master ~]# kubectl apply -f deploy-with-preferred-podAffinity.yaml 
deployment.apps/myapp-with-preferred-pod-affinity created
[root@k8s-master ~]# kubectl get pods -o wide |grep myapp-with-preferred-pod-affinity  
myapp-with-preferred-pod-affinity-5c44649f58-cwgcd   1/1       Running    0          1m        10.244.1.40    k8s-node01
myapp-with-preferred-pod-affinity-5c44649f58-hdk8q   1/1       Running    0          1m        10.244.1.42    k8s-node01
myapp-with-preferred-pod-affinity-5c44649f58-kg7cx   1/1       Running    0          1m        10.244.1.41    k8s-node01
```

### 4.3、Pod反亲和度

`podAffinity`定义了Pod对象的亲和约束，而Pod对象的反亲和调度则是用`podAntiAffinty`属性进行定义，下面的配置清单中定义了由同一Deployment创建但是彼此基于节点位置互斥的Pod对象：

```
[root@k8s-master ~]# cat deploy-with-required-podAntiAffinity.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-with-pod-anti-affinity
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      name: myapp
      labels:
        app: myapp
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - {key: app,operator: In,values: ["myapp"]}
            topologyKey: kubernetes.io/hostname
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1

[root@k8s-master ~]# kubectl apply -f deploy-with-required-podAntiAffinity.yaml 
deployment.apps/myapp-with-pod-anti-affinity created
[root@k8s-master ~]# kubectl get pods -l app=myapp
NAME                                            READY     STATUS    RESTARTS   AGE
myapp-with-pod-anti-affinity-79c7b6c596-77hrz   1/1       Running   0          8s
myapp-with-pod-anti-affinity-79c7b6c596-fhxmv   0/1       Pending   0          8s
myapp-with-pod-anti-affinity-79c7b6c596-l9ckr   1/1       Running   0          8s
myapp-with-pod-anti-affinity-79c7b6c596-vfv2s   1/1       Running   0          8s
```

由于在配置清单中定义了强制性反亲和性，所以创建的4个Pod副本必须 运行在不同的节点当中呢，但是集群中只存在3个节点，因此，肯定会有一个Pod对象处于Pending的状态。

## 5、污点和容忍度

污点（taints）是定义在节点上的一组键值型属性数据，用来让节点拒绝将Pod调度到该节点上，除非该Pod对象具有容纳节点污点的容忍度。而容忍度（tolerations）是定义在Pod对象上的键值型数据，用来配置让Pod对象可以容忍节点的污点。

前面的节点选择器和节点亲和性的调度方式都是通过在Pod对象上添加标签选择器来完成对特定类型节点标签的匹配，实现的是Pod选择节点的方式。而污点和容忍度则是通过对节点添加污点信息来控制Pod对象的调度结果，让节点拥有了控制哪种Pod对象可以调度到该节点上的 一种方式。

Kubernetes使用PodToleratesNodeTaints预选策略和TaintTolerationPriority优选函数来完成这种调度方式。

### 5.1、定义污点和容忍度

污点的定义是在节点的nodeSpec，而容忍度的定义是在Pod中的podSpec，都属于键值型数据，两种方式都支持一个`effect`标记，语法格式为`key=value: effect`，其中key和value的用户和格式和资源注解类似，而`effect`是用来定义对Pod对象的排斥等级，主要包含以下3种类型：

- NoSchedule：不能容忍此污点的新Pod对象不能调度到该节点上，属于强制约束，节点现存的Pod对象不受影响。
- PreferNoSchedule：NoSchedule属于柔性约束，即不能容忍此污点的Pod对象尽量不要调度到该节点，不过无其他节点可以调度时也可以允许接受调度。
- NoExecute：不能容忍该污点的新Pod对象不能调度该节点上，强制约束，节点现存的Pod对象因为节点污点变动或Pod容忍度的变动导致无法匹配规则，Pod对象就会被从该节点上去除。

在Pod对象上定义容忍度时，其支持2中操作符：`Equal`和`Exists`

- Equal：等值比较，表示容忍度和污点必须在key、value、effect三者之上完全匹配。
- Exists：存在性判断，表示二者的key和effect必须完全匹配，而容忍度中的value字段使用空值。

在使用kubeadm部署的集群中，master节点上将会自动添加污点信息，阻止不能容忍该污点的Pod对象调度到该节点上，如下：

```
[root@k8s-master ~]# kubectl describe node k8s-master
Name:               k8s-master
Roles:              master
......
Taints:             node- role. kubernetes. io/ master: NoSchedule
......
```

而一些系统级别的应用在创建时，就会添加相应的容忍度来确保被创建时可以调度到master节点上，如flannel插件：

```
[root@k8s-master ~]# kubectl describe pods kube-flannel-ds-amd64-2p8wm -n kube-system
......
Tolerations:     node-role.kubernetes.io/master:NoSchedule
                 node.kubernetes.io/disk-pressure:NoSchedule
                 node.kubernetes.io/memory-pressure:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute
                 node.kubernetes.io/unreachable:NoExecute
......
```

### 5.2、管理节点的污点

**使用命令行向节点添加污点**

```
语法：kubectl taint nodes <nodename> <key>=<value>:<effect>......

#定义k8s-node01上的污点
[root@k8s-master ~]# kubectl taint nodes k8s-node01 node-type=production:NoSchedule
node/k8s-node01 tainted

#查看节点污点信息
[root@k8s-master ~]# kubectl get nodes k8s-node01 -o go-template={{.spec.taints}}
[map[effect:NoSchedule key:node-type value:production]]
```

此时，node01节点上已经存在的Pod对象不受影响，仅对新建Pod对象有影响，需要注意的是，如果是同一个键值数据，但是最后的标识不同，也是属于不同的污点信息，比如再给node01上添加一个污点的标识为：`PreferNoSchedule`

```
[root@k8s-master ~]# kubectl taint nodes k8s-node01 node-type=production:PreferNoSchedule
node/k8s-node01 tainted

[root@k8s-master ~]# kubectl get nodes k8s-node01 -o go-template={{.spec.taints}}
[map[value:production effect:PreferNoSchedule key:node-type] map[key:node-type value:production effect:NoSchedule]]
```

**删除污点**

```
语法：kubectl taint nodes <node-name> <key>[: <effect>]-

#删除node01上的node-type标识为NoSchedule的污点
[root@k8s-master ~]# kubectl taint nodes k8s-node01 node-type:NoSchedule-
node/k8s-node01 untainted

#删除指定键名的所有污点
[root@k8s-master ~]# kubectl taint nodes k8s-node01 node-type-
node/k8s-node01 untainted

#补丁方式删除节点上的全部污点信息
[root@k8s-master ~]# kubectl patch nodes k8s-node01 -p '{"spec":{"taints":[]}}'
```

### 5.3、Pod对象的容忍度

Pod对象的容忍度可以通过`spec.tolerations`字段进行添加，同一的也有两种操作符：`Equal`和`Exists`方式。Equal等值方式如下：

```
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "Noexecute"
  tolerationSeconds: 3600
```

Exists方式如下：

```
tolerations: 
- key: "key1" 
  operator: "Exists" 
  effect: "NoExecute" 
  tolerationSeconds: 3600
```



# Helm包管理器

## 1、Helm的概念和架构

每个成功的软件平台都有一个优秀的打包系统，比如 Debian、Ubuntu 的 apt，Redhat、Centos 的 yum。而 Helm 则是 Kubernetes 上的包管理器。

Helm 到底解决了什么问题？为什么 Kubernetes 需要 Helm？

Kubernetes 能够很好地组织和编排容器，但它缺少一个更高层次的应用打包工具，而 Helm 就是来干这件事的。

举个例子，我们需要部署一个MySQL服务，Kubernetes则需要部署以下对象：

​	① 为了能够让外界访问到MySQL，需要部署一个mysql的service；

​	②需要进行定义MySQL的密码，则需要部署一个Secret；

​	③Mysql的运行需要持久化的数据存储，此时还需要部署PVC；

​	④保证后端mysql的运行，还需要部署一个Deployment，以支持以上的对象。

针对以上对象，我们可以使用YAML文件进行定义并部署，但是仅仅对于单个的服务支持，如果应用需要由一个甚至几十个这样的服务组成，并且还需要考虑各种服务的依赖问题，可想而知，这样的组织管理应用的方式就显得繁琐。为此就诞生了一个工具Helm，就是为了解决Kubernetes这种应用部署繁重的现象。

Helm的核心术语：

- Chart：一个helm程序包，是创建一个应用的信息集合，包含各种Kubernetes对象的配置模板、参数定义、依赖关系、文档说明等。可以将Chart比喻为yum中的软件安装包；
- Repository：Charts仓库，用于集中存储和分发Charts；
- Release：特定的Chart部署于目标集群上的一个实例，代表这一个正在运行的应用。当chart被安装到Kubernetes集群，就会生成一个release，chart可以多次安装到同一个集群，每次安装都是一个release。

Helm的程序架构：

Helm的架构很简单，由一个客户端Helm Client，一个服务端Helm Library，两者都是使用go语言编写，如下图：

- Helm Client： Helm的命令行客户端helm cli，用于helm的操作
- Helm Library： helm的服务端，通kuberents api进行交互，用于接收客户端的指令并执行部署，更新，删除和版本的管理。

简单的说：Helm 客户端负责管理 chart；服务器负责管理 release。在Helm3中移除了Tiller, 版本相关的数据直接存储在了Kubernetes中。

## 2、部署Helm

[helm部署文档](https://docs.helm.sh/using_helm/#quickstart-guide)

Helm的部署方式有两种：预编译的二进制程序和源码编译安装，这里使用二进制的方式进行安装

### （1）下载helm

```bash
[root@k8s-master ~]# wget https://get.helm.sh/helm-v3.4.0-linux-amd64.tar.gz 
[root@k8s-master ~]# tar -xf helm-v3.4.0-linux-amd64.tar.gz 
[root@k8s-master ~]# cd linux-amd64/
[root@k8s-master linux-amd64]# ls
helm  LICENSE  README.md
[root@k8s-master linux-amd64]# mv helm /usr/bin
[root@k8s-master linux-amd64]# helm version
version.BuildInfo{Version:"v3.4.0", GitCommit:"7090a89efc8a18f3d8178bf47d2462450349a004", GitTreeState:"clean", GoVersion:"go1.14.10"}
```

## 3、helm的使用

官方可用的Chart列表：

[https://hub.kubeapps.com](https://hub.kubeapps.com/)

```
helm常用命令：
- helm search:    搜索charts
- helm pull:     下载charts到本地目录
- helm install:   安装charts
- helm list:      列出charts的所有版本

用法:
  helm [command]

命令可用选项:
  completion  为指定的shell生成自动补全脚本（bash或zsh）
  create      创建一个新的charts
  delete      删除指定版本的release
  dependency  管理charts的依赖
  pull       下载charts并解压到本地目录
  get         下载一个release
  history     release历史信息
  home        显示helm的家目录
  init        在客户端和服务端初始化helm
  inspect     查看charts的详细信息
  install     安装charts
  lint        检测包的存在问题
  list        列出release
  package     将chart目录进行打包
  plugin      add(增加), list（列出）, or remove（移除） Helm 插件
  repo        add(增加), list（列出）, remove（移除）, update（更新）, and index（索引） chart仓库
  reset       卸载tiller
  rollback    release版本回滚
  search      关键字搜索chart
  serve       启动一个本地的http server
  status      查看release状态信息
  template    本地模板
  test        release测试
  upgrade     release更新
  verify      验证chart的签名和有效期
  version     打印客户端和服务端的版本信息
```

Charts是Helm的程序包，它们都存在在Charts仓库当中。Kubernetes官方的仓库保存了一系列的Charts，仓库默认的名称为`stable`。安装Charts到集群时，Helm首先会到官方仓库获取相关的Charts，并创建release。可执行 `helm search repo` 查看当前可安装的 chart 。

```
[root@k8s-master ~]# helm search repo
NAME                          	CHART VERSION	APP VERSION  	DESCRIPTION                                       
stable/acs-engine-autoscaler  	2.1.3        	2.1.1        	Scales worker nodes within agent pools            
stable/aerospike              	0.1.7        	v3.14.1.2    	A Helm chart for Aerospike in Kubernetes          
stable/anchore-engine         	0.1.3        	0.1.6        	Anchore container analysis and policy evaluatio...
......
```

这些 chart 都是从哪里来的？

前面说过，Helm 可以像 yum 管理软件包一样管理 chart。 yum 的软件包存放在仓库中，同样的，Helm 也有仓库。

```
[root@k8s-master ~]# helm repo list
NAME  	URL                                                   
local 	http://127.0.0.1:8879/charts                          
stable	https://kubernetes-charts.storage.googleapis.com
```

Helm 安装时已经默认配置好了两个仓库：`stable` 和 `local`。`stable` 是官方仓库，`local` 是用户存放自己开发的`chart`的本地仓库。可以通过`helm repo list`进行查看。

```shell
[root@k8s-master helm]# helm repo remove stable #移除stable repo
"stable" has been removed from your repositories
[root@k8s-master helm]# helm repo list
NAME 	URL                         
local	http://127.0.0.1:8879/charts

#增加阿里云的charts仓库
[root@k8s-master helm]# helm repo add stable https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts 
"stable" has been added to your repositories
[root@k8s-master helm]# helm repo list
NAME  	URL                                                   
local 	http://127.0.0.1:8879/charts                          
stable	https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
[root@k8s-master helm]# helm repo update #再次更新repo
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈ 
```

与 yum 一样，helm 也支持关键字搜索：

```shell
[root@k8s-master ~]# helm search repo mysql
NAME                         	CHART VERSION	APP VERSION	DESCRIPTION                                       
stable/mysql                 	0.3.5        	           	Fast, reliable, scalable, and easy to use open-...
stable/percona               	0.3.0        	           	free, fully compatible, enhanced, open source d...
stable/percona-xtradb-cluster	0.0.2        	5.7.19     	free, fully compatible, enhanced, open source d...
stable/gcloud-sqlproxy       	0.2.3        	           	Google Cloud SQL Proxy                            
stable/mariadb               	2.1.6        	10.1.31    	Fast, reliable, scalable, and easy to use open-...
```

包括 DESCRIPTION 在内的所有信息，只要跟关键字匹配，都会显示在结果列表中。

安装 chart 也很简单，执行如下命令可以安装 MySQL。

```shell
#helm安装mysql
[root@k8s-master helm]# helm install my-mysql stable/mysql
NAME: my-mysql
LAST DEPLOYED: Mon Mar  1 17:53:47 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
my-mysql-mysql.default.svc.cluster.local

To get your root password run:

    MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default my-mysql-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

To connect to your database:

1. Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

2. Install the mysql client:

    $ apt-get update && apt-get install mysql-client -y

3. Connect using the mysql cli, then provide your password:
    $ mysql -h my-mysql-mysql -p

To connect to your database directly from outside the K8s cluster:
    MYSQL_HOST=127.0.0.1
    MYSQL_PORT=3306

    # Execute the following commands to route the connection:
    export POD_NAME=$(kubectl get pods --namespace default -l "app=my-mysql-mysql" -o jsonpath="{.items[0].metadata.name}")
    kubectl port-forward $POD_NAME 3306:3306

    mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}
```

输出分为三部分：

- ① chart 本次部署的描述信息：

`NAME` 是 release 的名字，因为我们没用 `-n` 参数指定，Helm 随机生成了一个，这里是 `reeling-bronco`。

`NAMESPACE` 是 release 部署的 namespace，默认是 `default`，也可以通过 `--namespace` 指定。

`STATUS` 为 `DEPLOYED`，表示已经将 chart 部署到集群。

- ② 当前 release 包含的资源：Service、Deployment、Secret 和 PersistentVolumeClaim，其名字都是 `my-mysql-mysql`，命名的格式为 `ReleasName`-`ChartName`。
- ③ `NOTES` 部分显示的是 release 的使用方法。比如如何访问 Service，如何获取数据库密码，以及如何连接数据库等。

通过 `kubectl get` 可以查看组成 release 的各个对象：

```shell
[root@k8s-master helm]# kubectl get service my-mysql-mysql
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
my-mysql-mysql   ClusterIP   10.99.245.169   <none>        3306/TCP   3m

[root@k8s-master helm]# kubectl get deployment my-mysql-mysql
NAME                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
my-mysql-mysql   1         1         1            0           3m

[root@k8s-master helm]# kubectl get pvc  my-mysql-mysql
NAME                   STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-mysql-mysql   Pending                                                      4m

[root@k8s-master helm]# kubectl get secret  my-mysql-mysql
NAME                   TYPE      DATA      AGE
my-mysql-mysql   Opaque    2         4m
```

由于我们还没有准备 PersistentVolume，当前 release 还不可用。

`helm list` 显示已经部署的 release，`helm delete` 可以删除 release。

```shell
[root@k8s-master helm]# helm list
NAME          	REVISION	UPDATED                 	STATUS  	CHART      	NAMESPACE
my-mysql	1       	Wed Mar 27 03:10:31 2019	DEPLOYED	mysql-0.3.5	default  

[root@k8s-master helm]# helm delete my-mysql
release "my-mysql" deleted
```

## 4、chart 目录结构

chart 是 Helm 的应用打包格式。chart 由一系列文件组成，这些文件描述了 Kubernetes 部署应用时所需要的资源，比如 Service、Deployment、PersistentVolumeClaim、Secret、ConfigMap 等。

单个的 chart 可以非常简单，只用于部署一个服务，比如 Memcached；chart 也可以很复杂，部署整个应用，比如包含 HTTP Servers、 Database、消息中间件、cache 等。

chart 将这些文件放置在预定义的目录结构中，通常整个 chart 被打成 tar 包，而且标注上版本信息，便于 Helm 部署。

以前面 MySQL chart 为例。一旦安装了某个 chart，我们就可以在 ~/.cache/helm/repository/ 中找到 chart 的 tar 包。

```
[root@k8s-master ~]# cd .cache/helm/repository/
[root@k8s-master archive]# ll
-rw-r--r-- 1 root root 5536 Oct 29 22:04 mysql-0.3.5.tgz
-rw-r--r-- 1 root root 6189 Oct 29 05:03 redis-1.1.15.tgz
[root@k8s-master archive]# tar -xf mysql-0.3.5.tgz
[root@k8s-master archive]# tree mysql
mysql
├── Chart.yaml	
├── README.md	
├── templates
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── NOTES.txt
│   ├── pvc.yaml
│   ├── secrets.yaml
│   └── svc.yaml
└── values.yaml	
```

- Chart.yaml：YAML 文件，描述 chart 的概要信息。
- README.md：Markdown 格式的 README 文件，相当于 chart 的使用文档，此文件为可选。
- LICENSE：文本文件，描述 chart 的许可信息，此文件为可选。
- requirements.yaml ：chart 可能依赖其他的 chart，这些依赖关系可通过 requirements.yaml 指定。
- values.yaml：chart 支持在安装的时根据参数进行定制化配置，而 values.yaml 则提供了这些配置参数的默认值。
- templates目录：各类 Kubernetes 资源的配置模板都放置在这里。Helm 会将 values.yaml 中的参数值注入到模板中生成标准的 YAML 配置文件。
- templates/NOTES.txt：chart 的简易使用文档，chart 安装成功后会显示此文档内容。 与模板一样，可以在 NOTE.txt 中插入配置参数，Helm 会动态注入参数值。

## 5、chart模板

Helm 通过模板创建 Kubernetes 能够理解的 YAML 格式的资源配置文件，我们将通过例子来学习如何使用模板。

以 `templates/secrets.yaml` 为例：



![img](https://img2018.cnblogs.com/blog/1349539/201903/1349539-20190327162832186-1379196654.png)



从结构上看，文件的内容和我们在定义Secret的配置上大致相似，只是大部分的属性值变成了{{ xxx }}。这些{{ xx }}实际上是模板的语法。Helm采用了Go语言的模板来编写chart。

- ① `{{ template "mysql.fullname" . }}` 定义 Secret 的 `name`。

关键字 `template` 的作用是引用一个子模板 `mysql.fullname`。这个子模板是在 `templates/_helpers.tpl` 文件中定义的。

![img](https://img2018.cnblogs.com/blog/1349539/201903/1349539-20190327162923902-149922480.png)

这个定义还是很复杂的，因为它用到了模板语言中的对象、函数、流控制等概念。现在看不懂没关系，这里我们学习的重点是：如果存在一些信息多个模板都会用到，则可在 `templates/_helpers.tpl` 中将其定义为子模板，然后通过 `templates` 函数引用。

**这里 `mysql.fullname` 是由 release 与 chart 二者名字拼接组成。**

根据 chart 的最佳实践，所有资源的名称都应该保持一致，对于我们这个 chart，无论 Secret 还是 Deployment、PersistentVolumeClaim、Service，它们的名字都是子模板 `mysql.fullname` 的值。

- ② `Chart` 和 `Release` 是 Helm 预定义的对象，每个对象都有自己的属性，可以在模板中使用。如果使用下面命令安装 chart：

```
[root@k8s-master templates]# helm search repo stable/mysql
NAME        	CHART VERSION	APP VERSION	DESCRIPTION                                       
stable/mysql	0.3.5        	           	Fast, reliable, scalable, and easy to use open-...
[root@k8s-master templates]# helm install my stable/mysql 
```

那么：

`{{ .Chart.Name }}` 的值为 `mysql`。

`{{ .Chart.Version }}`的值为`0.3.5`

`{{ .Release.Name }}` 的值为 `my`。

`{{ .Release.Service }}` 始终取值为 `Helm`，如果安装的helm2 此处值为`Tiller`。

`{{ template "mysql.fullname" . }}` 计算结果为 `my-mysql`。

- ③ 这里指定 `mysql-root-password` 的值，不过使用了 `if-else` 的流控制，其逻辑为：

如果 `.Values.mysqlRootPassword` 有值，则对其进行 base64 编码；否则随机生成一个 10 位的字符串并编码。

`Values` 也是预定义的对象，代表的是 `values.yaml` 文件。而 `.Values.mysqlRootPassword` 则是 `values.yaml` 中定义的 `mysqlRootPassword` 参数：

![img](https://img2018.cnblogs.com/blog/1349539/201903/1349539-20190327163221224-2049393973.png)

因为 `mysqlRootPassword` 被注释掉了，没有赋值，所以逻辑判断会走 `else`，即随机生成密码。

`randAlphaNum`、`b64enc`、`quote` 都是 Go 模板语言支持的函数，函数之间可以通过管道 `|` 连接。`{{ randAlphaNum 10 | b64enc | quote }}` 的作用是首先随机产生一个长度为 10 的字符串，然后将其 base64 编码，最后两边加上双引号。

`templates/secrets.yaml` 这个例子展示了 chart 模板主要的功能，我们最大的收获应该是：模板将 chart 参数化了，通过 `values.yaml` 可以灵活定制应用。

无论多复杂的应用，用户都可以用 Go 模板语言编写出 chart。无非是使用到更多的函数、对象和流控制

## 6、定制安装MySQL chart

### （1）chart安装准备

作为准备工作，安装之前需要先清楚 chart 的使用方法。这些信息通常记录在 values.yaml 和 README.md 中。除了下载源文件查看，执行 `helm inspect values` 可能是更方便的方法。

```shell
[root@k8s-master ~]# helm inspect values stable/mysql
## mysql image version
## ref: https://hub.docker.com/r/library/mysql/tags/
##
image: "mysql"
imageTag: "5.7.14"

## Specify password for root user
##
## Default: random 10 character string
# mysqlRootPassword: testing

## Create a database user
##
# mysqlUser:
# mysqlPassword:

## Allow unauthenticated access, uncomment to enable
##
# mysqlAllowEmptyPassword: true
......
```

输出的实际上是 values.yaml 的内容。阅读注释就可以知道 MySQL chart 支持哪些参数，安装之前需要做哪些准备。其中有一部分是关于存储的：

```shell
## Persist data to a persistent volume
persistence:
  enabled: true
  ## database data Persistent Volume Storage Class
  ## If defined, storageClassName: <storageClass>
  ## If set to "-", storageClassName: "", which disables dynamic provisioning
  ## If undefined (the default) or set to null, no storageClassName spec is
  ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
  ##   GKE, AWS & OpenStack)
  ##
  # storageClass: "-"
  accessMode: ReadWriteOnce
  size: 8Gi
```

chart 定义了一个 PersistentVolumeClaim，申请 8G 的 PersistentVolume。由于我们的实验环境不支持动态供给，所以得预先创建好相应的 PV，其配置文件 `mysql-pv.yml` 内容为：

```shell
[root@k8s-master volumes]# pwd
/root/mainfests/volumes
[root@k8s-master volumes]# cat mysql-pv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv2
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 8Gi
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /data/volume/db
    server: stor01

[root@k8s-master volumes]# kubectl apply -f mysql-pv.yaml 
persistentvolume/mysql-pv2 created

[root@k8s-master volumes]# kubectl get pv
NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS        CLAIM                       STORAGECLASS   REASON    AGE
mysql-pv2   8Gi        RWO            Retain           Available                                                          5s
```

### （2）定制化安装chart

除了接受 values.yaml 的默认值，我们还可以定制化 chart，比如设置 `mysqlRootPassword`。

Helm 有两种方式传递配置参数：

1. 指定自己的 values 文件。
   通常的做法是首先通过 `helm inspect values mysql > myvalues.yaml`生成 values 文件，然后设置 `mysqlRootPassword`，之后执行 `helm install --values=myvalues.yaml mysql`。
2. 通过 `--set` 直接传入参数值，比如：

```
[root@k8s-master ~]# helm install stable/mysql --set mysqlRootPassword=abc123 -n my
NAME:   my
LAST DEPLOYED: Tue Oct 30 22:55:22 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Secret
NAME      TYPE    DATA  AGE
my-mysql  Opaque  2     1s

==> v1/PersistentVolumeClaim
NAME      STATUS  VOLUME     CAPACITY  ACCESS MODES  STORAGECLASS  AGE
my-mysql  Bound   mysql-pv2  8Gi       RWO           1s

==> v1/Service
NAME      TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)   AGE
my-mysql  ClusterIP  10.103.41.193  <none>       3306/TCP  1s

==> v1beta1/Deployment
NAME      DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
my-mysql  1        1        1           0          1s

==> v1/Pod(related)
NAME                       READY  STATUS   RESTARTS  AGE
my-mysql-79b5d4fdcd-bj4mt  0/1    Pending  0         0s
......
```

`mysqlRootPassword` 设置为 `abc123`。另外，`-n` 设置 release 为 `my`，各类资源的名称即为`my-mysql`。

通过 `helm list` 和 `helm status` 可以查看 chart 的最新状态。

### （3）升级和回滚release

release 发布后可以执行 `helm upgrade` 对其升级，通过 `--values` 或 `--set`应用新的配置。比如将当前的 MySQL 版本升级到 5.7.15：

```shell
[root@k8s-master ~]# helm upgrade --set imageTag=5.7.15 repo stable/mysql
Release "my" has been upgraded. Happy Helming!
LAST DEPLOYED: Tue Oct 30 23:42:36 2018
......
[root@k8s-master ~]# kubectl get deployment my-mysql -o wide
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS   IMAGES         SELECTOR
my-mysql   1         1         1            0           11m       my-mysql     mysql:5.7.15   app=my-mysql
```

`helm history` 可以查看 release 所有的版本。通过 `helm rollback` 可以回滚到任何版本。

```shell
[root@k8s-master ~]# helm history my
REVISION	UPDATED                 	STATUS    	CHART      	DESCRIPTION     
1       	Tue Oct 30 23:31:42 2018	SUPERSEDED	mysql-0.3.5	Install complete
2       	Tue Oct 30 23:42:36 2018	DEPLOYED  	mysql-0.3.5	Upgrade complete
[root@k8s-master ~]# helm rollback my 1
Rollback was a success! Happy Helming!
回滚成功，MySQL 恢复到 5.7.14。

[root@k8s-master ~]# helm history my
REVISION	UPDATED                 	STATUS    	CHART      	DESCRIPTION     
1       	Tue Oct 30 23:31:42 2018	SUPERSEDED	mysql-0.3.5	Install complete
2       	Tue Oct 30 23:42:36 2018	SUPERSEDED	mysql-0.3.5	Upgrade complete
3       	Tue Oct 30 23:44:28 2018	DEPLOYED  	mysql-0.3.5	Rollback to 1   
[root@k8s-master ~]# kubectl get deployment my-mysql -o wide
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS   IMAGES         SELECTOR
my-mysql   1         1         1            1           13m       my-mysql     mysql:5.7.14   app=my-mysql
```



## 7、自定义chart

Kubernetes 给我们提供了大量官方 chart，不过要部署微服务应用，还是需要开发自己的 chart，下面就来实践这个主题。

### （1）创建chart

执行 `helm create mychart` 的命令创建 chart `mychart`：

```shell
[root@k8s-master ~]# helm create -h
[root@k8s-master ~]# helm create mychart
Creating mychart
[root@k8s-master ~]# tree mychart/
mychart/
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── ingress.yaml
│   ├── NOTES.txt
│   └── service.yaml
└── values.yaml
```

Helm 会帮我们创建目录 `mychart`，并生成了各类 chart 文件。这样我们就可以在此基础上开发自己的 chart 了。

### （2）调试chart

只要是程序就会有 bug，chart 也不例外。Helm 提供了 debug 的工具：`helm lint` 和 `helm install --dry-run --debug`。

`helm lint` 会检测 chart 的语法，报告错误以及给出建议。 故意修改mychart中的value.yaml，进行检测：

`helm lint mychart` 会指出这个语法错误。

```shell
[root@k8s-master ~]# helm lint mychart
==> Linting mychart
[INFO] Chart.yaml: icon is recommended
[ERROR] values.yaml: unable to parse YAML
	error converting YAML to JSON: yaml: line 11: could not find expected ':'

Error: 1 chart(s) linted, 1 chart(s) failed

mychart 目录被作为参数传递给 helm lint。错误修复后则能通过检测。

[root@k8s-master ~]# helm lint mychart
==> Linting mychart
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, no failures
```

`helm install --dry-run --debug` 会模拟安装 chart，并输出每个模板生成的 YAML 内容。

```shell
[root@k8s-master ~]# helm install sad-orangutan ./mychart/ --dry-run  --debug
[debug] Created tunnel using local port: '46807'

[debug] SERVER: "127.0.0.1:46807"

[debug] Original chart version: ""
[debug] CHART PATH: /root/mychart

NAME:   sad-orangutan
REVISION: 1
RELEASED: Wed Oct 31 01:55:03 2018
CHART: mychart-0.1.0
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
affinity: {}
image:
  pullPolicy: IfNotPresent
  repository: nginx
  tag: stable
ingress:
  annotations: {}
  enabled: false
  hosts:
  - chart-example.local
  path: /
  tls: []
nodeSelector: {}
replicaCount: 1
resources: {}
service:
  port: 80
  type: ClusterIP
tolerations: []

HOOKS:
MANIFEST:

---
# Source: mychart/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: sad-orangutan-mychart
  labels:
    app: mychart
    chart: mychart-0.1.0
    release: sad-orangutan
    heritage: Tiller
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: mychart
    release: sad-orangutan
---
# Source: mychart/templates/deployment.yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: sad-orangutan-mychart
  labels:
    app: mychart
    chart: mychart-0.1.0
    release: sad-orangutan
    heritage: Tiller
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mychart
      release: sad-orangutan
  template:
    metadata:
      labels:
        app: mychart
        release: sad-orangutan
    spec:
      containers:
        - name: mychart
          image: "nginx:stable"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
            {}
```

我们可以检视这些输出，判断是否与预期相符。

### （3）安装chart

安装 chart，Helm 支持四种安装方法：

1. 安装仓库中的 chart，例如：`helm install dev-nginx stable/nginx`
2. 通过 tar 包安装，例如：`helm install dev-nginx ./nginx-1.2.3.tgz`
3. 通过 chart 本地目录安装，例如：`helm install dev-nginx ./nginx`
4. 通过 URL 安装，例如：`helm install dev-nginx https://example.com/charts/nginx-1.2.3.tgz`

这里通过使用本地目录进行安装：

```shell
[root@k8s-master ~]# helm install anxious-wasp ./mychart/
NAME:   anxious-wasp
LAST DEPLOYED: Wed Oct 31 01:57:15 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Pod(related)
NAME                                 READY  STATUS             RESTARTS  AGE
anxious-wasp-mychart-94fcbf7d-dg5qn  0/1    ContainerCreating  0         0s

==> v1/Service
NAME                  TYPE       CLUSTER-IP    EXTERNAL-IP  PORT(S)  AGE
anxious-wasp-mychart  ClusterIP  10.111.51.71  <none>       80/TCP   0s

==> v1beta2/Deployment
NAME                  DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
anxious-wasp-mychart  1        1        1           0          0s


NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app=mychart,release=anxious-wasp" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:80
```

当 chart 部署到 Kubernetes 集群，便可以对其进行更为全面的测试。

### （4）将chart添加到仓库

chart 通过测试后可以将其添加到仓库，团队其他成员就能够使用。任何 HTTP Server 都可以用作 chart 仓库，下面演示在 `k8s-node1`192.168.56.12 上搭建仓库。

```
（1）在 k8s-node1 上启动一个 httpd 容器。
[root@k8s-node01 ~]# mkdir /var/www
[root@k8s-node01 ~]# docker run -d -p 8080:80 -v /var/www/:/usr/local/apache2/htdocs/ httpd

（2）通过 helm package 将 mychart 打包。
[root@k8s-master ~]# helm package mychart
Successfully packaged chart and saved it to: /root/mychart-0.1.0.tgz

（3）执行 helm repo index 生成仓库的 index 文件
[root@k8s-master ~]# mkdir myrepo
[root@k8s-master ~]# mv mychart-0.1.0.tgz myrepo/
[root@k8s-master ~]# 
[root@k8s-master ~]# helm repo index myrepo/ --url http://192.168.56.12:8080/charts
[root@k8s-master ~]# ls myrepo/
index.yaml  mychart-0.1.0.tgz

Helm 会扫描 myrepo 目录中的所有 tgz 包并生成 index.yaml。--url指定的是新仓库的访问路径。新生成的 index.yaml 记录了当前仓库中所有 chart 的信息：
当前只有 mychart 这一个 chart。

[root@k8s-master ~]# cat myrepo/index.yaml 
apiVersion: v1
entries:
  mychart:
  - apiVersion: v1
    appVersion: "1.0"
    created: 2018-10-31T02:02:45.599264611-04:00
    description: A Helm chart for Kubernetes
    digest: 08abeb3542e8a9ab90df776d3a646199da8be0ebfc5198ef032190938d49e30a
    name: mychart
    urls:
    - http://192.168.56.12:8080/charts/mychart-0.1.0.tgz
    version: 0.1.0
generated: 2018-10-31T02:02:45.598450525-04:00

（4）将 mychart-0.1.0.tgz 和 index.yaml 上传到 k8s-node1 的 /var/www/charts 目录。
[root@k8s-master myrepo]# scp ./* root@k8s-node01:/var/www/charts/
[root@k8s-node01 ~]# ls /var/www/charts/
index.yaml  mychart-0.1.0.tgz

（5）通过 helm repo add 将新仓库添加到 Helm。
[root@k8s-master ~]# helm repo add newrepo http://192.168.56.12:8080/charts
"newrepo" has been added to your repositories
[root@k8s-master ~]# helm repo list
NAME   	URL                                                   
local  	http://127.0.0.1:8879/charts                          
stable 	https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
newrepo	http://192.168.56.12:8080/charts                      

（6）现在已经可以 repo search 到 mychart 了。
[root@k8s-master ~]# helm search mychart
NAME           	CHART VERSION	APP VERSION	DESCRIPTION                
local/mychart  	0.1.0        	1.0        	A Helm chart for Kubernetes
newrepo/mychart	0.1.0        	1.0        	A Helm chart for Kubernetes

除了 newrepo/mychart，这里还有一个 local/mychart。这是因为在执行第 2 步打包操作的同时，mychart 也被同步到了 local 的仓库。

（7）已经可以直接从新仓库安装 mychart 了。
[root@k8s-master ~]# helm install newrepo/mychart

（8）如果以后仓库添加了新的 chart，需要用 helm repo update 更新本地的 index。
[root@k8s-master ~]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "newrepo" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈ 

这个操作相当于 Centos 的 yum update。
```

## 8、总结

- Helm是Kubernetes的包管理器，Helm 让我们能够像 yum 管理 rpm 包那样安装、部署、升级和删除容器化应用。
- Helm 由客户端和 Tiller 服务器组成。客户端负责管理 chart，服务器负责管理 release。
- chart 是 Helm 的应用打包格式，它由一组文件和目录构成。其中最重要的是模板，模板中定义了 Kubernetes 各类资源的配置信息，Helm 在部署时通过 values.yaml 实例化模板。
- Helm 允许用户开发自己的 chart，并为用户提供了调试工具。用户可以搭建自己的 chart 仓库，在团队中共享 chart。



# kubernetes CRD

官方文档：https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/

kubernetes 有两种机制供用户扩展API

第一种：**CRD**，复用Kubernetes的API Server，无须编写额外的APIServer。用户只需要定义CRD，并且提供一个CRD控制器，就能通过Kubernetes的API管理自定义资源对象了，同时要求用户的CRD对象符合API Server的管理规范。

第二种：**使用API聚合机制** ，用户需要编写额外的API Server，可以对资源进行更细粒度的控制（例如，如何在各API版本之间切换），要求用户自行处理对多个API版本的支持。

## CRD 定义：

CRD 的全称是 Custom Resource Definition。顾名思义，它指的就是，允许用户在 Kubernetes 中添加一个跟 Pod、Node 类似的、新的 API 资源类型，即：自定义 API 资源。

**Custom resources**：

> 是对K8S API的扩展，代表了一个特定的kubetnetes的定制化安装。在一个运行中的集群中，自定义资源可以动态注册到集群中。注册完毕以后，用户可以通过kubelet创建和访问这个自定义的对象，类似于操作pod一样。

**Custom controllers** 自定义控制器:

> Custom resources可以让用户简单的存储和获取结构化数据。只有结合控制器才能变成一个真正的declarative API（被声明过的API）。控制器可以把资源更新成用户想要的状态，并且通过一系列操作维护和变更状态。定制化控制器是用户可以在运行中的集群内部署和更新的一个控制器，它独立于集群本身的生命周期。
> 定制化控制器可以和任何一种资源一起工作，当和定制化资源结合使用时尤其有效。

**Operator模式**

> 是一个customer controllers和Custom resources结合的例子。它可以允许开发者将特殊应用编码至kubernetes的扩展API内。

通俗理解：CR即创建一个自定义资源，CRD是对资源的描述，CC是提供CRD对象的管理。

CRD本身只是一段声明，用于定义用户自定义的资源对象。**但仅有CRD的定义并没有实际作用，用户还需要提供管理CRD对象的CRD控制器（CRD Controller），才能实现对CRD对象的管理**。CRD控制器通常可以通过Go语言进行开发，并需要遵循Kubernetes的控制器开发规范，基于客户端库client-go进行开发，需要实现Informer 、ResourceEventHandler、Workqueue等组件具体的功能处理逻辑，

## 以官网的例子说明：

以下就是创建一个CronTab 资源。其中 Group: stable.example.com ,version:v1,resource:CronTab。而这个yaml文件，就是一个具体的“自定义API资源”，也叫CR（**Custom resources**），而为了让kubernetes 能够认识这个CR，你就需要让kubernetes 明白这个CR的宏观定义是什么，即CRD（Custom Resource Definition）

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
# Deprecated in v1.16 in favor of apiextensions.k8s.io/v1
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # list of versions supported by this CustomResourceDefinition
  versions:
    - name: v1
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
  # either Namespaced or Cluster
  #该API的生效范围，可选项为Namespaced（由Namespace限定）和Cluster（在集群范围全局生效，不局限于任何Namespace），默认值为Namespaced。
  scope: Namespaced 
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    # 复数形式的名称，要求全部小写
    plural: crontabs 
    # singular name to be used as an alias on the CLI and for display
    # 单数形式的名称，要求全部小写
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    # CRD的资源类型名称，要求以驼峰式命名规范进行命名
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    # 缩写形式的名称，要求全部小写
    shortNames:
    - ct
  preserveUnknownFields: false
  # CRD的校验（Validation）机制
  validation:
    openAPIV3Schema:
      type: object
      properties:
        spec:
          type: object
          properties:
            cronSpec:
              type: string
            image:
              type: string
            replicas:
              type: integer
```

## CRD的高级用途

### 1、CRD的subresources子资源

Kubernetes从1.11版本开始，在CRD的定义中引入了名为subresources的配置，可以设置的选项包括status和scale两类。

- stcatus：启用/status路径，其值来自CRD的.status字段，要求CRD控制器能够设置和更新这个字段的值。
- scale：启用/scale路径，支持通过其他Kubernetes控制器（如HorizontalPodAutoscaler控制器）与CRD资源对象实例进行交互。用户通过kubectl scale命令也能对该CRD资源对象进行扩容或缩容操作，要求CRD本身支持以多个副本的形式运行。

```
subresources:
    status: {}
```

### 2、CRD的校验（Validation）机制

```
validation:
    openAPIV3Schema:
      type: object
      properties:
        spec:
          type: object
          properties:
            cronSpec:
              type: string
            image:
              type: string
            replicas:
              type: integer
```

### 3、自定义查看CRD时需要显示的列

```
 # 自定义查看CRD时需要显示的列，即kubectl get 命令能够显示的字段
  additionalPrinterColumns:
  - JSONPath: .status.conditions[-1:].type
    name: State
    type: string
  - JSONPath: .metadata.creationTimestamp
    name: Age
    type: date
```

### 4、Finalizer（CRD资源对象的预删除钩子方法）

Finalizer设置的方法在删除CRD资源对象时进行调用，以实现CRD资源对象的清理工作。

接下来，好好研究怎么编写Custom controllers
参考：https://blog.csdn.net/weixin_41806245/article/details/94451734





# KIC实现gRPC服务访问



 GRPC是Google开源的一个高性能RPC通信框架，其通过Protocol Buffers作为其IDL，可以在不同语言开发的平台上使用，同时基于HTTP/2协议实现，继而提供了连接多路复用、头部压缩、流控等特性，极大地提高了客户端与服务端的通信效率。

### gRPC简介

gRPC是Google开源的一个高性能RPC通信框架，通过[Protocol Buffers](https://it.baiked.com/wp-content/themes/begin/inc/go.php?url=https://developers.google.com/protocol-buffers/)作为其IDL，可以在不同语言开发的平台上使用，同时基于HTTP/2协议实现，继而提供了连接多路复用、头部压缩、流控等特性，极大地提高了客户端与服务端的通信效率。

![K8S Ingress Controller实现gRPC服务访问](https://it.baiked.com/wp-content/uploads/2018/06/436b89c6ba63e71ed49ed114307f7511f33c1aa0-300x137.png)

在gRPC里客户端应用可以像本地方法调用一样可以调用到位于不同服务器上的服务端应用方法，你可以很方便地创建分布式应用和服务。同其他RPC框架一样，gRPC也需要定义一个服务接口，同时指定其能够被远程调用的方法和返回类型，服务端实现这个接口，同时起一个gRPC Server来处理客户端请求，而客户端则存在一个与服务端方法一样的存根。

### 环境准备

1. 搭建k8s集群
2. 安装grpcurl工具，具体可参考[这里](https://it.baiked.com/wp-content/themes/begin/inc/go.php?url=https://github.com/fullstorydev/grpcurl)
3. gRPC访问需要Ingress Controller 0.15.0-1及以上支持，具体可参考[发布公告](https://it.baiked.com/wp-content/themes/begin/inc/go.php?url=https://yq.aliyun.com/articles/598075)

### gRPC服务示例

定义如下服务接口，客户端可调用helloworld.Greeter服务的SayHello接口：

```
option java_multiple_files = true;
option java_package = "io.grpc.examples.helloworld";
option java_outer_classname = "HelloWorldProto";

package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

具体测试示例可参考[这里](https://it.baiked.com/wp-content/themes/begin/inc/go.php?url=https://github.com/grpc/grpc-go/tree/master/examples)。

### 部署示例服务

###### 1、部署gRPC服务：

```
[root@k8s-master ~]# vi grpc-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grpc-service
spec:
  replicas: 1
  selector: 
    matchLabels: 
      app: grpc-service
  template:
    metadata:
      labels:
        app: grpc-service
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/acs-sample/grpc-server:latest
        imagePullPolicy: IfNotPresent
        name: grpc-service
        ports:
        - containerPort: 50051
          protocol: TCP
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: grpc-service
spec:
  ports:
  - port: 50051
    protocol: TCP
    targetPort: 50051
  selector:
    app: grpc-service
  sessionAffinity: None
  type: NodePort
[root@k8s-master ~]# kubectl apply -f grpc-service.yaml
deployment "grpc-service" created
service "grpc-service" created
```

###### 2、创建SSL证书：

```
[root@k8s-master ~]# openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=grpc.example.com/O=grpc.example.com"
[root@k8s-master ~]# kubectl create secret tls grpc-secret --key tls.key --cert tls.crt
secret "grpc-secret" created
```

###### 3、配置Ingress路由规则：

```
[root@k8s-master ~]# vi grpc-ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: grpc-ingress
  annotations:
    # 注意这里：必须要配置以指明后端服务为gRPC服务
    nginx.ingress.kubernetes.io/grpc-backend: "true"
spec:
  tls:
    - hosts:
      # 证书域名
      - grpc.example.com
      secretName: grpc-secret
  rules:
  # gRPC服务域名
  - host: grpc.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: grpc-service
          servicePort: 50051
[root@k8s-master ~]# kubectl apply -f grpc-ingress.yaml
ingress "grpc-ingress" created
```

###### 4、测试gRPC服务访问：

###### 4.1、查看gRPC服务端提供的所有服务：

```
$ grpcurl -insecure grpc.example.com:443 list
grpc.reflection.v1alpha.ServerReflection
helloworld.Greeter
```

这里我们可以看到服务端提供了helloworld.Greeter服务

###### 4.2、查看helloworld.Greeter服务的接口方法：

```
$ grpcurl -insecure grpc.example.com:443 list helloworld.Greeter
SayHello
```

这里可以看到helloworld.Greeter服务提供了SayHello方法

###### 4.3、描述SayHello方法的具体协议参数：

```
$ grpcurl -insecure grpc.example.com:443 describe helloworld.Greeter.SayHello
helloworld.Greeter.SayHello is a method:
{
  "name": "SayHello",
  "inputType": ".helloworld.HelloRequest",
  "outputType": ".helloworld.HelloReply",
  "options": {

  }
}
```

###### 4.4、调用SayHello方法并传递name参数：

```
$ grpcurl -insecure -d '{"name": "gRPC"}' grpc.example.com:443 helloworld.Greeter.SayHello
{
  "message": "Hello gRPC"
}
$ grpcurl -insecure -d '{"name": "world"}' grpc.example.com:443 helloworld.Greeter.SayHello
{
  "message": "Hello world"
}
```



### 灰度发布gRPC服务

[通过阿里云容器服务K8S Ingress Controller实现应用服务的灰度发布](https://it.baiked.com/wp-content/themes/begin/inc/go.php?url=https://yq.aliyun.com/articles/594019)一文详细阐述了如何通过Ingress Controller来实现应用的灰度发布，当然我们也支持gRPC服务的灰度发布。
注意：由于[nginx](https://it.baiked.com/tag/nginx/) grpc_pass的限制，目前对于gRPC服务暂不支持service-weight配置

###### 1、部署新版本gRPC服务：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grpc-service-2
spec:
  replicas: 1
  selector: 
    matchLabels: 
      app: grpc-service
  template:
    metadata:
      labels:
        run: grpc-service-2
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/acs-sample/grpc-server:latest-2
        imagePullPolicy: Always
        name: grpc-service-2
        ports:
        - containerPort: 50051
          protocol: TCP
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: grpc-service-2
spec:
  ports:
  - port: 50051
    protocol: TCP
    targetPort: 50051
  selector:
    run: grpc-service-2
  sessionAffinity: None
  type: NodePort
[root@k8s-master ~]# kubectl apply -f grpc-service-2.yml
deployment "grpc-service-2" created
service "grpc-service-2" created
```

###### 2、修改Ingress路由规则：

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: grpc-ingress
  annotations:
    # 注意这里：必须要配置以指明后端服务为gRPC服务
    nginx.ingress.kubernetes.io/grpc-backend: "true"
    # 将请求头中满足foo=bar的请求转发到grpc-service-2服务中
    nginx.ingress.kubernetes.io/service-match: 'grpc-service-2: header("foo", "bar")'
spec:
  tls:
    - hosts:
      # 证书域名
      - grpc.example.com
      secretName: grpc-secret
  rules:
  # gRPC服务域名
  - host: grpc.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: grpc-service
          servicePort: 50051
      - path: /
        backend:
          serviceName: grpc-service-2
          servicePort: 50051
[root@k8s-master ~]# kubectl apply -f grpc-ingress-2.yml
ingress "grpc-ingress" configured
```

###### 3、测试gRPC服务访问：

```
 ## 不带foo=bar的请求头访问
$ grpcurl -insecure -d '{"name": "gRPC"}' grpc.example.com:443 helloworld.Greeter.SayHello
{
  "message": "Hello gRPC"
}
  ## 带foo=bar的请求头访问
$ grpcurl -insecure -rpc-header 'foo: bar' -d '{"name": "gRPC"}' grpc.example.com:443 helloworld.Greeter.SayHello
{
  "message": "Hello2 gRPC"
}
```



# Harbor部署

[官方文档](https://github.com/goharbor/harbor/blob/master/docs/installation_guide.md)

Harbor有两种安装的方式：

- 在线安装：直接从Docker Hub下载Harbor的镜像，并启动。
- 离线安装：在官网上下载离线安装包其地址为：https://github.com/goharbor/harbor/releases

1、环境需求

目标主机需要部署Docker和Docker-compose，以下为官方的软硬件要求：

**硬件需求**

| 资源   | 容量    | 推荐配置 |
| ------ | ------- | -------- |
| CPU    | >= 2C   | >= 4C    |
| Memory | >= 4GB  | >= 8GB   |
| Disk   | >= 40GB | >= 160GB |

**软件需求**

| 软件           | 版本          |
| -------------- | ------------- |
| Docker Engine  | >= 19.03.8-ce |
| Docker Compose | >= 1.18.0     |
| Openssl        | 最新版本      |

2、安装步骤

安装步骤归结为以下内容

- （1）下载安装程序，并安装docker-compose;
- （2）配置**harbor.yml** ;
- （3）运行**install.sh**安装并启动Harbor;



# 安装 cert-manager

参考官方文档: https://docs.cert-manager.io/en/latest/getting-started/install/kubernetes.html

介绍几种安装方式，不管是用哪种我们都先规划一下使用哪个命名空间，推荐使用 `cert-manger` 命名空间，如果使用其它的命名空间需要做些更改，会稍微有点麻烦，先创建好命名空间:

```bash
kubectl create namespace cert-manager
```

## 使用原生 yaml 资源安装

直接执行 `kubectl apply` 来安装:

```bash
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.13.1/cert-manager.yaml
```

> 使用 `kubectl v1.15.4` 及其以下的版本需要加上 `--validate=false`，否则会报错。

## 校验是否安装成功

检查 cert-manager 相关的 pod 是否启动成功:

```bash
$ kubectl get pods --namespace cert-manager

NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5c6866597-zw7kh               1/1     Running   0          2m
cert-manager-cainjector-577f6d9fd7-tr77l   1/1     Running   0          2m
cert-manager-webhook-787858fcdb-nlzsq      1/1     Running   0          2m
```



# nginx-ingress做tcp\udp 4层网络转发

> 检查nginx-ingress是否开启tcp\udp转发

官方配置文件[mandatory.yaml](https://github.com/NexClipper/kubernetes-yaml/blob/master/ingress/ingressnodeport/mandatory.yaml)默认是开启的,最好检查一下。

![截屏2020-12-14 下午4.16.12](/Users/jinhuaiwang/Desktop/截屏2020-12-14 下午4.16.12.png)



> 示例 kuard-demo.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kuard
spec:
  selector:
    matchLabels:
      app: kuard
  replicas: 1
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
      - image: paulcapestany/kuard-amd64:1
        imagePullPolicy: Always
        name: kuard
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: kuard
spec:
  ports:
  - port: 9527
    targetPort: 8080
    protocol: TCP
  selector:
    app: kuard
```

> 更新configmaps

```shell
$kubectl get cm -n ingress-nginx 
NAME                              DATA   AGE
ingress-controller-leader-nginx   0      10m
nginx-configuration               0      10m
tcp-services                      2      10m
udp-services                      0      10m
```



​	由于，入口不支持TCP或UDP服务。因此，此Ingress控制器使用这些标志 `--tcp-services-configmap`和 `--udp-services-configmap` 指向现有的配置映射，其中键是要使用的外部端口，并且该值使用以下格式指示要公开的服务： `<namespace/service name>:<service port>:[PROXY]:[PROXY]`

也可以使用端口号或名称。最后两个字段是可选的。添加 `PROXY` 最后两个字段中的一个或两个，我们可以在TCP服务[https://www.nginx.com/resources/admin-guide/proxy-protocol中](https://www.nginx.com/resources/admin-guide/proxy-protocol)使用代理协议解码（侦听）和/或编码（proxy_pass）

以下示例：使用9527端口转发到default/kuard:9527服务端口上

```yaml
# cat tcp-services.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
data:
  9527: "default/kuard:9527" #将9527端口转发到default/kuard:9527服务端口上）
```

> 进入nginx-ingress容器查看TCP services处会出现对应的负载配置

```shell
cat nginx.conf

# TCP services

server {
        preread_by_lua_block {
                ngx.var.proxy_upstream_name="tcp-default-kuard-9527";
        }

        listen                  9527;

        proxy_timeout           600s;
        proxy_pass              upstream_balancer;

}

# UDP services
```

最后即可通过边缘节点 ip:9527 访问。当pod节点扩容后红线标记的hostname也会随刷新变化。
![img](https://img2020.cnblogs.com/blog/355798/202007/355798-20200723175616196-1786809886.png)

[参考文档](https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/)





### 问题

在Kuberetes应用中，一般都是通过Ingress来暴露HTTP/HTTPS的服务。但是在实际应用中，还是有不少应用是TCP长连接的，这个是否也是可以通过Ingress来暴露呢？大家知道Kubernetes社区默认带了一个Nginx的Ingress的，而它本身又是支持TCP做反向代理的。所以也就能支持TCP方式的Ingress的。

具体可以参考：
https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/

### 原理

Ingress Controller在启动时会去watch两个configmap(一个tcp，一个 udp)，里面记录了后面需要反向代理的TCP的服务以及暴露的端口。如果里面的key-value发生变换，Ingress controller会去更改Nginx的配置，增加对应的TCP的listen的server以及对应的后端的upstream。

### 实践

#### 配置Nginx的Ingress的deployment/daemonSet

- 增加Ingress controller需要watch的configmap
  [![Snip20180621_41.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/c7c4c069117a7acd1d2cb2904f808a03.png)](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/c7c4c069117a7acd1d2cb2904f808a03.png)
- 创建对应的configmap，暂时不需要配置服务。（注意，阿里云的Kubernetes集群，已经默认创建好了对应的configmap: tcp-services, udp-services，无需再创建）

TCP configmap

```undefined
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: kube-system
```

UDP configmap

```undefined

apiVersion: v1
kind: ConfigMap
metadata:
  name: udp-services
  namespace: kube-system
```



#### 创建一个TCP服务

为了简单，可以直接用一个Nginx来模拟就可。如果创建这个deployment, service这个就不啰嗦了。

- 给TCP的服务配置"Ingress" TCP服务不像HTTP，不能使用Kubernetes的Ingress对象来配置，而是使用修改对应的Configmap来增加一个"Ingress"

kubectl edit configmap/tcp-services -n kube-system

- 然后增加data部分:

格式为：

<Nginx port>: <namespace/service name>:<service port>:[PROXY]:[PROXY]

[![Snip20180621_42.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/b87adfa0ede4f695f0338180c0244003.png)](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/b87adfa0ede4f695f0338180c0244003.png)

例子中将default/nginx这个服务暴露到5555端口

#### 修改Ingress Controller的Service

因为TCP部分是需要通过端口来区分服务的，所以每个服务都需要增加一个独立端口，所以需要给Ingress Controller增加新的端口来映射后端的TCP服务

[![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/5776e07f67a442b5eacf0766bebc88d8.png)](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/5776e07f67a442b5eacf0766bebc88d8.png)

注意：在阿里云的Kubernetes服务创建的集群，可以不用指定nodeport，对应的cloud provider会自动到对应的SLB上创建端口映射如：





# Tip:

```
弃用Ingress类注释 
IngressClass在Kubernetes 1.18中添加资源之前，通常会kubernetes.io/ingress.class在Ingress上使用注解指定类似的Ingress类概念。尽管从未正式定义此注释，但Ingress控制器已广泛支持此注释，现在应将其视为正式弃用。

设置默认的IngressClass
可以IngressClass在集群中将特定标记为默认值。ingressclass.kubernetes.io/is-default-class在IngressClass资源上将注释设置为true将确保未ingressClassName指定新的Ingress分配此默认值IngressClass。
```



# 外部访问Kubernetes中的Pod

前面几节讲到如何访问kubneretes集群，本文主要讲解访问kubenretes中的Pod和Serivce的几种方式，包括如下几种：

- hostNetwork
- hostPort
- NodePort
- LoadBalancer
- Ingress



## hostNetwork

这是一种直接定义Pod网络的方式。

如果在Pod中使用`hostNotwork:true`配置的话，在这种pod中运行的应用程序可以直接看到pod启动的主机的网络接口。在主机的所有网络接口上都可以访问到该应用程序。以下是使用主机网络的pod的示例定义：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: influxdb
spec:
  hostNetwork: true
  containers:
    - name: influxdb
      image: influxdb
```

部署该Pod：

```bash
$ kubectl create -f influxdb-hostnetwork.yml
```

访问该pod所在主机的8086端口：

```bash
curl -v http://$POD_IP:8086/ping
```

将看到204 No Content的204返回码，说明可以正常访问。

注意每次启动这个Pod的时候都可能被调度到不同的节点上，所有外部访问Pod的IP也是变化的，而且调度Pod的时候还需要考虑是否与宿主机上的端口冲突，因此一般情况下除非您知道需要某个特定应用占用特定宿主机上的特定端口时才使用`hostNetwork: true`的方式。

这种Pod的网络模式有一个用处就是可以将网络插件包装在Pod中然后部署在每个宿主机上，这样该Pod就可以控制该宿主机上的所有网络。



## hostPort

这是一种直接定义Pod网络的方式。

`hostPort`是直接将容器的端口与所调度的节点上的端口路由，这样用户就可以通过宿主机的IP:Port来访问Pod了，如:。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: influxdb
spec:
  containers:
    - name: influxdb
      image: influxdb
      ports:
        - containerPort: 8086
          hostPort: 8086
```

这样做有个缺点，因为Pod重新调度的时候该Pod被调度到的宿主机可能会变动，这样就变化了，用户必须自己维护一个Pod与所在宿主机的对应关系。

这种网络方式可以用来做 nginx ingress controller。外部流量都需要通过kubenretes node节点的80和443端口。



#  k8s生产问题

## Kubernetes 外部服务映射



集群内的应用有时候需要调用外部的服务，我们知道集群内部服务调用都是通过 Service 互相访问，那么针对外部的服务是否也可以保持统一使用 Service 呢？答案是肯定的，通过 Service 访问外部服务，除了方式统一以外，还能带来其他好处。如配置统一，不同环境（空间）相同应用访问外部不同环境的数据库，可以通过 Service 映射保持两边配置统一，达到不同空间应用通过相同 Service Name 访问不同的外部数据库。如图 `test-1` 和 `test-2` 两个空间为两个不同的业务环境，通过服务映射，不同空间相同的 Service 访问到对应外部不同环境的数据库：

![external-service.png](https://opskumu.oss-cn-beijing.aliyuncs.com/images/external-service.png)

另外，还可以保证最小化变更，如果外部数据库 IP 之类的变动，只需要修改 Service 对应映射即可，服务本身配置无需变动。具体场景如下：

## 1 外部域名映射到内部 Service -> `ExternalName`

从Pod中访问外部服务的最简单方法是创建`ExternalName` service, 创建 ExternalName 类型的服务yaml 如下：

```
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  externalName: mysql.example.com
  type: ExternalName
```

创建之后，同一空间 Pod 就可以通过 `mysql:3306` 访问外部的 MySQL 服务。

当查找主机 `mysql.default.svc.cluster.local`时，集群DNS服务返回CNAME记录，其值为`mysql.example.com`。 访问`mysql`的方式与其他服务的方式相同，但主要区别在于重定向发生在 DNS 级别，而不是通过代理或转发。

coredns增加如下配置:

```
data:
  Corefile: |   
   ......
    #只要mysql.exmaple.com域名,转发到192.168.10.142去解析
    mysql.example.com:53 {
        errors
        cache 30
        forward . 192.168.10.142 
    } 
```



需要注意的是，虽然 externalName 也支持填写 IP，但是并不会被 Ingress 和 CoreDNS 解析（KubeDNS 支持）。如果有 IP 相关的需求，则可以使用 Headless Service -> [Type ExternalName](https://kubernetes.io/docs/concepts/services-networking/service/#externalname) 。另外一个需要注意的是，因为 CNAME 的缘故，如果外部的服务又经过一层代理转发，如 Nginx，除非配置对应的 `server_name` ，否则映射无效。

## 2 外部 IP 映射到内部 Service

### 2.1 IP 映射

前文提过，虽然 externalName 字段可以配置为 IP 地址，但是 Ingress 和 CoreDNS 并不会解析，如果外部服务为 IP 提供，那么可以使用 Headless Service 实现映射。

```
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  type: ClusterIP
---
apiVersion: v1
kind: Endpoints
metadata:
 name: mysql
subsets:
 - addresses:
     - ip: 192.168.1.10
```

service 不指定 selector，手动维护创建 endpoint，创建之后就可以通过 `mysql:3306` 访问 `192.168.1.10:3306` 服务的目的。Headless Service 不能修改端口相关，如果要修改访问端口，则需要进一步操作。

### 2.2 IP + 端口映射

如果外部的端口不是标准的端口，想通过 Service 访问时候使用标准端口，如外部 MySQL 提供端口为 3307，内部想通过 Service 3306 访问，这个时候则可以通过如下方式实现：

```
apiVersion: v1
kind: Service
metadata:
 name: mysql
spec:
 type: ClusterIP
 ports:
 - port: 3306
   targetPort: 3307
---
apiVersion: v1
kind: Endpoints
metadata:
 name: mysql
subsets:
 - addresses:
     - ip: 192.168.1.10
   ports:
     - port: 3307
```

service 不指定 selector，手动维护创建 endpoint，创建之后就可以通过 `mysql:3306` 达到访问外部 `192.168.1.10:3307` 服务的目的。

## 3 小结

我们可以看出以上外部服务映射，externalName 和 Headless Service 方式映射外部服务是没有经过中间层代理的，都是通过 DNS 劫持实现。而有端口变更需求的时候，则要经过内部 kube-proxy 层转发。正常情况下，能尽可能少的引入中间层就少引用，特别是数据库类的应用，因为引入中间层虽然带来了便利，但也意味着可能会带来性能损耗，特别是那些对延迟比较敏感的服务。





另外，externalName既然是Service，它就可以被Ingress作为backend。这样做的话，就等于直接在我们的ingress上反向代理了exernalName Service上的服务。哦耶，我可以用k8s的语法来配置反向代理，哈哈。不过看着也就是个不疼不痒的功能。除了这个还有别的应用么？

一般我们为了方便都直接把一个解析域名解析到集群，比如我`*.renwei.net`解析到我的ingress集群，`blog.renwei.net`是一个集群内的服务，`http://wiki.renwei.net`是另一个集群的服务。等某一天，我想把`blog.renwei.net]`迁到另外一个地方，可能是另外的物理机或者另外的集群。很简单在新址搭建好后，新建一条`blog.renwei.net`的单独DNS解析到新址就可以了。这一条或几条DNS记录还好说，如果用`renwei.net`搭建了好多二级域名服务呢？都要迁到另一个集群上，就这么一条一条改配DNS记录么（配DNS当然也很简单）？有externalName Service就可以不必配这么多DNS记录了，我在新集群上给服务配上两个域名（一个原域名，一个临时的新域名，用这个新域名是因为externalName只支持域名），在原集群里，把老服务的service修改成externalName类型，值就是临时的新域名。等我一个一个把服务都这么迁过去后，我就可以`*.renwei.net`直接改解析到新集群即可。



## ExternalName 扩展

**Note**: NGINX Plus Support for Type ExternalName Services

​		当访问Ingress的时候Ingress指向到一个service，然后service指向到一个外部app（这个过程中nginx解析这个外部app的dns）

For example, the following ConfigMap configures one resolver:

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-config
  namespace: nginx-ingress
data:
  resolver-addresses: "10.0.0.10"
```

In the following yaml file we define an ExternalName service with the name my-service:

```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  type: ExternalName
  externalName: my.service.example.com
```

In the following Ingress resource we use my-service:

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: my-service
          servicePort: 80
```



# k8s中部署 Tomcat+MySQL服务

​		实现运行在Tomcat里的 Web app，ISP页面通过 JDBC 直接访问 MySQL数据库并展示数据。 

​		需求：Web App 容器 MySQL容器，web--->mysql 需要把MySQL容器的IP地址通过环境变量的方式注入 Web App容器里，同时，需要将Web App容器的 8080端口映射宿主机的 8080端口，以便在外部访问。 



**MySQL服务创建 deploy 文件**

```yaml
cat << eof > mysql-deploy.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "123456"
eof
```

创建和查看创建的pod

```
# kubectl create -f mysql-rc.yaml 
replicationcontroller/mysql created
# kubectl get pods | grep mysql
mysql-76999dd7c8-cgcwk              1/1     Running   0          40m
```

MySQL服务创建svc文件

```yaml
cat << eof > mysql-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
eof
```

创建和查看mysql svc状态

```
# kubectl create -f mysql-svc.yaml 
service/mysql created
# kubectl get svc | grep mysql
mysql           ClusterIP   10.0.0.127   <none>        3306/TCP         42m
```

MySQL服务分配了 10.0.0.127 的Cluster IP地址，这是一个虚拟IP地址，随后，Kubernetes集群中的其他新创建的 Pod就可以通过 Service 的 Cluster IP+端口号 3306来连接和访问它。



根据 Service 的唯一名字，容器可以从环境变量中获取到 Service对应的 Cluster IP地址和端口，从而发起 TCP/IP 连接请求。

 

Tomcat 服务创建deployment 文件

```yaml
cat << eof > myweb-deploy.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: myweb
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myweb
  template:
    metadata:
      labels:
        app: myweb
    spec:
      containers:
      - name: myweb
        image: kubeguide/tomcat-app:v1
        ports:
        - containerPort: 8080
        env:
        - name: MYSQL_SERVICE_HOST
          value: 'mysql'     #mysql service name
        - name: MYSQL_SERVICE_PORT
          value: '3306'
eof
```

创建和查看tomcat pod状态

```
# kubectl create -f myweb-rc.yaml 
replicationcontroller/myweb created
# kubectl get pods | grep myweb
myweb-77b4646d58-tcmds              1/1     Running   0          19m
myweb-77b4646d58-wkzx6              1/1     Running   0          19m
```

Tomcat 服务创建 svc 文件

```yaml
cat << eof > myweb-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: myweb
spec:
  type: NodePort
  ports:
  - port: 8080
    nodePort: 30001
  selector:
    app: myweb
eof
```

创建和查看tomcat svc状态

```
# kubectl create -f myweb-svc.yaml 
service/myweb created
# kubectl get svc | grep myweb
myweb           NodePort    10.0.0.27    <none>        8080:30001/TCP   44m
```

 

进行验证

```
# kubectl get pods -o wide | grep myweb
NAME           READY STATUS    RESTARTS   AGE   IP      NODE        NOMINATED NODE  READINESS GATES
myweb-77b4646d58-tcmds  1/1  Running   0      22m   172.17.30.6  192.168.10.148   <none>      <none>
myweb-77b4646d58-wkzx6  1/1  Running   0      22m   172.17.39.6  192.168.10.149   <none>      <none>
```



浏览器输入http://192.168.10.148:30001/demo

<img src="/Users/jinhuaiwang/Library/Application Support/typora-user-images/截屏2021-03-15 上午11.03.42.png" alt="截屏2021-03-15 上午11.03.42" style="zoom:50%;" />



添加内容并提交

![截屏2021-03-15 上午11.04.44](/Users/jinhuaiwang/Library/Application Support/typora-user-images/截屏2021-03-15 上午11.04.44.png)



通过MySQL容器进行验证查询

```mysql
mysql> select * from T_USERS where USER_NAME='luce';
+----+-----------+-------+
| ID | USER_NAME | LEVEL |
+----+-----------+-------+
|  8 | luce      | 98    |
+----+-----------+-------+
1 row in set (0.00 sec)

mysql> 
```





# Pod 相关脚本

## 清理 Evicted 的 pod

```bash
kubectl get pod -o wide --all-namespaces | awk '{if($4=="Evicted"){cmd="kubectl -n "$1" delete pod "$2; system(cmd)}}'
```

## 清理非 Running 的 pod

```bash
kubectl get pod -o wide --all-namespaces | awk '{if($4!="Running"){cmd="kubectl -n "$1" delete pod "$2; system(cmd)}}'
```

## 升级镜像

```bash
NAMESPACE="kube-system"
WORKLOAD_TYPE="daemonset"
WORKLOAD_NAME="ip-masq-agent"
CONTAINER_NAME="ip-masq-agent"
IMAGE="ccr.ccs.tencentyun.com/library/ip-masq-agent:v2.5.0"
kubectl -n $NAMESPACE patch $WORKLOAD_TYPE $WORKLOAD_NAME --patch '{"spec": {"template": {"spec": {"containers": [{"name": "$CONTAINER_NAME","image": "$IMAGE" }]}}}}'
```

# 节点资源查看

## 表格输出各节点占用的 podCIDR

```bash
kubectl get no -o=custom-columns=INTERNAL-IP:.metadata.name,EXTERNAL-IP:.status.addresses[1].address,CIDR:.spec.podCIDR
```

示例输出:

```
INTERNAL-IP     EXTERNAL-IP       CIDR
10.100.12.194   152.136.146.157   10.101.64.64/27
10.100.16.11    10.100.16.11      10.101.66.224/27
10.100.16.24    10.100.16.24      10.101.64.32/27
10.100.16.26    10.100.16.26      10.101.65.0/27
10.100.16.37    10.100.16.37      10.101.64.0/27
```

## 表格输出各节点总可用资源 (Allocatable)

```bash
kubectl get no -o=custom-columns="NODE:.metadata.name,ALLOCATABLE CPU:.status.allocatable.cpu,ALLOCATABLE MEMORY:.status.allocatable.memory"
```

示例输出：

```
NODE       ALLOCATABLE CPU   ALLOCATABLE MEMORY
10.0.0.2   3920m             7051692Ki
10.0.0.3   3920m             7051816Ki
```

## 输出各节点已分配资源的情况

所有种类的资源已分配情况概览：

```bash
kubectl get nodes --no-headers | awk '{print $1}' | xargs -I {} sh -c "echo {} ; kubectl describe node {} | grep Allocated -A 5 | grep -ve Event -ve Allocated -ve percent -ve --;"
```

示例输出:

```
10.0.0.2
  Resource           Requests          Limits
  cpu                3040m (77%)       19800m (505%)
  memory             4843402752 (67%)  15054901888 (208%)
10.0.0.3
  Resource           Requests   Limits
  cpu                300m (7%)  1 (25%)
  memory             250M (3%)  2G (27%)
```

表格输出 cpu 已分配情况:

```bash
kubectl get nodes --no-headers | awk '{print $1}' | xargs -I {} sh -c 'echo -n "{}\t" ; kubectl describe node {} | grep Allocated -A 5 | grep -ve Event -ve Allocated -ve percent -ve -- | grep cpu | awk '\''{print $2$3}'\'';'
```

示例输出：

```bash
10.0.0.2	3040m(77%)
10.0.0.3	300m(7%)
```

表格输出 memory 已分配情况:

```bash
kubectl get nodes --no-headers | awk '{print $1}' | xargs -I {} sh -c 'echo -n "{}\t" ; kubectl describe node {} | grep Allocated -A 5 | grep -ve Event -ve Allocated -ve percent -ve -- | grep memory | awk '\''{print $2$3}'\'';'
```

示例输出：

```bash
10.0.0.2	4843402752(67%)
10.0.0.3	250M(3%)
```



## 线程数排名统计

```bash
printf "    NUM  PID\t\tCOMMAND\n" && ps -eLf | awk '{$1=null;$3=null;$4=null;$5=null;$6=null;$7=null;$8=null;$9=null;print}' | sort |uniq -c |sort -rn | head -10
```

- 第一列表示线程数
- 第二列表示进程 PID
- 第三列是进程启动命令



k8s各个组件状态查看

```
systemctl is-active kube-apiserver.service kube-controller-manager.service kubelet.service  kube-proxy.service  kube-scheduler.service 
```





# Tools

## shell自动补全

kubectl包含了一个自动补全命令功能，可以大大提高工作效率。

CentOS可能需要先安装 `bash-completion` 软件包:

```
yum install bash-completion -y
```

Ubuntu则默认已经安装了 `bash-completion`

为了能够在当前shell中使用kubectl的自动补全功能，请执行 `source <(kubectl completion bash)`

也可以加入shell环境变量，这样登陆就可以使用:

```
echo "source <(kubectl completion bash)" >> ~/.bashrc
```



# Kubernetes排查



## crictl工具

`crictl` 是CRI兼容容器runtime的命令行接口。可以通过 `crictl` 在Kubernetes节点检查( `inspect`)和debug容器runtime和应用程序。

### 安装cri-tools

- 安装crictl:

  ```
  VERSION="v1.17.0"
  wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
  sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
  rm -f crictl-$VERSION-linux-amd64.tar.gz
  ```

- 安装critest:

  ```
  VERSION="v1.17.0"
  wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/critest-$VERSION-linux-amd64.tar.gz
  sudo tar zxvf critest-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
  rm -f critest-$VERSION-linux-amd64.tar.gz
  ```

### 使用crictl

`crictl` 命令默认连接到 `unix:///var/run/dockershim.sock` 如果要连接到其他runtimes，需要设置 endpoint:

- 可以通过命令行参数 `--runtime-endpoint` 和 `--image-endpoint`
- 可以通过设置环境变量 `CONTAINER_RUNTIME_ENDPOINT` 和 `CONTAINER_RUNTIME_ENDPOINT`
- 可以通过配置文件的endpoint设置 `--config=/etc/crictl.yaml`

当前配置为 `/etc/crictl.yaml`

```
runtime-endpoint: unix:///var/run/dockershim.sock
image-endpoint: unix:///var/run/dockershim.sock
timeout: 10
debug: true
```

### crictl命令案例

- 列出pods:

  ```
  crictl pods
  ```



## 获取容器日志

在Pods中有多个容器，则需要获取容器日志，需要通过 `-c` 参数指定容器:

```
kubectl logs podname -c containername --previous
```

这里 `podname` 指定pod，`containername` 指定该pod中的容器， `--previous` 则输出最近一次容器启动后生成的日志。

对于pod，可以通过 `describe pod` 命令来获取pod的event记录，对于排查pod异常会很有帮助

```
kubectl describe pod  podname [-n my-namespace]
```

对于namespace，可以整体获得这个namespace中日志信息:

```
kubectl get event [-n my-namespace]
```



## Kubernetes pod CrashLoopBackOff错误排查

很多时候部署Kubernetes应用容器，经常会遇到pod进入 `CrashLoopBackOff` 状态，但是由于容器没有启动，很难排查问题原因。

### CrashLoopBackOff错误解析

`CrashloopBackOff` 表示pod经历了 `starting` , `crashing` 然后再次 `starting` 并再次 `crashing` 。

这个失败的容器会被kubelet不断重启，并且按照几何级数(exponentially)延迟(10s,20s,40s…)直到5分钟，最后一次是10分钟后重置。默认使用的是 [podRestartPolicy](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy)

```
PodSpec` 有一个 `restartPolicy` 字段，值可以是 `Always` , `OnFailure` 和 `Never` ，默认是 `Always
```

导致 `CrashLoopBackOff` 的原因通常有:

- 容器中应用程序持续crash
- pod/container的参数配置错误
- 当部署Kubernetes pod是出现错误

观察方法如下:

```
kubectl get pods -n <NameSpace>
```

### 排查方法

- 首先通过 `describe` pod获取信息:

  ```
  kubectl describe pod <podname>
  ```

如果要非常详细的信息，则可以加上一个 `-v=9` 参数

- 此外，可以通过 `get events` 来获得上述 `describe` 的事件部分信息:

  ```
  kubectl get event -n <NameSpace>
  ```

注意，Docker容器必须保持PID 1进程持续运行，否则容器就会退出(也就是主进程退出)。对于Docker而言PID 1退出就是容器停止，此时如果容器在Kubernetes中就会重启容器。

当容器重启后，Kubernetes就会申明这个Pod进入 `Back-off` 状态。此时通过 `kubectl get pods -n <NameSpace>` 就会看到容器重启计数增加:

```
$ kubectl get pods -n test-kube
NAME                         READY   STATUS             RESTARTS   AGE
challenge-7b97fd8b7f-cdvh4   0/1     CrashLoopBackOff   2          60s
```

- 接下来检查pod的日志:

  ```
  kubectl logs <podname> -n <NameSpace>
  ```

注意，容器运行需要有输出，通常是容器中运行程序的日志输出(容器通常就是运行一个应用)

- 最后查看 `Liveness/Readiness` probe:

  ```
  kubectl describe pod <podname> -n <NameSpace>
  ```

为了能够让容器不退出，你可以在运行命令中添加一段死循环:

```
while true; do sleep 20; done;
```

这样就可以保持容器持续运行，方便你查看日志

### 实践案例

我在制作 [Docker tini进程管理器](https://cloud-atlas.readthedocs.io/zh_CN/latest/docker/init/docker_tini.html#docker-tini) 镜像之后，已经能够在docker运行。但是我通过以下简单的 `deployments.yaml` 部署到Kubernetes集群时，发现pod不断重启:

```
NAME                           READY   STATUS             RESTARTS   AGE
onesre-core-5d4d4d8b5f-zn7lg   0/1     CrashLoopBackOff   6          9m1s
```

此时检查events显示:

```
kubectl --kubeconfig meta/admin.kubeconfig -n onesre describe pods onesre-core-5d4d4d8b5f-zn7lg
```

输出显示:

```
Warning  BackOff               9s (x4 over 24s)   kubelet            Back-off restarting failed containe
```

为何会判断容器失败呢？

参考 [配置存活、就绪和启动探测器](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) 添加侦测 `args` 部分:

```
spec:
  containers:
  - image: onesre:20210205-1
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 60
```

- 检查 `get pods`

  ```
  kubectl -n onesre get pods onesre-core-7697868c67-pmhng -o yaml
  ```

显示输出:

```
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2021-02-05T16:10:49Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2021-02-05T16:10:49Z"
    message: 'containers with unready status: [onesre]'
    reason: ContainersNotReady
    status: "False"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2021-02-05T16:10:49Z"
    message: 'containers with unready status: [onesre]'
    reason: ContainersNotReady
    status: "False"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2021-02-05T16:10:49Z"
    status: "True"
    type: PodScheduled
```

实际原因是Kubernetes启动pod一定要在container中运行一个程序，并根据运行程序返回来判断容器是否启动。最初我没有配置执行命令，考虑的是等容器启动之后，手工去部署。但是这不符合k8s定义。

所以添加以下内容:

```
command: [ "/bin/bash", "-ec"]
args: [ date; sleep 10; echo 'Hello from the Kubernetes cluster'; sleep 1; while true; do sleep 20; done;]
```

但是遇到问题，发现没有启动 tini

实际上我的案例是因为没有在Kubernetes启动pod时运行任何主程序，所以导致无法判断容器状态。





# iptables 与 ipvs对比



kube-proxy 是 Kubernetes 中的关键组件。他的角色就是在服务（ClusterIP 和 NodePort）和其后端 Pod 之间进行负载均衡。kube-proxy 有三种运行模式，每种都有不同的实现技术：**userspace**、**iptables** 或者 **IPVS**。

userspace 模式非常陈旧、缓慢，已经不推荐使用。但是 iptables 和 IPVS 该如何选择呢？本文中我们会对这两种模式进行比较，看看他们在真正的微服务上下文中的表现，并解释在特定情况下的选择方法。

首先我们说一下这两种模式的背景，然后开始测试并查看结果。

## iptables 模式

iptables 是一个 Linux 内核功能，是一个高效的防火墙，并提供了大量的数据包处理和过滤方面的能力。它可以在核心数据包处理管线上用 Hook 挂接一系列的规则。iptables 模式中 kube-proxy 在 `NAT pre-routing` Hook 中实现它的 NAT 和负载均衡功能。这种方法简单有效，依赖于成熟的内核功能，并且能够和其它跟 iptables 协作的应用（例如 Calico）融洽相处。

然而 kube-proxy 的用法是一种 O(n) 算法，其中的 n 随集群规模同步增长，这里的集群规模，更明确的说就是服务和后端 Pod 的数量。

## IPVS 模式

IPVS 是一个用于负载均衡的 Linux 内核功能。IPVS 模式下，kube-proxy 使用 IPVS 负载均衡代替了 iptable。这种模式同样有效，IPVS 的设计就是用来为大量服务进行负载均衡的，它有一套优化过的 API，使用优化的查找算法，而不是简单的从列表中查找规则。

这样一来，kube-proxy 在 IPVS 模式下，其连接过程的复杂度为 O(1)。换句话说，多数情况下，他的连接处理效率是和集群规模无关的。

另外作为一个独立的负载均衡器，IPVS 包含了多种不同的负载均衡算法，例如轮询、最短期望延迟、最少连接以及各种哈希方法等。而 iptables 就只有一种随机平等的选择算法。

IPVS 的一个潜在缺点就是，IPVS 处理数据包的路径和通常情况下 iptables 过滤器的路径是不同的。如果计划在有其他程序使用 iptables 的环境中使用 IPVS，需要进行一些研究，看看他们是否能够协调工作。（Calico 已经和 IPVS kube-proxy 兼容）

## 性能对比

iptables 的连接处理算法复杂度是 O(n)，而 IPVS 模式是 O(1)，但是在微服务环境中，其具体表现如何呢？

在多数场景中，有两个关键属性需要关注：

- 响应时间：一个微服务向另一个微服务发起调用时，第一个微服务发送请求，并从第二个微服务中得到响应，中间消耗了多少时间？
- CPU 消耗：运行微服务的过程中，总体 CPU 使用情况如何？包括用户和核心空间的 CPU 使用，包含所有用于支持微服务的进程（也包括 kube-proxy）。

为了说明问题，我们运行一个微服务作为客户端，这个微服务以 Pod 的形式运行在一个独立的节点上，每秒钟发出 1000 个请求，请求的目标是一个 Kubernetes 服务，这个服务由 10 个 Pod 作为后端，运行在其它的节点上。接下来我们在客户端节点上进行了测量，包括 iptables 以及 IPVS 模式，运行了数量不等的 Kubernetes 服务，每个服务都有 10 个 Pod，最大有 10,000 个服务（也就是 100,000 个 Pod）。我们用 golang 编写了一个简单的测试工具作为客户端，用标准的 NGINX 作为后端服务。

## 响应时间

响应时间很重要，有助于我们理解连接和请求的差异。典型情况下，多数微服务都会使用持久或者 `keepalive` 连接，这意味着每个连接都会被多个请求复用，而不是每个请求一次连接。这很重要，因为多数连接的新建过程都需要完成三次 TCP 握手的过程，这需要消耗时间，也需要在 Linux 网络栈中进行更多操作，也就会消耗更多 CPU 和时间。

![Round-Trip Response TIme vs Number of Services](https://blog.fleeto.us/post/iptables-or-ipvs/images/Picture1.png)

这张图展示了两个关键点：

- iptables 和 IPVS 的平均响应时间在 1000 个服务（10000 个 Pod）以上时，会开始观察到差异。
- 只有在每次请求都发起新连接的情况下，两种模式的差异才比较明显。

不管是 iptables 还是 IPVS，kube-proxy 的响应时间开销都是和建立连接的数量相关的，而不是数据包或者请求数量，这是因为 Linux 使用了 Conntrack，能够高效地将数据包和现存连接关联起来。如果数据包能够被 Conntrack 成功匹配，那就不需要通过 kube-proxy 的 iptables 或 IPVS 规则来推算去向。Linux conntrack 非常棒！（绝大多数时候）

值得注意的是，例子中的服务端微服务使用 NGINX 提供一个静态小页面。多数微服务要做更多操作，因此会产生更高的响应时间，也就是 kube-proxy 处理过程在总体时间中的占比会减少。

还有个需要解释的古怪问题：既然 IPVS 的连接过程复杂度是 O(1)，为什么在 10,000 服务的情况下，非 Keepalive 的响应时间还是提高了？我们需要深入挖掘更多内容才能解释这一问题，但是其中一个因素就是因为上升的 CPU 用量拖慢了整个系统。这就是下一个主题需要探究的内容。

## CPU 用量

为了描述 CPU 用量，下图关注的是最差情况：不使用持久/keepalive 连接的情况下，kube-proxy 会有最大的处理开销。

![Total CPU](https://blog.fleeto.us/post/iptables-or-ipvs/images/Picture2.png)

上图说明了两件事：

- 在超过 1000 个服务（也就是 10,000 个 Pod）的情况下，CPU 用量差异才开始明显。
- 在一万个服务的情况下（十万个后端 Pod），iptables 模式增长了 0.35 个核心的占用，而 IPVS 模式仅增长了 8%。

有两个主要因素造成 CPU 用量增长：

第一个因素是，缺省情况下 kube-proxy 每 30 秒会用所有服务对内核重新编程。这也解释了为什么 IPVS 模式下，新建连接的 O(1) 复杂度也仍然会产生更多的 CPU 占用。另外，如果是旧版本内核，重新编程 iptables 的 API 会更慢。所以如果你用的内核较旧，iptables 模式可能会占用更多的 CPU。

另一个因素是，kube-proxy 使用 IPVS 或者 iptables 处理新连接的消耗。对 iptables 来说，通常是 O(n) 的复杂度。在存在大量服务的情况下，会出现显著的 CPU 占用升高。例如在 10,000 服务（100,000 个后端 Pod）的情况下，iptables 会为每个请求的每个连接处理大约 20000 条规则。如果使用 NINGX 缺省每连接 100 请求的 keepalive 设置，kube-proxy 的 iptables 规则执行次数会减少为 1%，会把 iptables 的 CPU 消耗降低到和 IPVS 类似的水平。

客户端微服务会简单的丢弃响应内容。真实世界中自然会进行更多处理，也会造成更多的 CPU 消耗，但是不会影响 CPU 消耗随服务数量增长的事实。

## 结论

在超过 1000 服务的规模下，kube-proxy 的 IPVS 模式会有更好的性能表现。虽然可能有多种不同情况，但是通常来说，让微服务使用持久连接、运行现代内核，也能取得较好的效果。如果运行的内核较旧，或者无法使用持久连接，那么 IPVS 模式可能是个更好的选择。

抛开性能问题不谈，IPVS 模式还有个好处就是具有更多的负载均衡算法可供选择。

如果你还不确定 IPVS 是否合适，那就继续使用 iptables 模式好了。这种传统模式有大量的生产案例支撑，他是一个不完美的缺省选项。

## 补充：Calico 和 kube-proxy 的 iptables 比较

本文中我们看到，kube-proxy 中的 iptables 用法在大规模集群中可能会产生性能问题。有人问我 Calico 为什么没有类似的问题。答案是 Calico 中 kube-proxy 的用法是不同的。kube-proxy 使用了一个很长的规则链条，链条长度会随着集群规模而增长，Calico 使用的是一个很短的优化过的规则链，经由 ipsets 的加持，也具备了 O(1) 复杂度的查询能力。

下图证明了这一观点，其中展示了每次连接过程中，kube-proxy 和 Calico 中 iptables 规则数量的平均值。这里假设集群中的节点平均有 30 个 Pod，每个 Pod 具有 3 个网络规则。

![calico vs kube-proxy](https://blog.fleeto.us/post/iptables-or-ipvs/images/Picture3.png)

即使是使用 10,000 个服务和 100,000 个 Pod 的情况下，Calico 每连接执行的 iptables 规则也只是和 kube-proxy 在 20 服务 200 个 Pod 的情况基本一致。





# kube-proxy配置 ipvs模式

> 在k8s中，提供相同服务的一组pod可以抽象成一个service，通过service提供的统一入口对外提供服务，每个service都有一个虚拟IP地址（clusterip）和端口号供客户端访问。
>
> Kube-proxy存在于各个node节点上，主要用于Service功能的实现，具体来说，就是实现集群内的客户端pod访问service，或者是集群外的主机通过NodePort等方式访问service。
>
> kube-proxy默认使用的是iptables模式，通过各个node节点上的iptables规则来实现service的负载均衡，但是随着service数量的增大，iptables模式由于线性查找匹配、全量更新等特点，其性能会显著下降。
>
> 从k8s的1.8版本开始，kube-proxy引入了IPVS模式，IPVS模式与iptables同样基于Netfilter，但是采用的hash表，因此当service数量达到一定规模时，hash查表的速度优势就会显现出来，从而提高service的服务性能。



下面我们先来学习一下IPVS的相关知识

> IPVS是LVS的核心组件，是一种四层负载均衡器。IPVS具有以下特点：
> 与Iptables同样基于Netfilter，但使用的是hash表；
> 支持TCP, UDP，SCTP协议，支持IPV4，IPV6；
> 支持多种负载均衡策略：rr, wrr, lc, wlc, sh, dh, lblc…
> 支持会话保持；
>
> LVS主要由两部分组成：
> ipvs(ip virtual server)：即ip虚拟服务，是工作在内核空间上的一段代码，主要是实现调度的代码，它是实现负载均衡的核心。
> ipvsadm： 工作在用户空间，负责为ipvs内核框架编写规则，用于定义谁是集群服务，谁是后端真实服务器。我们可以通过ipvsadm指令创建集群服务

```
# ipvsadm -A -t 192.168.2.xx:80 -s rr //创建一个DR，并指定调度算法采用rr。
# ipvsadm -a -t 192.168.2.xx:80 -r 192.168.10.xx 
# ipvsadm -a -t 192.168.2.xx:80 -r 192.168.11.xx //添加两个RS
```

> IPVS基于Netfilter， Netfilter 中制定了数据包的五个挂载点（Hook Point），这5个挂载点分别是PRE_ROUTING、INPUT、OUTPUT、FORWARD、POST_ROUTING，如下图所示，IPVS工作在其中的INPUT链上。

![kube-proxy配置 ipvs模式](https://s4.51cto.com/images/blog/201811/14/de4480c3071cd7fba17a75449c68ddf2.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

> 当客户端的请求到达负载均衡器的内核空间时，首先会到达PREROUTING链。
> 当内核发现请求数据包的目的地址是本机时，将数据包送往INPUT链。
> 当数据包到达INPUT链时，首先会被IPVS检查，如果数据包里面的目的地址及端口没有在IPVS规则里面，那么这条数据包将被放行至用户空间。
> 如果数据包里面的目的地址及端口在IPVS规则里面，那么这条数据报文的目的地址被修改为通过负载均衡调度算法选好的后端服务器（DNAT），并送往POSTROUTING链。
> 最后经由POSTROUTING链发往后端服务器。

下面我们来配置kube-proxy模式为IPVS

## 开启node节点的内核参数

```
# grep -v '^#' /etc/sysctl.conf    
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
# sysctl -p
```

## 安装ipvs相关软件包

```
# yum -y install ipvsadm ipset
```

## 修改kube-proxy启动脚本

```shell
# cat /usr/lib/systemd/system/kube-proxy.service   
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/sbin/kube-proxy \
  --bind-address=192.168.4.23 \
  --hostname-override=server23 \
  --cluster-cidr=172.35.0.0/16 \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
  --masquerade-all \
  --feature-gates=SupportIPVSProxyMode=true \
  --proxy-mode=ipvs \
  --ipvs-min-sync-period=5s \
  --ipvs-sync-period=5s \
  --ipvs-scheduler=rr \
  --logtostderr=true \
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

## 重启服务与验证

```shell
# systemctl daemon-reload
# systemctl restart kube-proxy

# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  127.0.0.1:30080 rr
  -> 172.17.55.13:9090            Masq    1      0          0         
TCP  127.0.0.1:30180 rr
  -> 172.17.74.12:80              Masq    1      0          0         
TCP  127.0.0.1:42345 rr
  -> 172.17.86.16:9090            Masq    1      0          0         
TCP  172.17.86.0:30080 rr
  -> 172.17.55.13:9090            Masq    1      0          0         
TCP  172.17.86.0:30180 rr
  -> 172.17.74.12:80              Masq    1      0          0         
TCP  172.17.86.0:30443 rr
  -> 172.17.74.12:443             Masq    1      0          0         
TCP  172.17.86.0:42345 rr
  -> 172.17.86.16:9090            Masq    1      0          0         
TCP  172.17.86.1:30080 rr
  -> 172.17.55.13:9090            Masq    1      0          0         
TCP  172.17.86.1:30180 rr
  -> 172.17.74.12:80              Masq    1      0          0         
......     
```





# References

* [kubernetes源码讲解](https://draveness.me/kubernetes-service/)
* [Debugging Kubernetes nodes with crictl](https://kubernetes.io/docs/tasks/debug-application-cluster/crictl/)

- [Troubleshoot pod CrashLoopBackOff error:: Kubernetes](https://beanexpert.co.in/troubleshoot-pod-crashloopbackoff-error-kubernetes/)
- [Helm 3入门到实战](https://cloudnative.run/post/1616.html)
- [iptable与ipvs对比](https://blog.fleeto.us/post/iptables-or-ipvs/)

