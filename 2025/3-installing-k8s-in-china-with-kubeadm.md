# 国内安装k8s——kubeadm方式

> 本文的每一节都是有顺序的操作步骤，并穿插解释和可能出现的问题的解决方案。

TODO：需要补充更多的troubleshoot

---
## 关闭swap
swap是指交换内存，就是拿硬盘充当内存，使用k8s需要关闭swap。
1. 检查本机是否开启swap：执行free命令，有swap这一列就是开启了swap。
2. 暂时关闭虚拟内存：`swapoff -a`
3. 永久关闭：

设置开机启动时不要使用虚拟内存(在/etc/fstab中用#注释掉带swap字样的行):
```
sed -i '/swap/s/^/#/' /etc/fstab
```


## 安装配置docker和cri-docker
cri是container runtime interface，是k8s使用容器的接口。而docker不知道为什么没有实现cri，因此docker需要一个额外的软件cri-docker，来让k8s能控制docker。
除了docker，k8s的另一个常用的容器运行时是[containerd](https://containerd.io/)，containerd实现了cri，由于生态原因，暂时不考虑使用containerd。
### 安装和配置docker
- 通过清华镜像安装docker，安装方法参考[清华镜像文档](https://mirrors.tuna.tsinghua.edu.cn/help/docker-ce/) 
```bash
# 删除旧docker
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do apt-get remove $pkg; done

# 信任并添加apt源
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装docker
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
接下来要配置docker的cgroup驱动

> 为什么要配置cgroup：
> 1. cgroup或者cgroups意为Linux Control Groups，属于Linux内核机制，负责管理资源的分配，是虚拟化技术的两大基石之一，另一个是namespace。Linux内核的cgroups API有两个版本，一个v1一个v2
> 2. cgroup有两种常见驱动，一个是systemd，另一个是cgroupfs。cgroupfs是Linux内核cgroups接口的原始实现，cgroupfs提供了底层的、更细粒度的控制，但不如systemd易用。
> 3. docker和k8s需要使用cgroup来管理容器和pod，需要确保他们使用的是同一个cgroup驱动，如果一个用cgroupfs驱动一个用systemd驱动会导致资源分配冲突。
> 4. 如果当前系统的初始化程序是systemd，则docker和k8s应该都将cgroup驱动设置为systemd。（如果能正常使用systemctl命令，则说明初始化程序是systemd而不是initV）
> 参考：
> 1. [配置 cgroup 驱动 | Kubernetes](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/)
> 2. [容器运行时 | Kubernetes](https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#cgroup-drivers)

```bash
cat > /etc/docker/daemon.json <<EOF
{
    "exec-opts":[
        "native.cgroupdriver=systemd"
    ]
}
EOF
systemctl restart docker
```
#### 检验安装配置docker是否成功
只要dockerd（docker的守护进程）启动了就好，检测dockerd是否启动的方法很多，其中一个方法是执行以下命令
```bash
docker ps
```
参考输出如下：
```bash
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
```
这个命令用于查看正在运行的容器，目前没有容器在运行所以会这样显示，如果dockerd不在运行则会输出
```bash
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```
### 安装cri-dockerd
参考 [cri-docker官方安装文档](https://mirantis.github.io/cri-dockerd/usage/install-manually/) 和 [cri-docker github页面](https://github.com/Mirantis/cri-dockerd)

安装cri-docker操作步骤：
1. 下载二进制文件：
```bash
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.15/cri-dockerd-0.3.15.amd64.tgz
```
2. 解压`tar -xf cri-dockerd-0.3.15.amd64.tgz`
3. 安装`install -o root -g root -m 0755 cri-dockerd/cri-dockerd /usr/local/bin/cri-dockerd`
4. 配置服务
```bash
cat > /etc/systemd/system/cri-docker.service <<EOF
[Unit]
Description=CRI Interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
Requires=cri-docker.socket

[Service]
Type=notify
ExecStart=/usr/local/bin/cri-dockerd --container-runtime-endpoint fd://
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

# Note that StartLimit* options were moved from "Service" to "Unit" in systemd 229.
# Both the old, and new location are accepted by systemd 229 and up, so using the old location
# to make them work for either version of systemd.
StartLimitBurst=3

# Note that StartLimitInterval was renamed to StartLimitIntervalSec in systemd 230.
# Both the old, and new name are accepted by systemd 230 and up, so using the old name to make
# this option work for either version of systemd.
StartLimitInterval=60s

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not support it.
# Only systemd 226 and above support this option.
TasksMax=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
EOF

cat > /etc/systemd/system/cri-docker.socket <<EOF
[Unit]
Description=CRI Docker Socket for the API
PartOf=cri-docker.service

[Socket]
ListenStream=%t/cri-dockerd.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target
EOF
```
5. 重启cri-docker
```bash
systemctl daemon-reload
systemctl enable --now cri-docker.socket
systemctl start cri-docker
```
#### 检验cri-docker是否安装成功
```bash
systemctl status cri-docker
```
如果正在运行会显示running，形如：
```
● cri-docker.service - CRI Interface for Docker Application Container Engine
     Loaded: loaded (/etc/systemd/system/cri-docker.service; disabled; preset: enabled)
     Active: active (running) since Sun 2024-11-24 07:28:15 UTC; 6h ago
TriggeredBy: ● cri-docker.socket
       Docs: https://docs.mirantis.com
   Main PID: 3584 (cri-dockerd)
      Tasks: 9
     Memory: 16.7M (peak: 17.3M)
        CPU: 56.851s
     CGroup: /system.slice/cri-docker.service
             └─3584 /usr/local/bin/cri-dockerd --container-runtime-endpoint fd://
```
### 国内如何通过github安装cri-docker
可以看到上面的cri-docker来自github，服务器网络环境不好的话也访问不到github。

方法一，通过cloudflare免费搭建一个github国内转发，方法参考[这个视频](https://www.bilibili.com/video/BV1xZtCecEES)，搭建成功后的效果就是你可以把任何github链接换成自己的代理地址来加速访问。

方法二就是土方法了，自己在有代理的电脑上手动下载安装包再传过去。

方法三，通过公共镜像服务，比如gh-proxy.com，下载github中的内容。

## 安装kubeadm、kubelet和kubectl
参考 [安装kubeadm、kubelet和kubectl | Kubernetes](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)
- kubeadm：由于k8s安装过于复杂而专门出现的辅助安装工具
- kubelet：k8s的守护进程，kubectl和kubelet的关系就像docker和dockerd。
- kubectl：能够控制k8s集群的命令行工具。

对于使用apt和yum包管理器的Linux发行版，官方提供了软件仓库，国内也有很多k8s软件仓库的镜像源，下面是apt通过[清华镜像源](https://mirrors.tuna.tsinghua.edu.cn/help/kubernetes/)安装kubeadm、kubelet和kubectl的命令。

可以把 `1.31` 修改成任意 [清华镜像源的文件目录](https://mirrors.tuna.tsinghua.edu.cn/kubernetes/core%3A/stable%3A/) 中存在的版本
```bash
curl -fsSL https://mirrors.tuna.tsinghua.edu.cn/kubernetes/core%3A/stable%3A/v1.31/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

cat > /etc/apt/sources.list.d/kubernetes.list <<EOF
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/kubernetes/core:/stable:/v1.31/deb/ /
EOF

apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
systemctl enable kubelet
```

## 拉取k8s所用的镜像

执行该命令可以拉取运行k8s所需的镜像，由于拉取镜像操作需要使用docker，k8s并不原生支持docker，我们需要手动指定kubeadm通过cri-docker来控制docker。
```bash
kubeadm config images pull --cri-socket unix:///var/run/cri-dockerd.sock
```

除了kubelet和kubectl，k8s的其他组件基本都在容器内运行，接下来的安装过程中会自动拉取，但是由于国内网络原因很难直接拉取成功，我们可以查看k8s需要哪些镜像，然后通过其他方法获取镜像。
### 查看需要哪些镜像
```bash
kubeadm config images list
```

输出形如：
```
registry.k8s.io/kube-apiserver:v1.31.3
registry.k8s.io/kube-controller-manager:v1.31.3
registry.k8s.io/kube-scheduler:v1.31.3
registry.k8s.io/kube-proxy:v1.31.3
registry.k8s.io/coredns/coredns:v1.11.3
registry.k8s.io/pause:3.10
registry.k8s.io/etcd:3.5.15-0
```

> 如果你想知道kubeadm还有哪些其他命令，可以查看k8s官方文档->参考->安装工具->[kubeadm](https://kubernetes.io/zh-cn/docs/reference/setup-tools/kubeadm/)。其中`init`、`config` 命令会比较常用，可以细看。
### 通过其他方式获取镜像
#### 手动传
方法一，你可以在有代理的机器上拉取上面的镜像，然后通过 `docker save`命令把镜像导出为文件，把文件转移到要部署的服务器上再通过`docker load`命令导入，即可获取镜像。
#### 国内镜像源
方法二，设置国内镜像源。
1. 要注意的是k8s的镜像并不放在dockerhub中，k8s有自己的镜像仓库，因此对dockerhub的加速并不适用。
2. 阿里云有k8s镜像仓库的镜像源你可以通过这个命令来使用阿里云拉取需要的镜像
```bash
kubeadm config images pull --image-repository registry.aliyuncs.com/google_containers --cri-socket unix:///var/run/cri-dockerd.sock
```
`--cri-soket`参数是必须的，这个参数告诉kubeadm要控制哪一个容器运行时，提供的这个地址是cri-docker默认的，这一点无需过多操心。
拉取镜像之后你可以通过以下命令查看拉取到的镜像
```bash
docker images
```
拉取到镜像之后需要重新tag，否则后续会安装不成功。比如当前需要的`pause`镜像版本是3.10，我通过阿里云拉取到的是
```txt
registry.aliyuncs.com/google_containers/pause:3.10
```
而后续在执行`kubeadm init`的时候它会去尝试拉取
```txt
registry.k8s.io/pause:3.9
```
但是又拉取不到，就会导致安装失败。解决办法是把阿里云的前缀重新tag成k8s镜像仓库的前缀，参考命令：
```bash
docker tag registry.aliyuncs.com/google_containers/pause:3.10 registry.k8s.io/pause:3.10
```

把上面所有需要的镜像都重新tag成他要拉取的镜像即可。

#### 镜像版本问题
这里可能会遇到镜像版本问题，你通过`kubeadm config images pull`拉取到的镜像和后面kubeadm init的时候实际拉取的镜像版本不同，因此这里提前拉取这一步基本是没用的。。必须要在实际拉取的时候看docker日志才能获知实际采用的版本。

#### 总结
因此通过国内源拉取镜像的命令如下（k8s版本为1.31）
```bash
docker pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.31.0
docker pull registry.aliyuncs.com/google_containers/kube-controller-manager:v1.31.0
docker pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.31.0
docker pull registry.aliyuncs.com/google_containers/kube-proxy:v1.31.0
docker pull registry.aliyuncs.com/google_containers/coredns:v1.11.3
docker pull registry.aliyuncs.com/google_containers/etcd:3.5.15-0
docker pull registry.aliyuncs.com/google_containers/pause:3.9

docker tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.31.0 registry.k8s.io/kube-apiserver:v1.31.0
docker tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.31.0 registry.k8s.io/kube-controller-manager:v1.31.0
docker tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.31.0 registry.k8s.io/kube-scheduler:v1.31.0
docker tag registry.aliyuncs.com/google_containers/kube-proxy:v1.31.0 registry.k8s.io/kube-proxy:v1.31.0
docker tag registry.aliyuncs.com/google_containers/coredns:v1.11.3 registry.k8s.io/coredns/coredns:v1.11.3
docker tag registry.aliyuncs.com/google_containers/etcd:3.5.15-0 registry.k8s.io/etcd:3.5.15-0
docker tag registry.aliyuncs.com/google_containers/pause:3.9 registry.k8s.io/pause:3.9
```

---
通过其他方式获取镜像，方法三：类似之前的github代理，你可以搭建一个自己的docker代理，但是指向k8s的镜像仓库而不是dockerhub的镜像仓库。
## 在master节点创建集群
1. 创建集群对应的命令是`kubeadm init`，这条命令的执行是一个[很复杂的过程](https://blog.csdn.net/shida_csdn/article/details/83176735)，之前的任何一步出错都有可能导致`kubeadm init`执行失败。
2. init的过程可以使用yaml文件也可以使用命令行参数来进行配置。可用的命令行参数参考[kubeadm init | Kubernetes](https://kubernetes.io/zh-cn/docs/reference/setup-tools/kubeadm/kubeadm-init/)，kubeadm的配置文件的解释请参考[kubeadm 配置 (v1beta4) | Kubernetes](https://kubernetes.io/zh-cn/docs/reference/config-api/kubeadm-config.v1beta4/)，一个可供参考的`kubeadm-config.yaml`文件如下
```
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta4
kubernetesVersion: v1.31.0
networking:
  serviceSubnet: "10.96.0.0/16" # service用什么网段
  podSubnet: "10.244.0.0/24" # pod用什么网段
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///var/run/cri-dockerd.sock # 需要手动指定cri套接字，这是cri-dockerd默认的套接字
```
3. 写好配置文件之后执行以下命令来开始创建集群
```bash
kubeadm init --config kubeadm-config.yaml
```
### 如果kubeadm init成功
恭喜你创建集群成功，最容易出问题的部分结束了。
1. 按照init成功的提示，复制kubelet的配置文件
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
2. 保存提示中出现的join命令，其他节点将要通过kubeadm join命令来加入集群，提示中出现的join命令包含了token，形如：
```bash
kubeadm join 192.168.2.100:6443 --token v6n70f.k27dhe1gzb93frpl \
        --discovery-token-ca-cert-hash sha256:d3ae78040f88424c7321939128ff17d8798241253b07522e5130a9bc0e69301e
```
### 如果kubeadm init失败
如果失败，你需要重置kubeadm init操作，检查之前的步骤是否成功，检查kubelet日志，以及重新理解整个安装过程发生了什么，为什么这么做。
用下面这个命令重置kubeadm init的操作
```bash
kubeadm reset --cri-socket unix:///var/run/cri-dockerd.sock
```
用下面这个命令查看kubelet日志
```bash
systemctl status kubelet
```
### 一次kubeadm init失败debug的示例
问题是`kubeadm init`之后卡在了这一句
```txt
[api-check] Waiting for a healthy API server. This can take up to 4m0s
```
上网翻了一些别人的debug文章，这一篇给了很多思路：
[k8s单机部署报错，连接不上6443接口序言 不知到我脑子抽什么风了要来整K8s，又不知道出什么毛病了启动一直报错，整的 - 掘金](https://juejin.cn/post/7158425242003013639)（而且这篇文章给了很多其他有用的参考文章）

看完一些相关文章，可以判断出目前情况是api server启动不起来，应该`systemctl status kubelet`看一下kubelet的日志，在日志中找到了蛛丝马迹
```
failed pulling image \"registry.k8s.io/pause:3.9\": Error response from daemon: Head \"https://us-west2-docker.pkg.dev/v2/k8s-artifacts-prod/images/pause/manifests/3.9\": dial tcp 74.125.199.82:443: connect: connection refused"
```
发现kubelet拉取`registry.k8s.io/pause:3.9`失败，可是实际应该使用的是3.10的pause，结合kubeadm init命令回显中的一些提示，尝试重新tag镜像前缀，把阿里云的前缀改成k8s镜像仓库的前缀，发现问题解决。
## 其他节点加入集群
去其他节点，执行之前的步骤，关闭swap、安装配置docker和cri-docker、拉取k8s镜像，然后在其他节点执行之前复制的join命令，但要有所变化，需要指定cri-socket，形如
```bash
kubeadm join 192.168.2.100:6443 --token v6n70f.k27dhe1gzb93frpl \
        --discovery-token-ca-cert-hash sha256:d3ae78040f88424c7321939128ff17d8798241253b07522e5130a9bc0e69301e \
--cri-socket unix:///var/run/cri-dockerd.sock
```
### 验证是否加入集群
在master执行
```bash
kubectl get node
```
可以看到其他节点已经加入集群：
```txt
NAME      STATUS     ROLES           AGE    VERSION
master    NotReady   control-plane   6m1s   v1.31.3
worker1   NotReady   <none>          37s    v1.31.3
worker2   NotReady   <none>          35s    v1.31.3
```
## 安装CNI插件
### CNI插件的作用
现在在主节点执行`kubectl get node`已经可以看到子节点了，但是status是NotReady，是因为没有安装CNI插件（[Container network interface](https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)）。
1. k8s具有一个复杂的[网络模型](https://kubernetes.io/zh-cn/docs/concepts/services-networking/)，其中一部分负责Pod之间的网络（称为pod网络或者集群网络），pod网络负责两点：
2. 第一，集群内所有pod，无论是否在同一节点，都可以直接相互通信而无需代理和NAT
3. 第二，节点上的代理（比如kubelet）可以直接与本节点的pod通信
4. pod网络的实现有很多，他们被统称为CNI插件，k8s官方文档提供了[一个不完全的CNI插件列表](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/addons/#networking-and-network-policy)。
### 安装calico的步骤
CNI插件选择calico，安装过程参考 [Quick Start| Calico Docs](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)
1. 安装Tigera Calico operator
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/tigera-operator.yaml
```
注意这个配置文件很长，不可以用kubectl apply，可以用create和replace。
2. 安装custom resource，这部分是自定义配置，需要修改后再安装。获取[yaml文件](https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/custom-resources.yaml)并编辑：
```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 26
      # cidr改成之前在kubeadm config 设置的pod网段
      cidr: 10.244.0.0/24
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
```
3. 安装custom resource
```bash
kubectl apply -f custom-resources.yaml
```
4. 通过这个命令查看pod状态，当全部running的时候就成功了。
```bash
watch kubectl get pods -n calico-system
```
### 一个安装失败的debug示例
查看pod状态会显示
```
No resources found in calico-system namespace.
```
没有calico-system命名空间。我们可以尝试查看所有命名空间：
```bash
kubectl get ns
```
输出：
```
NAME              STATUS   AGE
default           Active   69m
kube-node-lease   Active   69m
kube-public       Active   69m
kube-system       Active   69m
tigera-operator   Active   50m
```
可以看到有一个tigera-operator可能和calico相关，可以查看这个命名空间有哪些pod。
执行：
```bash
kubectl get all -n tigera-operator
```
输出：
```txt
NAME                                   READY   STATUS              RESTARTS   AGE
pod/tigera-operator-76c4976dd7-t699c   0/1     ContainerCreating   0          44m

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/tigera-operator   0/1     1            0           51m

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/tigera-operator-76c4976dd7   1         1         0       51m
```
可以看到有一个pod，可以发现他的状态一直卡在ContainerCreating。
查一下官网，了解pod有哪些状态，参考[Pod 的生命周期 | Kubernetes](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/)
尝试查看pod的日志：
```bash
kubectl logs pod/tigera-operator-76c4976dd7-t699c -n tigera-operator
```
输出：
```
Error from server (BadRequest): container "tigera-operator" in pod "tigera-operator-76c4976dd7-t699c" is waiting to start: ContainerCreating
```
没有办法，google搜索这个报错，出现一个stackoverflow的问题（找不到链接了）高赞回答提示可以用 kubectl describe命令查看pod的信息。
执行
```bash
kubectl describe pod/tigera-operator-76c4976dd7-t699c -n tigera-operator
```
输出很长，发现最后有类似日志的东西，叫做Events，里面有关键信息：
```
Events:
  Type     Reason                  Age                   From               Message
  ----     ------                  ----                  ----               -------
  Normal   Scheduled               9m57s                 default-scheduler  Successfully assigned tigera-operator/tigera-operator-76c4976dd7-t699c to worker1
  Warning  FailedCreatePodSandBox  22s (x16 over 9m34s)  kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = failed pulling image "registry.k8s.io/pause:3.9": Error response from daemon: Head "https://us-west2-docker.pkg.dev/v2/k8s-artifacts-prod/images/pause/manifests/3.9": dial tcp 142.251.8.82:443: connect: connection refused
```
发现子节点试图拉取`registry.k8s.io/pause:3.9`失败，pause是每一个pod中都会存在的容器，它的作用参考：

[Kubernetes中的Pause容器到底是干嘛的-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2332162)

检查镜像发现已经有`registry.k8s.io/pause:3.10`了，不知道为什么又要3.9，从其他机器pull下来然后装上。重新查看describe，发现进入了下一个event，pod状态也进入running了，原本应该有的calico-system命名空间也出现了。

然后calico-system下的几个pod也不顺利，如法炮制，用相似的办法解决。
容器有点多，节点也多，传起来很麻烦，建议自建个仓库方便一些，只需要传到这个仓库或者给这个仓库做代理，其他的节点就都可以拉取到了。

## 参考
- [containerd官网](https://containerd.io/)
- [清华镜像：docker-ce帮助文档](https://mirrors.tuna.tsinghua.edu.cn/help/docker-ce/) 
- [清华镜像：安装kubeadm、kubelet和kubectl帮助文档](https://mirrors.tuna.tsinghua.edu.cn/help/kubernetes/)
- [清华镜像：k8s各版本文件目录](https://mirrors.tuna.tsinghua.edu.cn/kubernetes/core%3A/stable%3A/)
- [k8s官方文档：如何配置cgroups驱动](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/)
- [k8s官方文档：容器运行时 - 介绍cgroup驱动](https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#cgroup-drivers)
- [k8s官方文档：安装kubeadm、kubelet和kubectl](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)
- [k8s官方文档：kubeadm](https://kubernetes.io/zh-cn/docs/reference/setup-tools/kubeadm/)
- [k8s官方文档：kubeadm init 命令行参数说明](https://kubernetes.io/zh-cn/docs/reference/setup-tools/kubeadm/kubeadm-init/)
- [k8s官方文档：kubeadm init 配置文件参考](https://kubernetes.io/zh-cn/docs/reference/config-api/kubeadm-config.v1beta4/)
- [k8s官方文档：CNI网络插件](https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
- [k8s官方文档：Kubernetes 网络模型](https://kubernetes.io/zh-cn/docs/concepts/services-networking/)
- [k8s官方文档：不完全的CNI插件列表](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/addons/#networking-and-network-policy)
- [k8s官方文档：pod的生命周期](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/)
- [cri-docker官方安装文档](https://mirantis.github.io/cri-dockerd/usage/install-manually/)
- [cri-docker github页面](https://github.com/Mirantis/cri-dockerd)
- [cloudflare搭建github代理视频教程](https://www.bilibili.com/video/BV1xZtCecEES)
- [calico官方文档：快速安装指南](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)
- [网络文章：K8S 源码探秘 之 kubeadm init 执行流程分析](https://blog.csdn.net/shida_csdn/article/details/83176735)
- [网络文章：k8s单机部署报错，连接不上6443接口](https://juejin.cn/post/7158425242003013639)
- [网络文章：Kubernetes中的Pause容器到底是干嘛的](https://cloud.tencent.com/developer/article/2332162)