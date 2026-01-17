# 小众软件

我乐意分享和总结我在 windows 和 linux 上的一些软件和常用设置。[小众软件](https://www.appinn.com/) 其实是一个很有意思的分享站/论坛，网友们在这里分享各种软件，这群人（包括我在内）最大的共同点就是喜欢研究如何更高效有趣地使用电脑。

---

## 操作系统和平台

分享我使用过的各种操作系统和平台。

我没有提到 nas 是因为我在 nas 上也用 linux；我没有提到 openwrt 是因为我没折腾过路由器；我没有提到 mac 是因为我没那么有钱。

### Termux

对于Termux，我曾经写过 [这篇文章](../1.旧文章/Termux.md)，引用其中一句话来介绍Termux：

> termux拥有非常活跃的生态，在“不root安卓系统的情况下使用Linux”这个场景下，termux几乎是唯一的权威，因此它坐拥一个相当大且活跃的社区。

但 termux 只是好玩，我并不觉得它特别有用，我曾经想用 termux 让我的平板具备一些生产力，最后以失败告终。

我看 macbook 搞得不错，生产力极大丰富，续航问题基本消灭，系统流畅，软件生态也受重视，如果加上价格亲民，macbook 就是我们理想中的 termux。

### WSL2

wsl 让 windows 成为最好的 linux 发行版（只是玩梗），而且 wsl 的 [官方文档](https://learn.microsoft.com/zh-cn/windows/wsl/) **非常好读**，不太需要额外的教程和其它文档。如果实在需要视频教程可以移步 [哔哩哔哩搜索页: wsl](https://search.bilibili.com/all?keyword=wsl)。

1. 但wsl并不是没有**缺点**，我经常使用某个永远有人唱反调的社交平台，如果想了解的话可以移步这个帖子 [你为什么放弃了wsl](https://www.zhihu.com/question/659267767) 。
2. wslg 可以让你在windows中直接显示wsl中的窗口程序，[这个up主](https://space.bilibili.com/441832003/lists/5917715) 发了不少关于wslg的内容，很有意思。
3. ***wsl 在 2025年5月开源了***，[htts://wsl.dev](htts://wsl.dev) 可以看到架构文档。
### Docker

用docker就像用手机一样自然，我不知道该怎么介绍docker。

我没看过 [docker入门文档](https://docs.docker.com/get-started/)，质量看起来还可以，我不确定。需要入门的新手也可以考虑移步 [哔哩哔哩搜索页：docker](https://search.bilibili.com/all?keyword=docker)。

服务器使用 docker 容易遇到网络问题，我在另一篇文章专门聊这个话题： [国内服务器网络问题](../4.国内服务器网络问题.md)，个人开发者电脑用 docker 直接开虚拟网卡就好了。

#### devcontainer

开发者个人环境还可以使用 [devcontainer](https://containers.dev/) ，这个工具的思路是在开发环境项目根目录映射到容器内，code server(vscode的服务端)、开发时的运行环境比如队列和数据库，或者一些只有开发期会用到的工具，都在容器环境运行。

使用 devcontainer 两个非常大的好处，一是环境隔离，避免这个项目的某些依赖在宿主机中有冲突，二是可以通过配置文件定义开发环境，即使时过境迁沧海桑田，你也可以快速获得一整套一模一样的开发环境。

我曾经一度为我所有新建的仓库都使用 devcontainer，但其实没必要，如果是所有的依赖都在 node_modules 或者 venv 里的简单项目，大多数时候不需要 devcontainer 也可以隔离依赖。

devcontainers 最有趣的地方是 features ，它的思路是声明式的定义需要的工具，它就会为我执行某些社区维护的脚本来自动安装，像这样写devcontainer.json：

```json
"features": {
    "ghcr.io/rocker-org/devcontainer-features/pandoc:latest": {
        "version": "2.17"
    }
}
```

其实声明式定义环境在 k8s 和 gitops 中早就烂大街了，只是我以前不懂其中的珍贵。直到发现 devcontainer features 能够让我在 dockerfile 中少花很多时间，我才知道声明式定义环境原来这么方便。

但是 features 有一个严重的缺点，就是每次创建容器的时候都会重新执行features脚本，而不是使用构建缓存，大概是 features 就没有缓存，我没有仔细了解过devcontainer的实现。

我大约在一年前发现了features的缓存问题，至今一直不知道如何解决，刚刚随手翻查了 devcontainer 的文档，注意到一篇讲 [预构建](https://containers.dev/guide/prebuild) 的文档，我注意到 预构建 + features 不仅可以彻底解决features缓存问题，还能解决 docker 构建不能声明式定义的问题，通过devcontainer cli 可以连同 features 一起构建镜像，这样以后就只需要写 features（实际上就是写shell脚本），不需要再折腾 dockerfile 了。


### 关于虚拟化的插叙

我在弄清楚 wsl2 和 docker 的工作方式的过程中学习了很多虚拟化知识，比如类型一和类型二的 hypervisor，比如 namespace 和 cgroup，


### 多系统

“如何安装双系统” 是一个高质量教程云集的入门话题，不懂装系统的新手可以移步：[哔哩哔哩搜索页：如何安装双系统](https://search.bilibili.com/all?keyword=%E5%A6%82%E4%BD%95%E5%AE%89%E8%A3%85%E5%8F%8C%E7%B3%BB%E7%BB%9F) 

如果要简单的烧录启动U盘，我用 [rufus](https://rufus.ie/zh/)（windows可用），这是一个轻量的、只做好烧录这一件事的简单应用。另外还推荐 [ventory](https://www.ventoy.net/cn/index.html)（多平台可用），引用ventory官网的话来介绍它：

> 简单来说，Ventoy是一个制作可启动U盘的开源工具。  
> 
> 有了Ventoy你就无需反复地格式化U盘，你只需要把 ISO/WIM/IMG/VHD(x)/EFI 等类型的文件直接拷贝到U盘里面就可以启动了，无需其他操作。  
> 
> 你可以一次性拷贝很多个不同类型的镜像文件，Ventoy 会在启动时显示一个菜单来供你进行选择 (参见 [截图](https://www.ventoy.net/cn/screenshot.html))。  
> 
> 你还可以在 Ventoy 的界面中直接浏览并启动本地硬盘中的 ISO/WIM/IMG/VHD(x)/EFI 等类型的文件。  
> 
> Ventoy 安装之后，同一个U盘可以同时支持 x86 Legacy BIOS、IA32 UEFI、x86_64 UEFI、ARM64 UEFI 和 MIPS64EL UEFI 模式，同时还不影响U盘的日常使用。  
> 
> Ventoy 支持大部分常见类型的操作系统 （Windows/WinPE/Linux/ChromeOS/Unix/VMware/Xen ...）

#### 管理系统引导

这是一个有点登堂入室的话题，“引导”就是指按下开机键之后，主板上的固件程序查找可用的操作系统并启动的过程，英文叫做Boot，其实就是启动的意思，“引导”是专门描述这个过程的术语。

在管理系统引导之前至少要先学习引导是怎么发生的，我不知道大家是不是都这样，我觉得学习系统引导是一个比较艰苦的过程，虽然我会组装电脑和装系统，也能写代码，但对主板一无所知。

网络上关于引导过程的描述真的有很多谬误，甚至包括我以前在知乎写的文章现在看也有很多信息是错的，就连2014年的文章中也都说网上大部分信息都是错误的（我没有经历过那个时候，我觉得那个时候的互联网可能会更容易出现高质量信息）。

如果你想了解正确的知识，我有一份推荐的阅读列表，原谅我推荐的都是比较老的文章，因为UEFI真的早就普及了，虽然他们的文章也算是二手知识，但至少不是十八手知识，而且我建议你一遍看他们的文章，一边浏览你的ESP分区来验证（看看可以，不建议乱动）。

1. [《UEFI 启动是如何工作的? 》](https://www.happyassassin.net/posts/2014/01/25/uefi-boot-how-does-that-actually-work-then/) 作者 Adam Williamson，写于2014年，文章用语介于通俗和专业之间，介绍了旧的BIOS的工作方式，UEFI是如何从问题出发来进行设计的，以及Boot Manager和Secure Boot。这篇文章写的非常好，我担心原链接丢失所以把这篇文章的原网页制作成了google机翻的 [中英双语版PDF](assets/小众软件index/UEFI_boot_how_does_that_actually_work_Adam_Williamson.pdf) ，看起来内容很多，但实际上这个网页有四分之三都是评论。
2. [《管理UEFI Boot Loader》](https://www.rodsbooks.com/efi-bootloaders/index.html) 作者Rod Smith，写于2011。Rod Smith在他的网站中写了很多关于系统引导的文章，很权威。

如果真的很想追求一手知识，那就去读[《UEFI Specification 2.11》](https://uefi.org/specs/UEFI/2.11/) ，一般可以在 [UEFI论坛](https://uefi.org/) 找到最新版。


