# Gitea

因为需要搭建cicd，以及最近（2025年4月14日）github疑似主动限制中国IP访问（以前都是GFW导致无法访问，这次是github主动的），于是计划重新搭建gitea服务。

并且大部分文件资源通过保存在对象存储，因为服务器的水管太小了；再挂一个反向代理，因为gitea部署在内网。

## 安装
参考[Installation with Docker | Gitea Documentation](https://docs.gitea.com/installation/install-with-docker#installation)，通过docker安装gitea，compose如下：
```yaml
services:
  gitea:
    image: gitea/gitea
    restart: always
    volumes:
      - ./data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "222:22"
    environment:
      - USER_UID=1000
      - USER_GID=1000
```
此时就可以直接通过3000端口访问了。

## 内网穿透
由于我的情况是gitea部署在内网，所以我需要一个工具（[frp|github](https://github.com/fatedier/frp)）来进行内网穿透。通过docker分别在公网服务器和内网机器部署frp server和frpc client。

frps的compose文件和配置文件如下（[frp文档](https://gofrp.org/zh-cn/docs/features/common/configure/)）:
```yaml
# compose.yaml
services:
  frps:
    image: snowdreamtech/frps
    container_name: frps
    restart: always
    network_mode: "host"
    volumes:
      - ./frps.toml:/etc/frp/frps.toml
```

```toml
# frps.toml
bindPort = 7000
auth.token = "自定义你的token"
```

frpc的compose和配置文件如下：
 ```yaml
# compose.yaml
services:
  frp:
    # https://hub.docker.com/r/snowdreamtech/frpc
    image: snowdreamtech/frpc
    restart: always
    network_mode: host
    volumes:
      - ./frpc.toml:/etc/frp/frpc.toml
    container_name: wuyu-frp
 ```

```toml
# frpc.toml

user = "客户端的名称"
auth.token = "你的token，和服务端的设置要一样"
serverAddr = "服务器地址"
serverPort = 7000

# 我也忘了为什么要设置这个tls了
transport.tls.enable = false
transport.tls.disableCustomTLSFirstByte = false

[[proxies]]
name = "gitea"
type = "tcp"
localIP = "localhost"
localPort = 3000
remotePort = 3000
```

假设我的公网服务器是example.com，由于内网穿透我现在可以通过example.com:3000来访问服务器了。

## 反向代理
显然我并不想通过example.com:3000访问gitea，我想使用gitea.example.com访问(也可以使用example.com/gitea这样访问，但是gitea文档并不推荐)，这就需要反向代理，我们要做两件事：

第一步是要把gitea.example.com域名解析到服务器。这没什么好说的，推荐使用cloudflare解析，如果是其他厂商购买的域名，需要在原厂商更改DNS解析服务器为cloudflare的服务器。

第二步我们在公网服务器通过nginx监听80端口，可以把来自gitea.example.com的请求转发到3000端口，这样我们的80端口就可以被nginx根据不同的域名转发到不同的服务。

于是在公网服务器再创建一个nginx服务，compose和配置文件如下：
```yaml
# compose.yaml
services:
  nginx:
    image: nginx
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    network_mode: host
```

```conf
# 配置nginx如何处理并发 留空就是默认 但是没有这个字段会报错。
events{}

http {
    # 让nginx识别正确的content-type
    include /etc/nginx/mime.types;
    # http1.1开始默认使用长连接，多个HTTP连接可以共用同一个TCP，这是TCP的超时时间。
    keepalive_timeout 300;
    
	# 这个server会检测是否是通过gitea.example.com域名访问80端口的，
	# 如果是这个域名访问的，就把请求转发给localhost:3000
	# 也可以创建更多的server监听80端口，不同域名转发不同的服务
    server {
        # 监听80端口
        listen 80;
        
        # 接受什么域名的请求 下划线表示允许任何域名的请求
        # 改成你自己的
        server_name gitea.example.com;
        
        # location字段会用正则表达式匹配资源路径，代码块内对匹配到的请求做处理。
        # 这里是把请求转发到localhost:3000
        location / {
          client_max_body_size 10G;
          proxy_pass http://localhost:3000;
          proxy_set_header Connection $http_connection;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
	      }

    }
}
```

启动nginx之后就可以通过gitea.example.com访问gitea了。

## gitea配置

第一次启动gitea的配置没什么好说的，随便填，反正后面可以再改。

### 优化静态资源的流量
#### 思路一

参考[反向代理 | gitea文档](https://docs.gitea.com/zh-cn/administration/reverse-proxies#%E4%BD%BF%E7%94%A8-nginx-%E7%9B%B4%E6%8E%A5%E6%8F%90%E4%BE%9B%E9%9D%99%E6%80%81%E8%B5%84%E6%BA%90)， 我们可以把一些静态的css、图片、js存放到其他服务器上，这些资源就不会占用我们服务器的带宽了。

下载gitea源码（[gitea|github](https://github.com/go-gitea/gitea) 根目录真够乱的）然后执行`make frontend`，windows虽然可以安装make但是不太方便，可以用wsl或者其他linux。

如果实在一个linux都用不了，翻一下makefile你会发现`make frontend`其实就做了三件事，安装一下node依赖`npm instll`，设置环境变量`BROWSERSLIST_IGNORE_OLD_DATA=true`，然后执行`npx webpack`（node version >=18）。npx webpack这一步比较久，我花了一分半钟，这一步结束之后，public目录下就会多出css, fonts, js 和 licenses.txt，本来只有img的。这些就是我们的静态资源。

我们可以把这些静态资源放到其他静态部署服务中，来节省当前服务器的网络，但是实操下来发现即使是设置了跨域头，还是会有shared worker的安全报错。虽然你也可以改写响应的主机头，让静态服务和gitea服务保持同一个域名来避免跨域，但是在cloudflare上，主机头的改写是收费的。
#### 思路二
直接使用cloudflare的cdn，在cloudflare中打开你的域名，找到左边的规则，添加一个缓存规则。

把条件设置为 URL包含`https://gitea.example.com/assets/`，因为gitea的静态资源都是在/assets下访问的，这样设置assets就都会被缓存。

然后设置浏览器TTL和边缘TTL，设置大概一两天左右，这样浏览器和边缘服务器就会自动保存我们的静态资源，从而节省服务器带宽。


### 使用对象存储

参考: [配置说明 | Gitea Documentation](https://docs.gitea.com/zh-cn/administration/config-cheat-sheet#%E5%AD%98%E5%82%A8-storage)

我们可以把`附件、lfs、头像、仓库头像、仓库归档、软件包、操作日志、artifacts` 这些东西放到对象存储中。

AWS的S3是对象存储的事实标准，其他的对象存储基本都支持S3协议，也可以自建minio，minio是一个高性能对象存储服务。

```ini
[storage]  
STORAGE_TYPE = minio  
MINIO_ACCESS_KEY_ID =  
MINIO_SECRET_ACCESS_KEY =
MINIO_BUCKET = gitea
; 如果是自建的minio可以不写地区
; MINIO_LOCATION = us-east-1  
MINIO_USE_SSL = false  
MINIO_INSECURE_SKIP_VERIFY = false  
; 开启这个可以从客户端直接访问对象存储
SERVE_DIRECT = true
MINIO_BUCKET_LOOKUP_TYPE = auto
```

开启了对象存储之后，重启gitea，如果此时再上传一些东西就会保存到对象存储中了。

### 渲染mermaid、word或者jupyter

外部渲染可以在网页中渲染docx或者notebook，参考[外部渲染器 | Gitea Documentation](https://docs.gitea.com/zh-cn/administration/external-renderers#%E7%A4%BA%E4%BE%8Bhtml)

## CICD

**不推荐在自部署的Gitea上使用部署复杂的CICD**，原因如下:
1. 如果你的cicd涉及镜像仓库服务，对带宽是很大的挑战
2. gitea actions的actions生态都来自github，一方面gitea不能很好的兼容github actions，另一方面国内服务器访问github也不方便。


---
### 部署runner
gitea可以实现在某些时候自动触发一些任务，类似github action，任务是通过[gitea/act_runner](https://gitea.com/gitea/act_runner)执行的，而非gitea本身，Runner会在容器中执行任务。

但是act_runner的官方示例有一个问题，act_runner是直接在宿主机的docker上运行任务的。出于环境隔离的考虑，我们可以创建一个dind(docker in docker)容器，让runner使用容器内的docker来执行任务。

我尝试了在同一个compose中创建dind和act_runner，然后把dind的docker.sock映射出来给act_runner使用，但是套接字文件映射不出来，那就只能在同一个容器中安装docker和act_runner了,于是我采用了[vegardit/docker-gitea-act-runner](https://github.com/vegardit/docker-gitea-act-runner)的容器。

需要注意的是，这个容器的数据目录需要提前创建并设置777权限，否则连接gitea的时候他无法保存数据，就会导致一直重试，他在我的gitea上注册了200个无效的runner。

compose文件:
```yaml

```

### 清除runner
为了清除runner，我写了个爬虫，运行环境python版本3.13，DrissionPage版本4.0.5.6，代码如下
```python
from DrissionPage import ChromiumPage, ChromiumOptions
# 启动浏览器
page = ChromiumPage(
    ChromiumOptions()
    # 使用系统默认的用户数据文件夹
    # 这样可以保持你的登录状态
    .use_system_user_path()
)

for i in li:
    try:
    # 打开runners页面
        page.get(f"https://gitea.example.com/user/settings/actions/runners")
        # 选择runners页面的第二个runner的编辑按钮
        # 这个xpath可以在浏览器开发者工具对元素右键复制得到
        page.ele("xpath:/html/body/div/div/div/div[2]/div/div/div[2]/table/tbody/tr[2]/td[8]/a").click()
        # 此时进入了runner的设置页面 点击删除按钮
        button = page.s_ele("text:删除运行器")
        if not button:
            continue
        page.ele("text:删除运行器").click()
        # 确认删除
        page.ele("xpath:/html/body/div[2]/div/div[3]/button[2]").click()
        page.wait.url_change("gitea")
    except Exception as e:
        continue

```


### actions如何工作?
action的触发时机可以在yaml文件中配置，当一个action被触发时，action下的多个job就会被对应的runner执行(可以设置并发执行或者顺序执行)。

#### action如何触发
```yaml
name: action name
run-name: 本次运行的名称，运行触发者{{ gitea.actor }}
on: push

jobs:
  ...
```

每一次push触发就是像上面这样写，还可以手动触发或者pr触发等等，更多的写法参考参考 [Triggering a workflow - GitHub Docs](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/triggering-a-workflow)。gitea和github的action并没有在这方面有什么[区别](https://docs.gitea.com/zh-cn/usage/actions/comparison)

#### job与runner如何匹配？

runner的配置文件中有关于标签的配置(参考 [runner标签](https://docs.gitea.com/zh-cn/usage/actions/act-runner#%E6%A0%87%E7%AD%BE))，一个runner可以具备多个标签，一个标签形如`ubuntu-22.04:docker://node:16-bullseye`,格式形如`标签名:运行方式`，这里的标签名是`ubuntu-22.04`，当action设置为`runs-on: ubuntu-22.04`时，该runner就是可以运行该job的。

job大概是这个样子:
```yaml
jobs: 
  label_issue: //job名称
    runs-on: ubuntu-latest //用于匹配runner
```

#### runner如何运行job？

运行方式有两种，一种是主机运行，一种是容器运行。

主机运行，可以把标签写成`ubuntu-22.04:host`或者只有标签名:`ubuntu-22.04`，就会直接在runner所在的主机环境运行job。

也可以写`ubuntu-22.04:docker://node:16-bullseye`，那就是通过容器运行，双斜杠后面就是要运行job的容器环境。(gitea的act runner是[nektos/act](https://github.com/nektos/act)的分支，这个分支添加的三个特性之一就是，为每一个job创建新的容器，保证环境隔离。[其他的特性](https://docs.gitea.com/zh-cn/usage/actions/design#act))

#### job可以做什么操作?

首先我们可以为job设置一些运行条件和依赖关系，参考[Using jobs in a workflow - GitHub Docs](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/using-jobs-in-a-workflow)

job的具体行为是通过steps字段设置的，可以直接运行一些命令(默认通过bash执行，也可以[改成其他shell](https://docs.gitea.com/zh-cn/usage/actions/faq?_highlight=powershell#act-runner%E6%94%AF%E6%8C%81%E5%93%AA%E4%BA%9B%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F))，也可以通过`uses`字段运行一些可重用的action，[github actions market place](https://github.com/marketplace?type=actions) 有很多action，gitea的act runner也可以直接运行他们。

另外gitea有一个额外功能就是，虽然默认是从github下载，但你也可以通过[绝对链接改用自定义的action](https://github.com/marketplace?type=actions)，或者通过[Go编写action](https://blog.gitea.com/creating-go-actions/)。

下面这个示例用到的action就是签出仓库的
```yaml
jobs:
  Explore-Gitea-Actions:
    runs-on: ubuntu-latest
    steps:
      - run: echo "🎉 The job was automatically triggered by a ${{ gitea.event_name }} event."
      - run: echo "🐧 This job is now running on a ${{ runner.os }} server hosted by Gitea!"
      - run: echo "🔎 The name of your branch is ${{ gitea.ref }} and your repository is ${{ gitea.repository }}."
      - name: Check out repository code
        uses: actions/checkout@v4
```

这是一个部署到生产服务器的action示例:
```yaml
name: Deploy to Server
on:
  push:
    branches: [ main ]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Deploy via SSH
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USERNAME }}
        password: ${{ secrets.PASSWORD }}
        script: |
          cd /path/to/your/app
          git pull
          npm install
          npm restart
```

#### 如何使用变量

可以看到前面的job使用了一些秘钥和token，在gitea中，这些变量分用户、组织和仓库级别的，你可以自行创建。

创建配置变量后，它们会自动变成大写，并填充到 `vars` 上下文中。可以在工作流中使用类似 `${{ vars.VARIABLE_NAME }}` 这样的表达式来使用它们。

**TODO: 关于secrets**


## 踩坑
### docker容器上传失败
