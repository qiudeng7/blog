# Prometheus

[Prometheus](https://prometheus.io/docs/introduction/overview/) 是一个毕业于 [CNCF](https://www.cncf.io/projects/) 的开源的系统监控和警报工具，它是 CNCF 的第二个项目（第一个是 Kubernetes），另外 CNCF 还有一个 [Jaeger](https://www.jaegertracing.io/) 项目用于请求链路分析，Prometheus 主要用于监控系统性能。

Prmethues 常与 Grafana 搭配使用，实际生产中一般通过 [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) 进行部署。

## 概念

Prometheus 的核心是一个时间序列数据库 (TSDB)。在 TSDB 中，一个 metric (指标) 对应一条时间序列，时间序列上的每一条数据被称为样本（Sample），每一个样本都包含时间戳和一些索引（label）。比如 `api_http_requests_total` （某个 api 收到的全部请求）就是一个 metric，某一个具体的请求就是一个 Sample，请求中包含这些 label： `method="POST"`, `code="200"`。

所以我们可以用这样的格式对 Prometheus 进行查询:
```
<metric name>{<label name>="<label value>", ...}
```

有这几种 metric：
1. **Counter 计次**，比如请求次数
2. **Gauge 计量**，比如 CPU 温度
3. **Histogram 直方图**，需要我们预定义几个区间，新的数据到来时对区间进行操作。
4. **Summary 摘要**，类似 Histogram，但是数据不存在 Prometheus 中，而是 Prometheus 直接向客户端查询某一个分位数的值。

## 架构

Prometheus 的架构如下

1. **Prometheus Server**: Prometheus 主体，核心是一个时序数据库。
2. **Exporters**: 数据导出器，Prometheus 为各种语言提供了客户端，方便构建 exporter 作为 Prometheus 的数据源。
3. **Pushgateway**: Prometheus 的设计是长期地向客户端 Pull 数据，但是对于短期任务，客户端可以把数据 push 到 pushgateway，适用场景比如一些短期执行任务脚本，或者特别的物联网设备。
4. **Service Discovery**: Prometheus 可以直接通过 Kubernetes 进行服务发现，自动找到 Kubernetes 中的资源进行监控。
5. **Alertmanager**：用于警告和通知
6. **Grafana**：通过 PromQL 语言查询数据，并汇总成图表。

![prometheus 架构图|700x420](https://prometheus.io/assets/docs/architecture.svg)

## 手动部署

>  下面的部署过程理论上是可以通过的，但是我测试的时候并没有成功，原因未知。由于手动部署的方式并不重要，我没有仔细研究，读者通过部署过程大致了解 Prometheus 工作方式即可。或者可以尝试[官方文档的部署方式](https://prometheus.io/docs/prometheus/latest/getting_started/#using-the-graphing-interface)

第一步，先获取 Prometheus 配置文件
```bash
mkdir prom && cd prom
docker run -d --name temp_prometheus prom/prometheus
docker cp temp_prometheus:/etc/prometheus/prometheus.yml ./prometheus.yml
docker stop temp_prometheus
docker rm temp_prometheus
```

第二步，用 kind 创建集群
```bash
cat > kind.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
EOF

kind create cluster --config kind.yaml
```

第三步，在 k8s 创建一个 prometheus 服务账户，先写一个资源清单
```yaml
# prometheus-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-k8s  # 角色名称
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - nodes/metrics
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
```

创建身份并获取 token：
```bash
# 创建namespace
kubectl create ns prom-test
# 创建ServiceAccount
kubectl create sa prometheus -n prom-test
# 创建clusterRole
kubectl apply -f prometheus-clusterrole.yaml
# 绑定clusterRole
kubectl create clusterrolebinding prometheus-k8s \
  --clusterrole=prometheus-k8s \
  --serviceaccount=prom-test:prometheus
# 获取token并输出到 token.txt
kubectl create token prometheus -n prom-test > token.txt
```

第四步，编辑 Prometheus 配置，主要修改 scrape_configs 部分，这里添加了一个 job ，并且配置了 k8s 服务发现
```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.

scrape_configs:
  
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  
  - job_name: "k8s-nodes"
    scheme: https
    tls_config:
      insecure_skip_verify: true
    bearer_token_file: /etc/prometheus/k8stoken
    
    # k8s服务发现配置
    kubernetes_sd_configs:
      - role: node
        api_server: "127.0.0.1:6443"
        tls_config:
          insecure_skip_verify: true
        bearer_token_file: /etc/prometheus/k8stoken
```

通过下面这个 compose.yaml 启动 Prometheus
```yaml
services:
  prometheus:
    image: prom/prometheus
    network_mode: host
    ports:
      - 9090:9090
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./token.txt:/etc/prometheus/k8stoken
```

然后就可以打开 https://localhost:9090 查看 Prometheus 了。

## kube-prometheus-stack

[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) 是一个 helm chart，它是 [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus) 项目的 helm 形式。

kube-prometheus 的设计目的是构建出一个 k8s 资源清单，能够一键部署在 k8s 场景下的常用 Prometheus 资源。包括以下资源：

| 资源                  | 作用                                                                    |
| :------------------ | :-------------------------------------------------------------------- |
| Prometheus Operator | 整个监控栈的大脑，负责自动创建和管理Prometheus、Alertmanager等资源，让监控配置变得声明式和自动化。          |
| Prometheus Server   | 监控系统的核心，负责抓取、存储时间序列数据，并根据规则评估和触发告警。                                   |
| Alertmanager        | 专用于处理由Prometheus发送来的告警，负责进行去重、分组、静默，并通过路由策略将告警通知发送给正确的人。              |
| Grafana             | 数据可视化平台，内置了Prometheus数据源和大量预制的监控仪表盘，用于将监控数据以图形化方式直观展示。                |
| kube-state-metrics  | 监听Kubernetes API，生成关于集群内资源对象（如Deployment、Pod状态、资源请求/限制）的指标，关注的是资源的状态。 |
| node-exporter       | 以DaemonSet形式运行在每个节点上，用于收集节点级别的硬件和操作系统指标（如CPU、内存、磁盘、网络流量）。             |

部署过程如下：

第一步，得到一个 k8s 集群，要求开启 `authentication-token-webhook=true` 和 `authorization-mode=Webhook`。

>  问题:我不理解为什么要开启这两个项目，他们原本的作用是什么，在这个场景下又是怎么工作的？
>  
> 核心作用：安全控制
> 
> 原理解析
> 
> 认证（authentication-token-webhook）
> • 作用：验证你是谁
> 
> • 机制：kubelet 问 API Server"这个 token 有效吗？"
> 
> • 场景：Prometheus 用 ServiceAccount token 证明身份
> 
> 授权（authorization-mode=Webhook）  
> • 作用：检查你能做什么
> 
> • 机制：kubelet 问 API Server"这个用户能访问 /metrics 吗？"
> 
> • 场景：验证 Prometheus 是否有权限读指标
> 
> 工作流程
> 
> 
> Prometheus → kubelet(/metrics)
>            ↓ 提供 ServiceAccount token
> kubelet → API Server"验证token+权限？"
>            ↓ 
> API Server → 检查RBAC规则
>            ↓
> 允许/拒绝访问指标
> 
> 
> 对比传统方式
> 
> • 旧方式：客户端证书 → 过度授权（完全kubelet访问权）
> 
> • 新方式：token+RBAC → 最小权限原则（仅指标读取权）
> 
> 本质：用K8s原生RBAC精细控制指标访问权限，避免安全风险。
>

通过 kind 创建符合要求的集群，kind的基本使用参考 https://kind.sigs.k8s.io/ 

先创建 `kind-config.yaml`

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        authentication-token-webhook: "true"
        authorization-mode: "Webhook"
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        authentication-token-webhook: "true"
        authorization-mode: "Webhook"
- role: worker
```

启动集群
```bash
kind create cluster --config ./kind-config.yaml
```

直接helm部署
```bash
helm install [RELEASE_NAME] oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack
```
部署成功之后可以看到上面提到的资源都会被创建，我们主要可以访问的是Prometheus, Grafana 和 AlertManager，想要访问他们需要转发端口。

转发Grafana的端口：
```bash
kubectl port-forward service/<RELEASE_NAME>-grafana -n <NAMESPACE> 3000:80
```

通过 http://localhost:3000 打开Grafana UI，账号是`admin`，密码通过以下命令获取。

```bash
kubectl get secret <RELEASE_NAME>-grafana -n <NAMESPACE> -o jsonpath="{.data.admin-password}" | base64 --decode
```

打开可以看到 Grafana 已经自动连接到了 Prometheus 和 AlertManager，Prometheus 也已经连接上 NodeExporter 和各个k8s组件。

还对这些默认提供的数据预置了一些监控面板的前端，大体上分为 k8s系统、计算资源、网络资源、监控系统本身 这几个部分的面板，可以自行探索。

## 其他

我发现Grafana可以对接ClickHouse，文档里的示例图片如下，各种数据库都可以接

![Grafana-ClickHouse](assets/Prometheus/Grafana-ClickHouse.png)


CNCF的官网指向了一个叫做 [CNCF cloud native landscape](https://landscape.cncf.io/) 的项目，中文可以翻译为CNCF云原生全景图该项目的目标是收集和组织所有云原生开源项目，将源项目和专有产品归类，并提供概览。

除了这张图，项目还提供了一个文档来全面的解释云原生领域各个类别的定义、解决的问题、优势以及基本的入门教程，前三部分假定读者没有任何技术背景，而技术入门部分则面向刚刚接触云原生技术的工程师。

