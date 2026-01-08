# Obsidian 使用偏好

总体上我更愿意使用符合 Common Markdown 标准的设置，以及保持 Obsidian 和 Vscode 的一致。

## Obsidian 设置
### 编辑器

1. 开启严格换行
2. 笔记属性显示为源码
3. 关闭缩进行的参考线
4. 关闭折叠缩进

### 链接

1. 内部链接类型：相对路径
2. 始终更新内部链接：是
3. 使用 Wiki 链接：否
4. 检测所有文件类型：是
### 快捷键

快捷键要注意搜狗输入法对快捷键的抢占，尤其是 `Ctrl + ,`

1. 开关左侧边栏：Ctrl + Shift + E
2. 开关右侧边栏：Ctrl + Shift + B
3. 搜索文件： Ctrl + P
4. 命令面板：F 1
### Settings Search

这是一个插件名，插件的作用是可以搜索设置，插件本身没有什么需要设置的。

### 外观

1. 主题：我曾经很喜欢用 AnuPpuccin, 后来觉得折腾主题实在是无边地狱，就一直使用默认主题了。
2. 字体：所有字体改为 `Jetbrains Maple Mono`
3. 界面：
	1. 关闭 "显示页内标题"
	2. 关闭 "显示功能区", 可以直接用命令，完全不需要功能区。
4. Css 片段： 默认主题有一些小缺点，需要手动添加一些 css 代码片段。
```css
/* default-theme-enhance.css */

/* 
选择: 编辑时的行内代码块 和 渲染时的行内代码块
效果: 字体加粗，加深背景色 250 -> 240 */
.cm-inline-code,
code:not(pre code) {
    font-weight: 600;
    background-color: rgb(240, 240, 240) !important;
}

/* 
选择: 编辑时的代码块 和 渲染时的代码块
效果: 背景色加深 250 -> 245 */
.HyperMD-codeblock,
pre {
    background-color: rgb(245, 245, 245) !important;
}

/* 
选择: 编辑时的粗体字 和 渲染时的粗体字 
效果: 更粗 */
.cm-strong,
strong {
    font-weight: 700 !important;
}

/* 设置引用块背景色在亮色模式和暗色模式下的取色 */
.theme-light {
    --BlockqutoeColor_background-color: #f3effa;
}

.theme-dark {
    --BlockqutoeColor_background-color: rgb(66, 46, 85);
}


/*
选择: 编辑时引用块左侧的竖线
效果: 加粗*/
.markdown-source-view.mod-cm6.is-live-preview .HyperMD-quote:before,
.markdown-source-view.mod-cm6 .cm-blockquote-border:before {
    --blockquote-border-thickness: 5px !important;
}

/* 
选择: 编辑时的引用块
效果: 设置背景色和padding */
.HyperMD-quote {
    /* padding-top: 0.8em !important;
    padding-bottom: 0.8em !important;    */
    line-height: 1.9;
    padding-right: 0.8em !important;
    background-color: var(--BlockqutoeColor_background-color) !important;
}


/* 
选择: 渲染时的引用块
效果: 设置左侧竖线, 背景色和padding*/
.markdown-rendered blockquote {
    border-inline-start-width: 5px;
    padding-top: 0.8em;
    padding-bottom: 0.8em;
    background-color: var(--BlockqutoeColor_background-color);
}
```

## 附件管理

### Custom Attachment Location

这个插件的设计目的和我的需求一致，就是能够通过变量定义附件的保存路径，默认路径为 `./assets/${noteFileName}` , 就足够满足我的需求

其他设置如下：
1. 附件重命名格式：所有格式文件都会被重命名

### Image Converter

这个插件功能很多，我的主要需求是
1. 能够手动定义附件名称，并且同时修改文档中的引用路径和实际的附件文件名
2. 调整图片大小

我改动的设置全都在 Links Format 子页：
1. 格式选择 Markdown Relative
2. 关闭 Image Alignment，因为不符合 Markdown 原生标准
3. Show window 设置为 `always`, 这样每次粘贴都会询问你如何设置文件名

## 编辑器

### Paste Link

解决我两个需求：
1. 直接粘贴链接的时候，会连带粘贴网页的 title
2. 选中文字的时候粘贴，这段文字会链接向这个地址。

设置改动：
1. 打开 fetch page titles on paste

### Word Splitting for Simplified Chinese in Edit Mode and Vim *Mode*

这是一个中文分词插件，主要实现是使用 Ctrl + 方向键的时候，能够对中文进行分词。

### Blockier

这个插件的作用是，使用 Ctrl + A 的时候能够选择当前块，再次 `Ctrl + A` 才会选中整个文档。

这是一个需要手动安装的插件：
1. Github 仓库地址：[https://github.com/blorbb/obsidian-blockier](https://github.com/blorbb/obsidian-blockier) 
2. 在 Obsidian 仓库的 `.obsidian/plugins` 目录下新建一个 `obsidian-blockier` 目录
3. 把 Github Release 页面中的 main. Js, manifest. Json, style. Css 复制到刚刚新建的目录
4. 去 Obsidian 设置刷新已安装的插件，然后启用这个插件即可。

设置改动如下：
1. 开启 Select all on double activation, 即双击 Ctrl + A 选中全文
2. 开启 Select full code block，即 Ctrl + A 选中整个代码块。

### Outliner

这个插件的作用是，能够实现一些快捷的列表操作，我的需求是通过 `alt + 方向键` 调整列表项的顺序。 

有些设置不太看得懂作用，我的设置改动如下：
1. 关闭 Drag-and-Drop
2. 快捷键设置里 `Outliner: Move list and sublists down` 改成 `alt + ↓`
3. 快捷键设置里 `Outliner: Move list and sublists up` 改成 `alt + ↑`
### Easy Typing

这个插件概括性的来说，用于格式化输入，说几个我觉得特别有用的：
1. 支持中文符号的成对输入：输入 `《` 会自动弹出 `《》`
2. 中文符号转化成 markdown 语法：比如 `···` 会自动变成代码块
3. 中英文之间自动添加空格

只保留这几个功能，剩下的我全部关闭了。

## Open vault in VScode

作用和插件名称一样，不需要额外设置，我一般通过命令面板打开 vscode。

## 待探索

1. Quickadd
2. 我需要了解一些 linter 相关的功能，我想统一文档的规范，具体规范可以参考[阮一峰 中文技术文档的写作规范](https://github.com/ruanyf/document-style-guide) 。
