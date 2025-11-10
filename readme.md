# Blog

我的博客。

## 目录

1. [caddy和CA认证： 如何把我的链接从HTTP变成HTTPS？](./network-caddy-certificate.md)

## Todo

### 文章同步问题
博客的文章是从obsidian vault复制粘贴来的，如果日后在obsidian内更新，博客无法及时同步。

我尝试使用硬链接，我的编辑确实能够同步，但是我的obsidian仓库也是一个git仓库（私有），在git pull后就不会同步了。

如果我保持两个仓库，则有两个策略，一个是写一个github actions或者obsidian插件来保证内容同步，缺点是我需要自己写actions和插件；另一个是博客和obsidian内容分离，相同的内容只保持一份。缺点是维护体验一般。

如果我保持一个仓库，则可以通过一个已有的obsidian插件把markdown导出为html页面，似乎是一个不错的选择。


### 阅读体验问题

<s>我的文章通常内容较长，如果没有目录很容易阅读体验不佳。尤其github没有自带的悬浮目录。可能手动添加目录，或者自己搭建页面可以解决问题。</s>

你可以使用这个chrome插件为 github 添加 大纲 / toc  [GitHub TOC Sidebar](https://chromewebstore.google.com/detail/github-toc-sidebar/cdiiikhamhampcninkmmpgejjbgdgdnn?hl=zh-CN)

