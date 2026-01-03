·# Blog

这是我的博客。


<!-- 此处应有一个 类似 spacesniffer 的热点图 -->


## 设计意图

1. ***纯 Markdown 仓库***: 我通过 Obsidian 写作，但是内容始终保持 Common Markdown 格式以保证可迁移性。我很愿意在git中保存一些我的 Obsidian 配置，但是在 `.obsidian` 中，插件的用户配置和插件生成的数据共用同一个 `data.json` 文件，我不想用git保存这些由插件程序生成的数据。所以这个仓库不应该追踪 `.obsidian` 的变更，光从git的视角也就看不出 Obsidian 的痕迹了。