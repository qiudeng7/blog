# 再次尝试 Linux 桌面

2026 年 7 月，我计划再次尝试 Linux 桌面。操作之前我想了很多，预期的最终状态是这样的：

1. 给我现在用作 Home Server 的主机加一张 A 卡，并在这台主机上安装 Linux。
2. 当前的 3070 Ti 笔记本删掉其他系统，只保留 Windows 11。
3. 主机上也只保留一个 Linux 系统，发行版暂未决定。
4. Linux 使用窗口管理器（WM）而不是桌面环境（DE），使用触控板而不是鼠标。
5. 在 Linux 中将很多软件改为使用 TUI，例如 VS Code 换成 Neovim，Freelens 换成 K9s。
6. 闲鱼上淘来的 ThinkPad 只花了四百三十元，配置是 16 GB 内存和 256 GB 存储，还自带电源和外设，很适合做 Home Server。

下面记录这次尝试的几个主要阶段及其过程。

## 选择发行版

我长时间使用过的发行版只有 Ubuntu 和 Arch，这两个发行版的 WSL、桌面和服务器环境我都用过，也简单尝试过 NixOS 和 Manjaro。

Arch 的好处是方便。制作一个 Arch 软件包的成本极低，以至于有些软件的作者可能都不知道自己的软件已经被 Arch 支持了；坏处是容易受到供应链攻击。

我最终还是选择了纯 Arch，没有选择 Arch + Nix、NixOS 或其他不可变发行版。

### 为什么不用 NixOS

我非常喜欢 NixOS 的 Configuration as Code 理念，但是玩不来 NixOS，原因包括：

1. 我安装过几次，过程都不太顺利，进度条很容易卡很久。
2. 进入 NixOS 之后无从下手，最基本的操作都很难进行，仿佛第一次从 Windows 转向 Linux。
3. 我不太喜欢 Nix 语言本身的语法。
4. NixOS 的文档阅读难度有些大，许多地方会毫无征兆地引入超纲概念，没有合理的难度曲线。

虽然其中一些问题在 AI 时代已经不再是问题，但还是有很多地方不符合我的使用习惯。不是说 NixOS 不好，只是我更喜欢 Arch 的随心所欲和没有拘束。

### Vanilla OS

我退而求其次，想找一种和 NixOS 类似的不可变操作系统，于是找到了 Vanilla OS。它把操作系统分成三层：不可变基础、用户桌面和容器化子系统。

子系统使用一种叫 Distrobox 的容器工具。它允许你创建不同发行版的容器、使用各自的包管理器；容器内部的软件也可以访问用户桌面的 X11 和 Wayland 接口来显示 GUI，并访问下载、桌面等用户目录。

随后我意识到，我欣赏 NixOS 并不是因为它不可变。我喜欢的是“不断沉淀一个系统，让它越来越完善”的感觉，也就是 Configuration as Code 理念。因此我没有选择只有不可变特性的 Vanilla OS。

### Universal Blue

我尝试寻找一种新的发行版：可以像 NixOS 一样修改配置文件，然后通过重新构建配置来构筑系统，但配置本身是命令式的，就像编写并增量重构一个 Dockerfile；没有被持久化的改动则会被抛弃。

于是我找到了 Universal Blue。它并不是某个固定的发行版，而是一个构建发行版的框架。这个框架基于 [bootc](https://github.com/bootc-dev/bootc) 搭建：你可以分发标准 OCI 容器，而 bootc 会把这个容器变成可以在宿主机上运行的发行版。

bootc 看起来非常符合我的需求，但 bootc 和 Universal Blue 整体上都更倾向于使用 Fedora，这一点暂时阻止了我继续了解这条路径。

### Arch + Nix

另一条不错的路径是 Arch + Nix（Home Manager）。这样既可以享受 Arch 的方便，又可以把我想沉淀和记录的东西持久化到 Nix。

但是，“持久化到 Nix”的过程成本还是太高。我更愿意把需求写成自然语言，然后让 AI 帮我完成这些操作。

### Gentoo

我简单尝试过 Gentoo，用起来太累，放弃了。

### Ubuntu 和 Debian

使用 Arch 之前，我也觉得 Ubuntu 很好，但是 Arch 实在太方便，日常使用体验确实更好。现在我只会在服务器环境使用 Ubuntu，WSL 和桌面环境则只会使用 Arch。

## Desktop Shell

我注意到，使用触摸板比鼠标更舒服，而触摸板搭配平铺式窗口管理器的效果也更好。使用 Windows 原生的窗口管理方式时，触摸板会很难用，因此我决定尝试平铺式窗口管理。

### 为什么选择 Niri

使用平铺式布局未必一定要选择窗口管理器。Cosmic 就是一个支持平铺式布局的桌面环境，不过听说它离开 Pop!_OS 后不太好用，所以暂时不考虑。我的选项只剩下窗口管理器，最终选择了当下热门的 Niri。

我没有选择 Hyprland，是因为它的版本更新经常带来破坏性变更，让人难以接受。Sway 是 i3 在 Wayland 上的精神继承者，比 Niri 更朴素，但我对 Niri 的无限滚动更感兴趣。

选定窗口管理器后，问题才刚刚开始。窗口管理器只负责其中一部分功能，完整的桌面还需要状态栏、应用启动器、通知、壁纸、锁屏与空闲管理、权限和密码弹窗、X11 应用兼容以及文件管理器等。桌面环境通常会提供一套自洽的软件，而使用窗口管理器则意味着这些组件都要自己挑选和组合。

### 第一阶段：自己组装

DMS 和 Noctalia 这类 Desktop Shell 可以成套提供上述功能，但我看到社区里有不少对它们的批评，所以最初还是决定从头搭建一套。自己组装桌面环境后来证明是一条弯路，不过当时选出的组合是：

1. 终端：WezTerm，因为可配置性比较强。
2. 启动器：Rofi。之前尝试过 Fuzzel 和 Wox，Fuzzel 太老，Wox 在平铺式环境中不太可用。
3. 状态栏：Waybar，因为使用者很多。
4. 通知：Mako，由 AI 推荐。
5. 文件管理器：Nautilus，由 AI 推荐。
6. 登录管理器：一开始使用 greetd + ReGreet，因为比较轻量；后来换成了类似 Windows 的 `win11-sddm-theme`。

整体使用橙色加黑色的 Gruvbox Dark 配色，直接让 AI 按这套配色调整即可。

这套组合基本解决了“缺什么组件”的问题，但每个组件都有独立的配置方式。以后想修改一项桌面设置，就要先找到具体由哪个组件负责，再去编辑对应的配置，维护起来并不方便。

### 第二阶段：Niri + Noctalia

于是我换成了 Niri + Noctalia。最关键的原因是 Noctalia 自带统一的图形化系统设置，补上了自行组装方案最缺少的部分，让窗口管理器用起来更像一个桌面环境。

Noctalia 并没有接管所有组件：终端仍然使用 WezTerm；它也不负责登录管理，因此继续保留 Windows 11 风格的 SDDM 登录界面。

但 Noctalia 的生态还是太弱，经常会有小问题占用我二十分钟到一个小时。它确实降低了配置桌面的成本，却引入了另一种维护成本。

### 第三阶段：回到 GNOME

最后我放弃了 Noctalia，也不再坚持使用窗口管理器，转而选择 GNOME。安装 GNOME 后，我使用 GDM 管理登录，并通过扩展补齐自己需要的功能：

1. Dash to Dock：让 Dock 常驻。
2. Input Method Panel：修复 Fcitx 5 的位置。
3. AppIndicator：提供系统托盘。
4. Rounded Window Corners Reborn：提供窗口阴影和圆角。
5. Clipboard Indicator：管理剪贴板。
6. Caffeine：禁止休眠。
7. Blur My Shell：为顶栏添加模糊效果。

至此，桌面方案经历了三次变化：从自己组装 Niri，到使用 Niri + Noctalia，再到回归 GNOME。每次切换都在减少自由组合的空间，同时换取更完整、更统一的桌面体验。

## 终端应用

为了适应 Linux 桌面，我也整理了一套终端应用：

1. 终端模拟器不跨平台，不需要由 Nix 管理。Linux 首选 WezTerm，Windows 随意。
2. 使用 Niri 时，日常分屏依靠窗口管理器而不是 tmux；tmux 只用于在服务器上保持长时间任务。
3. Shell 及其增强工具：
   - Nushell：作为探索方向。
   - Zsh Autosuggestions。
   - Zsh Syntax Highlighting。
   - zoxide。
   - fzf。
   - Atuin：管理 Shell 历史记录。
4. 编辑器：Neovim。
5. 进程管理：htop。
6. Kubernetes 管理：K9s。
7. SQL 客户端：Rainfrog 或 usql。
8. ShellCheck：检查 Shell 的 lint 和格式。
9. jnv：交互式编写 jq。
10. hl：阅读 JSON 日志。
11. xh：简化版 curl，用于调试 HTTP 接口。
12. gh-dash：GitHub CLI 的 TUI 界面。
13. AI：Codex。

另外还有一些常用命令行工具：

- ripgrep：grep 的现代替代。
- fd：find 的常用场景替代。
- bat：带语法高亮和 Git 信息的 cat 或 pager。
- eza：更现代的 ls。
- dust、dua：磁盘空间分析。
- procs：更易读的 ps。
- hyperfine：命令行基准测试。
- ouch：统一处理 tar、zip、7z、zst、xz 等压缩格式。

## GUI 应用适配

把 Desktop Shell 配好，只意味着桌面框架能够运行，并不代表 GUI 应用已经适配妥当。桌面组件可以重新选择，但应用可能分别基于 Wayland、XWayland 或 Electron，很多行为并不会因为换了一套 Desktop Shell 就自动统一。

### 外观与基础工具

使用 Niri + Noctalia 时，我把整体配色从 Gruvbox Dark 换成了 Rose Pine，并将 Noctalia、VS Code、Obsidian 和 WezTerm 调整为同一套配色。

字体使用微软字体。

> TODO：这里需要具体解释字体方案。

输入法使用 Rime。到这里，外观和基本输入体验大致可以统一，真正麻烦的是不同 GUI 应用的行为。

### 应用兼容问题

QQ 和微信的主要问题是剪贴板同步异常，可以使用 `clipboard-sync` 缓解。图片查看器则存在滚轮速度异常的问题。

网易云音乐使用 r3playx，但需要重新打包。

换回 GNOME 可以绕开 Noctalia 自身的一些问题，但 GNOME 只能替换 Desktop Shell，无法消除应用层面的适配成本。我最初以为自己只是在挑选和配置桌面组件，后来才意识到，真正难以处理的是整个 Linux GUI 生态缺少统一行为。这也成了这次尝试最终失败的主要原因。

## 失败

上次我被 Linux 桌面劝退是因为 N 卡，这次专门买了一张 A 卡来尝试，结果还是不行。

这次的问题是 GUI 应用生态割裂。例如剪贴板：XWayland 应用使用的剪贴板和其他应用不同，需要额外同步，图片和文字的剪贴板处理也不一样；例如滚轮：很多应用没有统一的滚轮控制，各自管理自己的滚动速度，有的快、有的慢；再比如全局热键，没有办法统一管理，有些需要全局热键的应用甚至不知道应该去哪里注册；GUI 缩放也不统一。

这些问题太麻烦了。如果要解决它们，那还不如不用 GUI。

最终的回退方案是：Windows 只用来打游戏，MacBook Air M1 只用来写代码，MacBook Air M1 性能不足的地方则远程连接 Linux。

至于为什么不用 X11：虽然很多问题是 XWayland 造成的，但 Electron 也不遑多让。重点还是生态问题，这并不是某一个人能够解决的，需要整个社区共同推动。
