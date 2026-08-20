# 国内组网服务商

下面这张图是我总结的组网和穿透策略。最近由于需求和场景发生变化，我又开始折腾新的方案。

![组网与穿透策略](assets/2-国内组网服务商/networking-strategy.svg)

新的场景是：我既有自己的公网 IP 服务器，也有许多内网机器。

一开始我自己搭建了 FRP，但是它的心智模型很麻烦：需要考虑证书和协议，每添加一个服务都要修改配置。后来我开始使用 NetBird 组网，解决了这些问题。暴露服务时，只需要直接在 Caddy 或 Nginx 中指向组网后的局域网 IP 即可。

但如果考虑给小公司使用，单台服务器的网络质量和稳定性都难以保障，因此接入组网服务商、使用低成本套餐是更好的选择。

国内提供组网服务的厂商似乎不多，我找到的包括：

1. [CloudNet](https://cloudnet.world/)
2. [EasyTier Pro](https://www.easytier.net/index.html)
3. [量子互联](https://www.uulap.com/nsvpc)

比价之后，我最终选择了 CloudNet：

1. 经过实际使用，CloudNet 的组网带宽和稳定性都很好。
2. CloudNet 的穿透服务很不稳定。好在我有自己的服务器，可以自行暴露服务。
3. CloudNet 兼容 Tailscale 客户端。

## 使用 Tailscale 客户端访问 CloudNet

可以使用 Tailscale 官方客户端或兼容的第三方客户端接入 CloudNet 服务端。CloudNet 自己的客户端做得比较一般，Windows、Linux 和 macOS 版本都是如此，Android 和 iOS 更是没有客户端。

安装 Tailscale 客户端后，可以按照下面的方法连接 CloudNet。

> 如果使用过 Headscale，只需要知道一点：CloudNet 的控制面服务器地址是 `https://cp.cloudnet.world`。

### 安装 Tailscale 客户端

具体安装方法参考 [Download · Tailscale](https://tailscale.com/download)。

Linux 可以执行：

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### 获取 Auth Key

进入 CloudNet 后台，在设置页面创建一个子网，再为这个子网创建关联密钥并记下它。

### 使用命令行连接

Windows 安装 Tailscale 后，可以直接在 PowerShell 中使用 `tailscale` 命令。仅通过图形界面无法让 Windows 版 Tailscale 连接非官方控制面。

交互式登录可以执行下面的命令。它会返回一个 CloudNet 官网链接，在已经登录 CloudNet 账号的浏览器中打开即可完成连接。

```powershell
tailscale login --login-server=https://cp.cloudnet.world
```

非交互式连接可以执行：

```bash
tailscale up \
  --login-server=https://cp.cloudnet.world \
  --auth-key=你的密钥 \
  --accept-dns=false \
  --accept-routes=false
```

### Android

1. 打开 Tailscale，不要按照默认流程登录 Tailscale 账号。
2. 进入右上角设置 → `Accounts`（账户）→ 右上角 `⋮`。
3. 选择 `Use an alternate server`（使用备用服务器）。
4. 填入 `https://cp.cloudnet.world`。应用会自动跳转到浏览器，此时直接返回即可。
5. 再次打开同一个 `⋮` 菜单，选择 `Use an auth key`，填入 CloudNet 控制台生成的密钥。
6. 回到主界面登录或连接，并允许 Android 请求的 VPN 权限。

> 作者没有 iOS 设备，预计操作方式与 Android 大致相同。
