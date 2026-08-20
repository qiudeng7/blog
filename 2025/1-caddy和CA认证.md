# caddy和CA认证

如何把我的链接从HTTP变成HTTPS？

HTTPS是经过TLS(Transport Layer Security)协议加密的HTTP。TLS是SSL(Secure Sockets Layer)协议的继任者，工作在传输层。

HTTPS相比HTTP解决两个问题：
1. 如何保证客户端访问的目标网站是可信的 —— CA认证
	1. CA认证是指目标网站需要通过证书颁发机构Certificate Authority颁发证书，才可以证明自己的网站是可信的，客户端才会把数据发送给该服务端。
2. 如何保证数据在传输过程中不会被窃取 —— 非对称加密
	1. 非对称加密： 即公钥加密后的内容只有私钥可解密，私钥加密的内容只有公钥可以解密。服务器持有私钥，并对外公开公钥，客户端发送的内容均使用公钥加密。
	2. 客户端发送HTTP请求之前先获取公钥和证书，验证证书无误之后生成一个秘钥R，用公钥加密R值发送给服务器，服务器使用私钥解密R之后，双方通过R进行对称加密传递信息，即可保证数据在传递过程中的安全。

非对称加密是单纯的技术问题，有很多软件都内置了解决方案。主要难点是如何进行CA认证，最常见的方法是letsencrypt。

## letsentrypt

[letsencrypt](https://letsencrypt.org/getting-started/)是一个免费、自动化且开放的证书颁发机构，由非营利性的互联网安全研究小组（ISRG）运营。其致力于为网站提供免费的 TLS 证书，极大地推动了 HTTPS 加密在互联网的普及，降低了网站部署加密的门槛。由于证书过一段时间就会过期，letsencrpt支持使用ACME协议自动化获取和更新证书。

如何使用ACME协议？ACME要求客户端完成“ACME challenge”（中文翻译为质询），要求使用者做一些指定操作，来证明你是域名的所有者，才会给你颁发证书。

challenge有很多，常见的两种是HTTP challenge和DNS challenge。
1. HTTP challenge要求域名解析的那个服务器，在80端口下指定路径设置返回指定内容，则可以证明你是域名所有者。
2. DNS challenge要求DNS解析添加指定的TXT记录，则可以证明你是域名所有者。

ACME客户端会帮助我们自动完成challenge，最常见的ACME客户端是[certbot](https://certbot.eff.org/)。还有一个比较现代的方案是caddy，它既是ACME客户端，也是反向代理软件。下面对caddy和certbot分别进行介绍。

> 注意对于内网穿透的情况，也就是自己没有公网服务器的情况下，HTTP challenge是无法完成的，只能使用DNS challenge，为了让ACME客户端能够设置域名解析，通常要让ACME客户端使用服务商的token和API自动解析域名。

### 关于cloudflare证书
网上有cloudflare能拿到15年证书的说法，确实能拿到，但是cloudflare的证书分好几种，可以被客户端信任的证书是需要付费的，能免费拿到的证书是指你的源服务器可以被cloudflare信任，不会被任意客户端信任。
## certbot

安装：
1. certbot有很多种安装方式，有python虚拟环境安装的，也有各种软件包管理器安装的，[certbot官网](https://certbot.eff.org/instructions?ws=nginx&os=pip)提供了不同环境下的安装方式。
2. certbot的[文档](https://eff-certbot.readthedocs.io/en/latest/install.html#)和官网是两个网站，文档中介绍了snap、docker、pip等各种安装方式。

certbot是一个命令行工具，总的来说可以执行两种类型任务：
1. `certbot certonly`，单纯的获取证书文件。
2.  另一种是不仅获取证书文件，还自动安装到web服务器比如nginx和apache，他会修改这些web服务器的配置。

具体使用方式参考 [getting-certificates-and-choosing-plugins](https://eff-certbot.readthedocs.io/en/latest/using.html#getting-certificates-and-choosing-plugins)，笔者没有在certbot上有很多实践经验。

获取到证书文件之后，nginx中可以这样使用：
```nginx
server {
    listen 443 ssl;
    server_name example.com;  # 替换为你的域名
    # 配置 SSL 证书和私钥的路径
    ssl_certificate /etc/nginx/ssl/example.com.crt;  # 替换为你的证书文件路径
    ssl_certificate_key /etc/nginx/ssl/example.com.key;  # 替换为你的私钥文件路径

    # SSL 相关配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        root /var/www/html;  # 替换为你的网站根目录
        index index.html index.htm;
    }
}
```

## caddy的基本使用

caddy是一个go编写的、开源的web服务器，常常用来对标nginx，caddy的亮点是：
1. 内置的自动HTTPS认证
2. 简洁的配置文件
3. 现代化的设计

docker compose 运行caddy：
```yaml
services:
  caddy:
    image: caddy
    network_mode: host
    volumes:
	  - ./data:/data
      - ./conf:/etc/caddy
```

> caddy会自动重载他的配置文件`/etc/caddy/Caddyfile`，但是不要直接用docker挂载它，因为一些编辑器可能会更改文件的inode（比如vim），可能会导致caddy加载配置文件失败。这段说明参考自[caddy | Docker Hub](https://hub.docker.com/_/caddy)

Caddyfile的语法如下
![caddyfile syntax](https://caddyserver.com/old/resources/images/caddyfile-visual.png)

Caddyfile包括这些概念（参考自[Caddyfile concepts](https://caddyserver.com/docs/caddyfile/concepts)）
1. [全局配置](https://caddyserver.com/docs/caddyfile/concepts#global-options)，这是可选的，如果有的话，他需要出现在最上面。
2. [片段](https://caddyserver.com/docs/caddyfile/concepts#snippets)，也是可选的，如果有的话，需要出现在全局配置和站点配置之间。
3.  站点配置，是最主要的配置，决定了如何路由，这个部分包含左上角的address和代码块内的directive。
	1. [address](https://caddyserver.com/docs/caddyfile/concepts#addresses)负责匹配域名。
	2. [directive](https://caddyserver.com/docs/caddyfile/concepts#directives)是具体的操作，通常接受一些参数，参数包括matcher，matcher用于匹配路径，有时候还包括子命令，比如上面的reverse_proxy的内部代码块。
4. [环境变量](https://caddyserver.com/docs/caddyfile/concepts#environment-variables)
	1. Caddyfile在解析时会把这种语法替换为环境变量值 `{$SERVERS}`
	2. `{$DOMAIN:localhost}`，这种写法用来指定默认值。

语法方面有两点要注意：
1. 代码块
	1. `{`必须位于行尾，并且前置一个空格
	2. `}`必须单独一行
	3. 缩进必须工整
2. 如果整个文件都只有一个站点配置，则括号和缩进可以省略。

所以这有一个最简单的示例：
```caddyfile
localhost

reverse_proxy /api/* localhost:9001
```

这表示所有接收到的对`https://localhost/api`的访问都会被转发到9001端口。

其他基本功能的介绍：
1. 反向代理：[Reverse proxy quick-start — Caddy Documentation](https://caddyserver.com/docs/quick-starts/reverse-proxy)
2. 静态文件服务器：[Static files quick-start — Caddy Documentation](https://caddyserver.com/docs/quick-starts/static-files)
3. 自动HTTPS：[Automatic HTTPS — Caddy Documentation](https://caddyserver.com/docs/automatic-https)

关于ACME challenge，caddy会自动寻找并尝试所有开启了的challenge, caddy默认开启了以下两个：
1. HTTP challenge：需要在指定解析的服务器的80端口下设置特定的资源，CA机构查询到预期的文件之后才会发布证书。
2. TLS-ALPN challenge：类似上面，但是通过TLS协议加密过的协议，请求443端口的特定资源。

这要求CA机构能够直接访问目标机器的80端口或者443端口——也就是说必须是公网机器，那么caddy无需任何额外配置就可以直接获取到证书，哪怕是下面这样简单的配置：
```Caddyfile
test.example.com

reverse_proxy /api/* localhost:9001
```

对于更多的challenge请参考[caddy文档的acme challenge部分](https://caddyserver.com/docs/automatic-https#acme-challenges)，本文主要介绍的是HTTPS challenge，这需要添加dns插件。

## xcaddy构建dns插件

添加dns插件的方法参考[这个帖子](https://caddy.community/t/how-to-use-dns-provider-modules-in-caddy-2/8148)(官方文档亦推荐该帖，因为DNS challenge需要对接具体的服务商，这项工作由社区开展，于是文档在论坛的帖子当中。)

添加插件需要重新构建caddy，这需要用到一个叫做[`xcaddy`](https://github.com/caddyserver/xcaddy)的工具。使用xcaddy需要go运行环境，建议在docker中构建。windows系统的构建方式大体相同，后文会说明要做出哪些调整。

[caddy dockerhub](https://hub.docker.com/_/caddy)提供了普通的caddy镜像和builder镜像，后者包含了xcaddy。同时也都提供了windows版本。你可以像这样在dockerfile中构建caddy（参考 [caddy的dockerhub首页](https://hub.docker.com/_/caddy)）
```Dockerfile
FROM caddy:<version>-builder AS builder

RUN xcaddy build \
    --with github.com/caddyserver/nginx-adapter \
    --with github.com/hairyhenderson/caddy-teapot-module@v0.0.3-0

FROM caddy:<version>

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

xcaddy命令行的with参数可以设置插件的仓库，这些插件会被一起构建。也可以替换caddy core，参考 [xcaddy自定义构建 github](https://github.com/caddyserver/xcaddy?tab=readme-ov-file#custom-builds)

对于DNS插件，在[caddy-dns](https://github.com/caddy-dns)（一个github组织）搜索你要的dns插件，然后添加到上面的with参数中。构建得到的二进制文件就可以使用DNS challenge了。

需要在caddyfile中配置DNS challenge才可以使用，因为caddy会调整你的域名解析来完成challenge，这需要我们提供token。dns challenge的配置在caddyfile的global options中：
```Caddyfile
{ 
	acme_dns <provider> ...
}
```

dns插件提供的provider各不相同，需要参考各自插件内的说明。对于cloudflare，类似这样配置即可：
```Caddyfile
{
    acme_dns cloudflare 1484053787dJQB8vP1q0yc5ZEBnH6JGS4d3mBmvIeMrnnxFi3WtJdF
}
```

### cloudflare获取token

cloudflare获取token也有讲究（cloudflare的页面交互设计实在是next level），这部分内容参考  [How to get Cloudflare API token env variable?](https://caddy.community/t/how-to-get-cloudflare-api-token-env-variable/8307)

打开cloudflare > profile > API tokens，cloudflare有两种秘钥，token和key，token可以限制范围，key可以全局使用，但是用于api challenge的基本都是要用token，key会失效。

点击创建token，可以使用`Edit zone DNS`模板，他会自动添加一个具有编辑DNS权限的token。

这个token权限分三类，主账户相关(Account)、功能相关(Zone)和子用户相关(User)，DNS在Zone下面，有`DNS`和`DNS Settings`两种，选择`DNS`。然后权限设置为`Edit`，创建这个token即可。

对于其他类型的服务商，大致逻辑也是如此，只需要提供编辑DNS解析的权限即可。

### windows构建
windows构建caddy同样需要使用xcaddy，caddy的dockerhub页面提供了windows版本，但是笔者不太熟悉docker中的windows，故直接在windows宿主机环境直接构建。

需要获取xcaddy和go的windows版本，xcaddy有现成的windows发布版： [xcaddy release](https://github.com/caddyserver/xcaddy/releases)，go也有：[golang release](https://go.dev/dl/) （如果不想污染环境，下载zip就行）

需要把go的可执行文件临时添加到到环境变量中，方便xcaddy使用，下面是powershell命令：
```powershell
$env:PATH += ";C:\Users\qiudeng\Desktop\moyu_caddy\go\bin"
```

在命令行执行`go version`检查go是否可用，如果go可用，则执行xcaddy的构建命令，即可得到具有插件功能的`caddy.exe`
```powershell
./xcaddy.exe build --with 插件名
```

## xcaddy代理
构建时出现类似如下日志
```text
2025/07/07 09:02:42 [INFO] absolute output file path: C:\Users\qiudeng\Desktop\moyu_caddy\caddy.exe
2025/07/07 09:02:42 [INFO] Temporary folder: C:\Users\qiudeng\AppData\Local\Temp\buildenv_2025-07-07-0902.1281457826
2025/07/07 09:02:42 [INFO] Writing main module: C:\Users\qiudeng\AppData\Local\Temp\buildenv_2025-07-07-0902.1281457826\main.go
package main

import (
        caddycmd "github.com/caddyserver/caddy/v2/cmd"

        // plug in Caddy modules here
        _ "github.com/caddyserver/caddy/v2/modules/standard"
        _ "github.com/caddy-dns/alidns"
)

func main() {
        caddycmd.Main()
}
2025/07/07 09:02:42 [INFO] Initializing Go module
2025/07/07 09:02:42 [INFO] exec (timeout=0s): C:\Users\qiudeng\Desktop\moyu_caddy\go\bin\go.exe mod init caddy
go: creating new go.mod: module caddy
go: to add module requirements and sums:
        go mod tidy
2025/07/07 09:02:42 [INFO] Pinning versions
2025/07/07 09:02:42 [INFO] exec (timeout=0s): C:\Users\qiudeng\Desktop\moyu_caddy\go\bin\go.exe get -v github.com/caddyserver/caddy/v2 
2025/07/07 09:02:52 [INFO] SIGINT: Shutting down
2025/07/07 09:02:52 [FATAL] exit status 0xc000013a
```

关键信息在`C:\Users\qiudeng\Desktop\moyu_caddy\go\bin\go.exe get -v github.com/caddyserver/caddy/v2`

xcaddy尝试执行go get拉取github镜像，然后超时，并且设置系统代理无效。有两种方法，一个方法是提前下载caddy core并且替换，但是没有成功，方法参考xcaddy的readme

> You can even replace Caddy core using the --with flag:
> $ xcaddy build \ --with github.com/caddyserver/caddy/v2=../../my-caddy-fork
> $ xcaddy build \ --with github.com/caddyserver/caddy/v2=github.com/my-user/caddy/v2@some-branch
> This allows you to hack on Caddy core (and optionally plug in extra modules at the same > time!) with relative ease.

另一种方法是，注意到所执行的命令为`go.exe get`，显然是一种标准库或者通用模块，简单查询之后发现即使是windows系统也可以直接设置HTTP_PROXY来为它配置代理。当然直接设置TUN也是可以的。
## 其他caddy插件
### 转发TCP
参考 [使用Caddy转发内网的Minecraft服务器小结 - 冻葱Tewi](https://blog.dctewi.com/2024/06/summary-caddy-l4-minecraft/)

要用到这个插件 [mholt/caddy-l4: Layer 4 (TCP/UDP) app for Caddy](https://github.com/mholt/caddy-l4)，构建命令
```bash
xcaddy build --with github.com/mholt/caddy-l4
```
