# 国内服务器网络问题

国内服务器网络问题主要指的是，服务器上不容易访问国外的git仓库、软件包仓库和容器镜像仓库。
## 公益镜像和代理

网上有很多常见的公益镜像和代理，这是一个比较入门的方式，这里只列举一些可用的公益平台：
1. github：[http://gh-proxy.com](http://gh-proxy.com)
2. dockerhub：[DaoCloud/crproxy](https://github.com/DaoCloud/crproxy)
3. 软件包：[清华pip镜像](https://mirrors.tuna.tsinghua.edu.cn/help/pypi/)，[阿里npm镜像](https://npmmirror.com/) 

## 部署 worker 代理

这也是一个传播较广的方式， [cf-workers-proxy](https://github.com/jonssonyan/cf-workers-proxy) 允许通过 cloudflare worker 部署一个通用的代理，部署成功之后，比如目标网站是dockerhub，只需要修改docker的registry-mirror，或者把每次pull的domain修改为worker的地址，就可以让客户端使用这个代理了。具体使用方式参考项目的readme。

但是 cloudflare worker 目前已经禁止用于代理，可以参考他的代码在自己的服务器上写一个类似的代理。

## 部署透明代理

大多数包管理器和客户端都可以配置proxy，只需要自己在服务器上部署一个clash/mihomo（推荐使用 [clash for linux install](https://github.com/nelvko/clash-for-linux-install)）然后修改客户端的 proxy 为这个 clash 即可。

为 git 和 docker 设置代理的教程数量浩瀚如烟，但这篇文章 [如何优雅的给 Docker 配置网络代理 - CharyGao - 博客园](https://www.cnblogs.com/Chary/p/18096678) 对 docker 各个时期配置代理的方式总结的很全，值得推荐。


## pull through cache

Pull through cache 的意思是当收到客户端请求时，会先检查本地有没有缓存，如果没有就向一个或多个上游仓库中查询并代理请求，第二次拉取时就会使用本地的缓存了。

对于docker，[Distribution](https://github.com/distribution/distribution)（原名 Registry，docker官方开源的容器镜像仓库）就支持pull through cache，如果需要更高级的权限管理、图形化界面等企业级需求，可以考虑使用基于Distribution 的 [Harbor](https://github.com/goharbor/harbor) ，但代价是更高的资源占用。[zot](https://github.com/project-zot/zot) 也是一个不错的选择。

对于软件包，每个包管理器通常都有自己的开源项目，大多都能够满足pull through cache的需求。

也有一个能够同时结合许多软件包的仓库 Sonatype Nexus Repository，[Nexus 官方文档页: formats](https://help.sonatype.com/en/formats.html) 中的表格罗列了对所有类型仓库的支持程度，有相当一部分都支持pull through cache。但是 nexus 的文档管理和项目文件结构看起来都相当混乱，在社区看起来也有不少[抱怨之声](https://www.reddit.com/r/devops/comments/1q6e1xn/what_is_your_thoughts_on_nexus_sonatype/?tl=zh-hans)。

