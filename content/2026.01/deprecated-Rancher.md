# Rancher

问题背景：
1. 我的需求是通过便宜的竞价实例创建服务器池，用服务器池来搭建k8s集群。
2. 我尝试手搓服务器池和k8s部署
3. 发现腾讯云有自带的集群管理，尝试之后发现太贵了。
4. 发现腾讯云自带的免费的弹性伸缩可以当做服务器池，我只需要手写部署k8s。
5. 发现部署 k8s 可以用 KubeSphere
6. 不太喜欢 KubeSphere，发现了同类的 Rancher，似乎更好用。

> 弹性伸缩不能当服务器池，rancher也对腾讯云支持不好。

---

## 部署 Rancher Server

1. [离线安装](https://ranchermanager.docs.rancher.com/zh/getting-started/installation-and-upgrade/other-installation-methods/air-gapped-helm-cli-install)
2. [各类云平台安装](https://ranchermanager.docs.rancher.com/zh/getting-started/quick-start-guides/deploy-rancher-manager)
3. [手动在不支持的云平台上安装](https://ranchermanager.docs.rancher.com/zh/getting-started/quick-start-guides/deploy-rancher-manager/helm-cli)

Rancher Server需要部署在 k8s 上，官方推荐使用k3s。

方便起见先装个梯子，参考 [nelvko/clash-for-linux-install](https://github.com/nelvko/clash-for-linux-install)
```bash
git clone --branch master --depth 1 https://gh-proxy.org/https://github.com/nelvko/clash-for-linux-install.git \
  && cd clash-for-linux-install \
  && bash install.sh
```

#### 安装/卸载k3s

```bash
# 设置为你的服务器公网IP或地址，这个地址用于你通过本地kubectl访问k3s
K3S_PUBLIC_IP="${K3S_PUBLIC_IP:-119.45.125.220}"

# 安装k3s
curl -sfL https://get.k3s.io | sh -s - server --cluster-init

# 调整tls配置
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml <<EOF
tls-san:
  - ${K3S_PUBLIC_IP}
EOF

sudo systemctl restart k3s

# 允许普通用户访问kubeconfig
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
kubectl get nodes
```

如果你需要卸载，则执行 `k3s-uninstall`，安装k3s的时候自动添加了这个命令。

然后你就可以在本地的kubectl访问该k3s了，下面是一个获取kubeconfig然后用kubectl访问它的示例

```bash
scp ubuntu@119.45.125.220:/etc/rancher/k3s/k3s.yaml .
sed -i 's/127.0.0.1/119.45.125.220/g' ./k3s.yaml

kubectl get nodes --kubeconfig ./k3s.yaml
```

### TLS配置说明

如果不配置 TLS，本地尝试访问 k3s 会报错
```
kubectl get nodes --kubeconfig ./k3s.yaml

E0122 08:47:09.227432   22490 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"https://119.45.125.220:6443/api?timeout=32s\": tls: failed to verify certificate: x509: certificate is valid for 10.206.0.4, 10.43.0.1, 127.0.0.1, ::1, not 119.45.125.220"
```

错误原因是，kubectl知道自己连接的是ip A，但是从ip A获取到的TLS证书中并不包含ip A，所以kubectl认为该集群是不可信任的，我要么在集群证书中添加ip A，要么让kubectl无视风险。

如果要无视风险，则添加 --insecure-skip-tls-verify 参数
```bash
kubectl get nodes --kubeconfig ./k3s.yaml --insecure-skip-tls-verify=true
```

如果要配置证书，需要在k3s配置文件(`/etc/rancher/k3s/config.yaml`，没有则新建)的末尾添加以下内容然后重启k3s (`systemctl restart k3s`)
```yaml
tls-san:
  - 119.45.125.220
```

### 安装certmanager

rancher要求必须通过https访问，通过certmanager管理证书，所以需要先安装certmanager。
```bash

# ubuntu/debian 安装helm
# 其他系统参考 https://helm.sh/docs/intro/install
sudo apt-get install curl gpg apt-transport-https --yes
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

# 添加certmanager包
helm repo add jetstack https://charts.jetstack.io
helm repo update

# 安装certmanager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.19.2/cert-manager.crds.yaml
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --kubeconfig /etc/rancher/k3s/k3s.yaml
```

### 安装rancher server

这里 hostname 必须要用域名，临时环境你可以用伪域名比如 `119.45.125.220.sslip.io`，这种伪域名会被DNS直接解析成`119.45.125.220`

另外下面的命令自动使用m.daocloud.io对rancher镜像进行加速。

bootstrapPassword自己填，不少于12位。

```bash
# 添加rancher helm包
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest

# 安装rancher
kubectl create namespace cattle-system
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=[你的域名] \
  --set replicas=1 \
  --set bootstrapPassword=mxocB3kAF2QVJry0PMbQ \
  --set image.repository=m.daocloud.io/docker.io/rancher/rancher \
  --kubeconfig /etc/rancher/k3s/k3s.yaml
```

所有pod就绪之后就可以通过 `119.45.125.220.sslip.io` 访问了

