# Termux

原理上来说，[termux](https://wiki.termux.com/wiki/Main_Page)是一个“安卓终端模拟器”，他通过 [execve(2)](https://www.man7.org/linux/man-pages/man2/execve.2.html)（一个system call）来重定向标准输入输出和错误流，从而实现了终端功能。termux上可以使用一些linux上常见的软件，因为termux为他们专门提供了一些安卓相关的补丁，并且有特供的软件仓库。

但是termux的文件系统并不兼容标准的linux文件系统，用户的家目录位于
```
/data/data/com.termux/files/home
```
该目录可以通过`$HOME`环境变量访问。

而`/bin`, `/etc/`, `/var`等目录位于
```
/data/data/com.termux/files/usr
```
该目录称作prefix，并可以通过`$PREFIX`环境变量访问。

prefix目录不能随意修改，因为它已经被硬编码到各个二进制包中。

termux的更多与常见linux的区别参考[Differences from Linux - Termux Wiki](https://wiki.termux.com/wiki/Differences_from_Linux)

## 生态
termux拥有非常活跃的生态，在“不root安卓系统的情况下使用Linux”这个场景下，termux几乎是唯一的权威，因此它坐拥一个相当大且活跃的社区。

1. tmoe，这是一个纯shell开发的项目，提供了一些TUI界面帮助用户配置termux，包括换源、换字体、安装子系统等等。
2. proot-distro，也是纯shell的项目，帮助用户通过proot快速安装子系统。
3. aid learning，基于termux搭建的项目，自己开发了一套可视化界面，主要受众是AI学习者。
4. udroid，Ubuntu on android，旨在帮助用户快速在无root安卓上运行预装桌面环境的ubuntu，桌面环境包括xfce和gnome。
5. termux版本的qemu

待补充...

## 安装termux

安装termux的最好的方式应该是github和fdroid。google play中的已经是过时的版本。

清华镜像源中可以下载到fdroid安装包，并且设置镜像源。

## 终端美化
你可以在fdroid安装termux: style，然后可以简单优化一些背景色、字体。

关于nerdfont，可以使用这个仓库：[arnavgr/termux-nf: A better way to install NerdFonts on Termux](https://github.com/arnavgr/termux-nf)

## 用户软件仓库
待补充 termux-user-repo

[termux-user-repository/tur: A place for all types of Termux packages.](https://github.com/termux-user-repository/tur)

里面似乎包含i3、vscode、Chromium等软件，粗略看了一眼没找到package列表。


## 模拟linux

我们可以在termux内创建一个容器环境安装Linux。最首要的问题就是文件系统的虚拟化，通常使用chroot实现，chroot可以让某些进程认为指定的目录就是根目录，从而我们可以在这个目录中安装一个新的系统。但是chroot是一个需要root权限的system call，我们要求的条件是不对手机进行root。

于是就有了chroot的用户空间版本：[proot](https://github.com/proot-me/proot)(pseudo root)。它的原理是劫持并修改进程的所有systemcall，从而达到在用户空间实现虚拟化的效果，但是也因此牺牲了性能。注意proot并不是为termux而设计的，termux社区自行维护了一个[termux版本的proot](https://wiki.termux.com/wiki/PRoot)。

进一步地，termux社区还提供了一个基于proot直接安装发行版的工具[proot-distro](https://github.com/termux/proot-distro)，另外如果你的发行版需要另外的架构，proot-distro会使用termux版本的qemu安装它。另外也有其他类似的生态位的工具比如udroid也是基于proot和qemu，帮助用户一键安装可用的发行版。

下面这一段来自[proot官方文档](https://proot-me.github.io/)。
> When the guest Linux distribution is made for a CPU architecture incompatible with the host one, PRoot uses the CPU emulator QEMU user-mode to execute transparently guest programs. It's a convenient way to develop, to build, and to validate any guest Linux packages seamlessly on users' computer, just as if they were in a _native_ guest environment. That way all of the cross-compilation issues are avoided.

唯一的问题是，这些工具支持的发行版都是有限的([proot-distro bundled distributions](https://github.com/termux/proot-distro?tab=readme-ov-file#bundled-distributions) 尽管proot支持一定程度的使用自定义的发行版)，我们也许有时候需要直面proot，而非让这些shell脚本代劳(proot-distro和udroid都是shell编写的)。

### proot-distro和proot原理

proot利用Linux的ptrace机制，像调试器一样“附着”到子进程上，拦截所有系统调用。当进程调用如`open("/etc/passwd")`时，proot会把路径改写成你rootfs下的路径，比如

```
/data/data/com.termux/files/usr/var/lib/proot-distro/installed-rootfs/ubuntu/etc/passwd
```

proot的虚拟化是“用户态的系统调用重定向和模拟”，不需要 root，不需要内核支持。proot可以直接运行纯用户态的程序，但是对于system call的请求，proot会重定向路径、伪装环境信息，而对于需要root权限的systemcall，proot只能模拟和伪装，或者直接报错。

而proot-distro只是一个proot的简单封装，观察proot-distro源码发现，`proot-distro login`命令最终是拼接了一堆参数得到了一个proot命令并执行。

所以可以把proot-distro简单理解为一个proot指令的构造器。在login子命令的最终，我们会得到这样一个指令：
```bash
exec proot --rootfs=/data/data/com.termux/files/usr/var/lib/proot-distro/installed-rootfs/ubuntu \
		  --change-id=0:0 \
		  --cwd=/root \
		  --bind=/dev ... \
		  /usr/bin/env -i HOME=/root USER=root ... /bin/bash -l
```

那么`proot-distro install`又是如何工作的呢？这是代码中的注释:

> 1. Process arguments supplied to 'install' command.
> 2. Download the tarball of rootfs for requested distribution unless found in cache.
> 3. Verify SHA-256 checksum of the rootfs tarball.
> 4. Extract the rootfs under PRoot with link2symlink extension enabled.
> 5. Perform post-installation actions on distribution to make it ready.
> 6. Source the distribution configuration plug-in.
> 7. Ensure that requested distribution is supported and is not installed.

## termux和nixos

1. github用户leap0x7b和一个已注销的用户对这个问题有过讨论：
	1. 这个地方有很多失败的讨论，没有成功案例：[NixOS · Issue #215 · termux/proot-distro](https://github.com/termux/proot-distro/issues/215)
	2. [这个PR](https://github.com/termux/proot-distro/pull/223#issuecomment-1158769071)的讨论似乎把问题原因指向nixos不支持FHS
2. 这个[revert](https://github.com/termux/proot-distro/commit/32bb967f4dd7e73aeb82ed5bc97ad5bbc46d0a80)指出失败原因：
> 1. Non-deterministic rootfs layout. Although workarounds possible...
> 2. Certain things depend on services like nix-daemon. I was unable to get nix-daemon working properly.
> 3. Built-in help for certain tools unavailable out-of-box.
> 4. Possibly much more.
> 
> In general NixOS doesn't seem to be well suitable for chroot environment.


但是还是可以在termux中使用nix，只是不能用nixos而已。
1. 有一个现成的封装，[nix-on-droid](https://github.com/nix-community/nix-on-droid)
2. nix官方wiki中提到了[proot中安装nix的方案](https://nixos.wiki/wiki/Nix_Installation_Guide#PRoot)

## 真正的linux环境

我们会发现任何基于proot的项目都不是真正的linux，只是模拟而已。我们当然希望有一个真正的完全的linux环境。

那么为什么不直接用qemu安装完整的linux？因为运行起来 `VERY slowly`，参考：
[Would it be possible to install a native ARM app like QEMU? : r/termux](https://www.reddit.com/r/termux/comments/15ksp0x/would_it_be_possible_to_install_a_native_arm_app/)

也可以考虑使用远程控制，这个主要要克服的就是网络问题，而且也未必要使用termux了。我尝试了使用rustdesk，使用腾讯云的竞价服务器，临时租了一个小时的百兆服务器，价格是一小时二十几块钱，然后自己部署rustdesk，我的评价是远程控制的效果依然差强人意。


## 受限的桌面环境+完全的开发环境

实际上我们其实可以接收本地有一个不完全的linux环境，只要能够通过vscode或者cursor连接到完全的远程环境即可，在不需要远程桌面的情况下，延迟问题能够大幅优化。

1. 底层来说桌面环境有两大协议，x11和wayland。termux中都有相关的工具：[termux-x11](https://github.com/termux/termux-x11)，[wlroots-termux](https://github.com/xMeM/wlroots-termux)。

2. 一个不用proot直接在termux本地安装i3的例子：[How to install 🔥i3 WM🔥on Termux (X11) native DESKTOP on ANDROID - [No Root] - Linux on Android - YouTube](https://www.youtube.com/watch?v=Uqf9zk6W7S8)

3. [termux官方文档](https://wiki.termux.com/wiki/Graphical_Environment)提到的UI界面包括两个窗口管理器Fluxbox、Openbox和三个桌面环境XFCE、LXQt、mate

4. 应用层方面，udroid简化了ubuntu的桌面环境配置，可选xfce、gnome和mate。dextop做的工作同样也是基于proot安装XFCE。

## TODO
现在发现termux原生就有很多东西就可以用，他们好像只是打了一些patch就直接添加到termux的软件仓库里了，因为termux似乎提供了一个基于安卓的构建流程。所以确实没有完整linux的必要，只需要在termux有一个本地工作流即可，实际执行可以放在服务器上。

1. i3方案的博主的其他视频
2. 阅读i3方案的源代码 尝试hyperland
3. termux:x11配合其他de是否会更好用？
4. udroid是否会流畅？
5. [自己开发一个termux包](https://github.com/termux/termux-packages/wiki) 来支持更多的桌面环境比如Hyperland(有必要吗？)
6. 录一些视频介绍termux生态

## termux总结
termux只是工具，我的真实需求是希望能够在平板上也能够获得可用的开发环境。这个开发环境除了编辑器，还包括AI支持，浏览器，甚至包括画图等。

我有几种方式满足需求：
1. 本地运行完整的开发环境，这是体验最佳的路径，但是需要承担root和刷机的风险。
2. 远程桌面，同样会获得完整的开发环境，但主要有两个问题：
	1. 延迟问题
	2. 安卓系统的键盘快捷键会和远程控制的快捷键冲突。
3. 基于termux，硬件虚拟Linux桌面环境
	1. 运行非常慢，完全不可用。
4. 基于termux，软件虚拟Linux桌面环境
	1. root功能受限，比如termux就不可用。
5. 基于termux，使用vscode远程控制，或者ssh远程控制，或者在线IDE，问题有三：
	1. 会失去cursor的AI支持
	2. 平板自带的窗口切换并不灵活，浏览器和编辑器切换不顺手。
	3. 其他在开发环境使用的GUI工具可能都要转移到安卓浏览器上，比如画图工具等，并且操作逻辑不同。

而且就算上述问题全部解决，平板+键盘还不如直接用笔记本，所以不如花钱买一个长续航的笔记本，远程连接工作站。