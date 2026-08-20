# 未来 js 是什么样子

> 2026年8月20日注，这篇文章主要意义其实是我首次认真接触工具链，已更改文件名，但标题保留原文不变

本文主要聊聊现代 js/ts 在工具链和前后端领域的技术栈发展情况。

## 概述

毋庸置疑的是，现在的js开发者的开发方式都是写 typescript（或者即使是写js，但也需要经历构建），然后编译构建成 js 在各平台上运行 —— 这一句话包括了我们要聊的三个主要概念：typescript（开发时工具），编译构建和js运行时，js工具链完全围绕这三个概念生长。

上个时代的话题是前端工程化，我们会讨论:

1. git 版本管理
2. css 预处理器
3. jest 测试
4. eslint、prettier 控制代码格式和规范 
5. typescript 控制类型安全
6. webpack/vite 进行从 node 到浏览器的打包
7. yarn/pnpm 优化包管理器
8. 前端开发框架 vue/react/angular/...

不清楚是 js 本身的进步还是我个人的进步，我接触的话题开始发生了一些转变，但大致领域还是差不多的，下面随心的漫谈一下。

## github 的受信任度下降

按上面的列表顺序说，首先 git 方面没什么变化，框架语言年年换，git 和 linux 却永远是最高的山和最长的河。但略有一些变化的是 git 托管服务，26年开始社区对 github 的信任度出现了下降，主要有以下几个点：github 背靠微软，生态和资源都非常雄厚，本身就有垄断嫌疑；24年和25年有过一些 copilot 训练模型时不正当使用开源资产的行为；26年可能是由于github也引入了许多ai代码服务稳定性有所下降，cicd稳定性问题出现过几次风波（也有说法是因为github的Agent相关功能占用了大量的ci资源）；26年的5月，似乎是由于 AI 在网络安全方面的能力提升，集中爆发了非常多的安全漏洞，像 github 这样的大规模平台会更容易收到攻击。

总之我开始听到更多搭建私有git托管的声音 —— 当然这非常主观，我没有太多证据，我简单看了一下 gitea 的 star 增速（见下图），并没有迹象说明26年gitea的热度有超出预期的增速，如果 gitea 有比较明显的热度提升，那么可以轻微的辅助论证 github 正在流失用户，可惜 gitea 没有。

![](https://api.star-history.com/chart?repos=go-gitea/gitea&type=Date)


如果你要部署私有 git 服务，相比 gitlab 我更推荐 gitea，基于 golang 开发，并且功能更新相对克制，这使得资源占用很低，而且 gitea 的仓库源码非常优雅，还是国内公司开发的开源项目，gitea的社群（QQ群 328432459）氛围也非常好。


## css样式

对于 css，我曾经会说，有 css预处理器、css原子化、css in js以及一些组件库。但是对于26年中的情况来说，感觉预处理器和cssinjs已经路边了，在 ai 的偏好放大下，tailwind + 组件库的方案正在统治css世界。

另外有一个小技巧，tailwind + 组件库只能把常见的样式排列组合，如果要写点不一样的其实可以用svg，思路会打开许多。


> 测试 没什么好说的，我其实不太写测试，我大概知道 vitest 正在代替 jest, 这里略过。

## typescript: tsgo

25年发生一件大事，就是微软用 golang 重写ts编译器，实现了约10倍的性能提升，我一直知道这件事但是没有用上，这个项目叫做typescript go。

但是我到现在，26年六月才想起来这个问题，到底咋用。其实他已经快正式发布了，typescript 7就会直接使用tsgo，但是目前想用也差不多可以用了。

在运行时使用tsgo:
```bash
npm install @typescript/native-preview
npx tsgo # Use this as you would tsc.
```

在 vscode 中使用 tsgo: [TypeScript (Native Preview) - VSCode Plugin Marketplace](https://marketplace.visualstudio.com/items?itemName=TypeScriptTeam.native-preview)


## typescript: 运行时类型安全

typescript 的 type 和 interface只停留于编译期，最终运行的还是动态类型的js，我对这一点非常不喜欢，像go的结构体就可以在运行时保留，我更喜欢这样的设计。

其实也可以通过 node 原生的方式在运行时保留类型，那就是用class代替type，但我没有那么喜欢OOP，大多数js开发者也是，所以我们使用zod。

zod 用起来和type、struct差不多，也提供了自动化的运行时校验，生态也非常好，堪称 ts 必备。

## js运行时: node, bun 和 deno

### bun 体验一般

我没怎么了解过 deno，感觉生态比 bun 还小，但是我尝试过两次 bun，体验都非常不好。

第一次是我直接尝试了解 bun，我看到它内置了一个SQLite实现，但是实际情况是死活连不上。

第二次就是最近，我试图用 bun 去开发一个 nuxt + elysia 项目，bun官方文档写着 [`Bun supports Nuxt out of the box.`](https://bun.com/docs/guides/ecosystem/nuxt)， 而 elysia 更是一个完全 bun 原生的后端框架。

但是两个问题直接劝退了我，一个是 nuxt 的某个功能（好像是页面导航相关）在 bun 下无法正常工作，居然要回退到手动使用最原始的浏览器 API；另一个是 elysia 的类型推导居然有问题，写起来全是隐式any，实在是忍无可忍。

bun 作为 js 运行时来说，至少在前后端开发场景，还有很多工作要做。不过现在的coding agent包括claude code、oh my pi、opencode 都是 bun 开发的，在 TUI 应用和二进制打包这一块 bun 还是有一席之地，bun 作为包管理器其实还可以考虑。


### node 最近的更新

我平时会关注 python 的版本更新，但是我从来没关注过 node 在更新什么，借此机会来看一下：

> **Node 18 (2022)**
> - `fetch` 稳定——从此告别 `node-fetch` / `axios` 发请求
> - `node:test` 内置测试框架，最早萌芽
> - `--watch` 文件监听重启
> 
> **Node 20 (2023)**
> - `--env-file` 自动加载 `.env`
> - `node:test` 稳定
> - `import.meta.resolve`
> - 权限模型 `--experimental-permission`
> 
> **Node 22 (2024)**
> - `node:sqlite` 实验性
> - `--experimental-strip-types` 直接跑 `.ts`
> - WebSocket 原生（`new WebSocket(...)`）
> - `glob` 文件匹配
> 
> **Node 23 → 24 (2025-2026)**
> - strip-types 默认开启（仍在实验期）
> - `--experimental-transform-types` 支持 enum、namespace 等需转译语> 法
> - SEA 单二进制打包进入稳定

其中我比较感兴趣的是 **SEA、sqlite和原生typescript支持**。

### node:sqlite 和 pglite

我之前一直在用 better-sqlite3，node:sqlite 性能尚可但不如 better-sqlite3，因为 better-sqlite3 是通过 node-gyp 编译原生 C++构建来的，所以可以说node内置实现用起来更方便，better-sqlite3 性能更好更成熟。

那么除了 sqlite，能不能把 postgres 也塞到自己的进程里？答案是可以，用pglite。

pglite 的思路是把 postgres 引擎编译成 wasm，v8支持在隔离空间里运行wasm，这意味着 pglite 甚至是一个可以塞到浏览器里的postgres引擎。不过这样做是有代价的，虽然完全兼容原生的 postgres，但是要使用 postgres 插件就困难了，也要让他们都适配wasm，并且编译成wasm造成了一些性能损失。

pglite带来了这样一种可能性：开发和测试时用 pglite 把 postgres 塞到进程里（并且由于是wasm，可以支持许多现代语言），生产运行时再连接正式的 postgres。

### 二进制打包

node通过SEA的方式支持二进制打包，而bun从一开始就把编译当做一等公民。

SEA 的思路是，直接把 js 代码注入到已有的 node 二进制里面，这意味着你不能携带像better-sqlite那样的需要C++的包，只能用纯js，同时体积也不小。

bun的构建同样是自带 Bun 运行时，但node是在二进制里塞js，bun是编译的时候连带bun的源码一起编译，这使得bun可以做tree shaking优化，没用到的代码可以直接不参与构建，这种连带bun源码构建的能力可以做很多事情。

> 1. 交叉编译 — bun build --compile --target=bun-linux-x64 在 macOS 上打出 Linux 二进制，因为 bun 的运行时本身有跨平台源码，直接参与编译
> 
> 2. 原生模块也能带 — Node SEA 排斥 better-sqlite3、sharp 这类带 .node 的 C++ 模块，bun 因为走的是编译而非注入，没有这个限制
> 
> 3. 自定义裁剪 — 你只用 bun 的包管理功能，编译时可以不要它的测试框架和打包器模块，产物更小
> 
> 4. 依赖内联 — 不光是 tree shaking，bun 编译时能把 node_modules 里的依赖也一起编译打包，最终产物只有一个文件，不依赖外部 node_modules
> 
> 5. 运行时优化 — 因为 bun 源码（Zig/C++）和你的 JS 一起参与构建，可以做跨语言的内联优化，比 Node SEA 的「JS 脚本贴在二进制尾巴上」强

### typescript支持

其实我觉得在运行时层面内置 ts 支持其实帮助不是特别大，因为正儿八经的生产环境肯定跑的是 ts 编译后的js，直接运行的ts场景只有某些小脚本你也想用ts写而不是js这一个场景。

typescript 本身属于开发时的工具链，对源码进行 typecheck 来辅助查找错误用的，js运行时是否需要内置某些工具链，这本身就是一种设计取向，至少node是不需要的，只有bun和go会这么做，node干这个事情我只能理解为妥协，避免 bun 拿这个点来说事。

bun 和 node 在这块的实现思路差不多，叫 strip-type（类型剥离）：把 TypeScript 的类型注解抠掉之后直接当 js 跑，如果是正儿八经的ts编译还需要做类型检查和语法降级，因为ts编译可以设置不同的 target，但是 strip-type 不需要考虑多target。


## 包管理器

我用了很久的pnpm，在长久的使用中，感知最明显的相比 npm 的优点就是monorepo支持和缓存（其实npm也有monorepo支持，到写这篇文章的时候我才知道）

我又了解了一些pnpm的其他优点，我觉得比较实用的还有: pnpm 的 lockfile 比 npm 能更加精准的锁定具体的包，因为pnpm的lockfile还会计算包的哈希值；pnpm有个patch功能，可以对包进行一些修改；

我又看了一圈目前的js的包管理器情况，似乎暂时没有什么后起之秀能够挑战 pnpm。

bun 做为包管理器来说，它的 lockfile 是二进制的读取更快，安装包的时候也比 pnpm更快，但没办法赌它会不会在better-sqlite3这样需要 node-gyp 编译 C++ 原生模块的包上有兼容问题。

还有一个 vlt，他完全是一种和npm格式不同的 registry 协议，过于激进了。

## bundler 和 void zero

终于说到了我们压轴话题，js 工具链终将迎来它的皇帝 void zero。

先来看一下js工具链整体生态: 

```
源码阶段
  Package Manager   npm / pnpm / yarn  
  Type Checker      tsc                →  tsgo (TS7, Go 重写)
  Linter            ESLint             →  Oxlint / Biome
  Formatter         Prettier           →  Oxfmt / Biome

构建阶段
  Transpiler        语法转译          swc / esbuild / Oxc Transform
  Bundler           打包              Rolldown / Rspack / esbuild
  Minifier          压缩              swc-minify / Oxc Minify / esbuild
  Resolver          模块路径解析       Oxc Resolver / enhanced-resolve

测试阶段
  Test Runner       vitest / jest     (暂无 Rust 替代，Bun 自带)
  Coverage          c8 / istanbul     (暂无)
  Mock              MSW / vitest-mock

部署/运行
  Runtime           Node / Bun / Deno
  Dev Server        Vite dev server
  Hot Reload        Vite HMR / Bun --hot
```

这张图里除了包管理、类型检查和js运行时我们前面已经说过了，剩下的部分void zero团队会包揽linter、formatter和整个构建阶段，以及在测试阶段中获得庞大的生态。

VoidZero 是谁？

> 1. 2024 年成立，Evan You（尤雨溪）创办，就是 Vue 和 Vite 的作者
> 2. 旗下产品：Oxc（parser/linter/formatter）、Rolldown（bundler）、Vite（dev server + 构建编排）、Vitest（test runner）
> 3. 2026 年加入了 Cloudflare，资金和 Edge 生态打通
> 4. 目标：做 JS 世界的 Cargo/rustc/clippy/rustfmt——全链 Rust 统一工具链
> 5. 除了包管理、类型检查和运行时，其余全归它

可以看到 Evan You 很清晰的一条发展路线: vue => vite => 整个 js 工具链, vue 就不说了，从vite开始说。

vite是一种构建编排工具，可以给项目编排多种不同的构建路径，vite的设计风格为约定大于配置，所以有一套默认的构建方式


> 1. 入口是 index.html（不是 src/index.ts），Vite 从 HTML 出发递归分析所有 script 和 link 依赖
> 
> 2. 开发时不做 bundle，浏览器原生 ESM 按需加载模块，dev server 即时转译 TS/JSX/CSS
> 
> 3. 生产时底层调 Rollup，自动帮用户完成 tree-shaking、代码分割、scope hoisting、压缩，用户不必碰 rollup 配置
> 
> 4. 代码分割按 import() 自动切 chunk，不需要手动配 splitChunks
> 
> 5. 静态资源放 public/ 的直拷到 dist/，代码里 import 的图片/字体等会自动哈希命名
> 
> 6. CSS 自动处理：import './style.css' 就生效，*.module.css 自动开 CSS Modules，PostCSS 自动检测 postcss.config.js
> 
> 7. TS/JSX 零配置，开箱即跑，但只转译不做类型检查
> 
> 8. 环境变量：VITE_ 前缀的变量注入客户端，私密信息不泄漏
> 
> 9. dev server 默认 5173 端口，HMR 即开即用

而这些统统都是可以配置、替换的，可以说任何ts项目的构建都可以使用vite管理，几乎可以理解成ts世界的编译器，我们可以类比c/rust/go的行为，vite 的工作就是把源码构建成不同运行环境的 js，构建出来的 js 产物可以类比为二进制机器码，浏览器、node、serverless 等运行环境可以类比为不同CPU架构，ES标准可以类比为IR。

```
C/Rust 世界                           JS 世界
────────                             ────────
源码 (.c/.rs)                        TypeScript (.ts)
   │                                    │
   ▼                                    │ tsc/tsgo 剥离类型 
中间表示 (IR)                            ▼
   │                                 JavaScript ES2024+
   │                                    │
   ▼ Rolldown/esbuild/swc               │
不同平台的机器码                      不同 target 的 JS
   │                                    │
   ├─ x86_64-linux                   ├─ esnext (不降级)
   ├─ aarch64-macos                  ├─ node22
   └─ wasm32-wasi                    └─ es2015
```

> 我很长时间以来一直以为bundler只是从node环境构建出能在浏览器环境运行的产物，其实这个描述实在是过于狭隘了，vite的lib模式构建出来的包是可以在node环境运行的。

然后来看一下 bundler 的演进，没有太多好介绍的，直接看图了解一下即可。
```
2012-2020  JS 时代
  webpack ───────── 全功能但慢
  Rollup ────────── tree-shaking 之王，库构建
  Parcel ────────── 零配置先驱

2020-2024  原生跃进
  esbuild (Go) ──── 快 100x，但功能窄
  SWC (Rust) ────── Babel 替代
  Rspack (Rust) ─── webpack 兼容层 + Rust 速度
  Turbopack (Rust) ─ Next.js 专用

2024-2026 两大阵营

VoidZero 生态（Vite 团队）  
    ┌──────────────────────────┐
    │ Oxc      解析/检查/格式化 │
    │ Rolldown 打包             │
    │ Vite     开发服务/构建编排│
    └──────────────────────────┘
    全部 Rust，统一工具链大一统

独立阵营
    esbuild    仍然最快，但功能不再扩张
    Rspack     webpack 用户的最佳迁移路径
    Biome      独立 Rust linter+formatter
    Bun         自带 bundler（集成在运行时里）
```

目前虽然也有 "一家公司控制整个 JS 工具链"的担忧，但 Evan You 的 MIT 开源信誉 + Cloudflare 入局 + 多竞争者并存，目前还不到垄断那一步，而且这些工具质量都得到了社区认可:

| 项目     | Stars  | 团队                | 赛道                             |
| -------- | ------ | ------------------- | -------------------------------- |
| Vite     | 81,431 | VoidZero            | dev server + 构建编排            |
| Oxc      | 21,572 | VoidZero            | parser/linter/formatter/minifier |
| Vitest   | 16,703 | VoidZero            | test runner                      |
| Rolldown | 13,726 | VoidZero            | bundler                          |
| esbuild  | 39,921 | Evan Wallace (独立) | bundler/minifier (Go, 基本停滞)  |
| Biome    | 24,988 | Biome 独立团队      | linter + formatter (Rust)        |
| Rspack   | 12,754 | ByteDance           | bundler (Rust, webpack 兼容)     |

## 总结

写这篇文章不仅在梳理我自己对js现代技术栈的理解，也一边查资料学到了新兴的工具，我也准备尝试使用vite的更多功能，以及 Oxc、Vitest、pglite、tsgo.