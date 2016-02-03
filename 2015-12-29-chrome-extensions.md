---
title: chrome 效率插件推荐
date: 2015-12-29 10:24
categories: [技术]
tags: [chrome, 插件, vim, proxy, markdown, develop, github]
---

## 个人效率

### vimium

vimium 是 chrome 下的 Vim 模拟器，可以模拟 Vim 的快捷键来进行 chrome 的相关操作。我一直认为，如果你希望学会使用 Vim 编辑器，从 vimium 插件入门是一个不错的选择。vimium 的快捷键如下图所示：

![vimium](http://wulfric.qiniudn.com/vimium.png)

最基础的快捷键是页面滚动：

```
j: 向下滚动一行
k: 向上滚动一行
u: 向上滚动一页
d: 向下滚动一页
gg: 滚动到页面顶部
G: 滚动到页面底部
```

Vim 的上下左右移动的快捷键是 hjkl，vimium 也是这样，但是 web 页面通常没有左右移动的需要，只需要 jk 即可（JK 大好~），gg 和 G 省去了你安装各种手势插件的需求。在 Vim 中滚动页面的快捷键是`ctrl+u/d`，vimium 这里只需要`u/d`即可。

翻页：

```
Previous patterns: prev,previous,back,older,<,←,«,≪,<<
Next patterns: next,more,newer,>,→,»,≫,>>
```

上面是上下翻页的配置，你也可以配上中文，比如「上一页」和「下一页」。配好之后便可以使用`[[`和`]]`来翻页了。

快速跳转：

```
T: 列出当前所有标签页
o: 搜索你的历史纪录
b: 搜索你的收藏夹
f: 给当前页面的链接加上标记，方便跳转，跳转在当前页面完成
F: 同上，跳转在新标签页完成
```

这个不必多说。其中 F 和用鼠标中键点链接和 MAC 下用`cmd`加点击链接的效果是一样的。

`Treat find queries as regular expressions`在 vimium 配置页面可以将 vimium 的查找改为正则模式，这样查找页面内容更加方便了。

因为 chrome 的安全策略，所有插件都无法在 new tab 页工作，所以 chrome 下的 Vim 插件都不如 Firefox 下易用，在 chrome 下使用 Vim 插件需要尽量避免进入 new tab，另一方便 chrome 自带的一些快捷键也需要有所了解，这样才能在 vimium 失效的时候继续脱离鼠标操作。

~~然而换了 MAC 后触摸板太好用了……感觉用 vimium 的频率大大下降了。~~


### one click extensions manager

一图解千言。管理插件利器。

![one click extensions manager](http://wulfric.qiniudn.com/R-one-click-extension-manager.png)

### proxy switchyOmega

[SwitchyOmega](https://github.com/FelisCatus/SwitchyOmega) 不多做介绍了，从 Proxy Switchy!, Proxy SwitchySharp, 又或者是其它插件，你的浏览器中，总是需要有一个。该插件确实比其前作更加强大一些。

### zotero

论文管理利器，如下图所示。在有下载权限的前提下，可以一键下载论文到指定目录，生成索引等。现有的 pdf 论文库的迁移也很方便，有相应的插件支持。

![zotero](http://wulfric.qiniudn.com/zotero.png "zotero.png")

推荐参考阳老的 zotero [博文](http://www.yangzhiping.com/tech/zotero1.html)。一共有 6 篇。

### clearly

chrome 下的 evernote 插件，优雅浏览页面和同步到 evernote，还可以加标注。

### copy as markdown

把标签页的 URL 直接转成 markdown 格式。

### markdown here

gmail 等富文本输入框中可以先输入 markdown 然后右键 toggle 转成富文本，如下图示。

![markdown toggle](http://wulfric.qiniudn.com/markdown-toggle.png "markdown-toggle.png")

## 辅助开发

### postman

Postman 是强大易用的 API 测试工具，如下图所示。

![postman](http://wulfric.qiniudn.com/postman.png "postman")

左侧是常用 URL collection 和 history，还可以按照个人和团队来区分 collection，非常的方便。主面板就是调试页面，功能非常强大，提供了很多方便的工具，其中之一是将当前在调试的请求直接生成代码，如下图示，目前流行的语言基本都可以支持。此外还有环境变量和预设值，以及测试集成功能。

![postman code](http://wulfric.qiniudn.com/postman-code.png "postman-code.png")

### json formatter

调试的时候 get 的 json 页面可以以结构化的形式显示。这样的插件很多，功能也都是大同小异，随便选一个安装即可。

![json formatter](http://wulfric.qiniudn.com/R-json-formatter.png "json formatter")

### coffeeconsole

也许你已经厌倦了 js 的语法，想要使用 coffeescript 来调试。

### octotree

树形结构显示 github repo 中的文件，如下图所示。

![octotree](http://wulfric.qiniudn.com/R-octotree.png)

### web tech stack 探测

探测一个页面使用的技术。

![web tech stack 探测](http://wulfric.qiniudn.com/R-web-tech-stack.png)

## 系统增强

### ublock origin

更好的广告屏蔽插件。可以很直观的看到当前页面的屏蔽列表，还可以临时放行或屏蔽相关 URL（根据域名或 MIMETYPE）。私以为要强于 ABP。

![](http://wulfric.qiniudn.com/R-ublock-origin.png)

### google art project, speed dial 2, 各种 new tab

不多说了，看你希望你的 new tab 是什么样子的，是好看，还是高效。

### office 预览和编辑

Google 官方插件。装该插件之前，网页上的 office 文件点击之后会下载，装了之后可以直接浏览 office 文件。

### tampermonkey

神器，尤其是当你知道些 js 知识的时候，不需要专门写一个 chrome 插件，只需要一点点 js 代码即可。还可以在 [userscripts](https://greasyfork.org/) 里找他人写好的插件。

### stylebot

可以自定义相关页面的样式，还可以导入他人已经配置好的样式，如下图所示。

![stylebot](http://wulfric.qiniudn.com/R-stylebot.png)