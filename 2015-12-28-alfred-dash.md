---
title: Alfred 和 dash 介绍
date: 2015-12-28 13:33
categories: [技术]
tags: [MAC, alfred, dash]
---

你生活在二维 Mac 世界里，在`Applications`下散落着各种应用，在`Finder`下散落着各种文件和文件夹，而文本内容又散落在各个文件中。命运如潮水般汹涌而来，你在平面里穿梭，寻找那命中注定的字符串。

够了！你冲出纷至沓来的字符串洪流，艰难地走向书柜，掏出一本「哆啦A梦」--- 是时候穿越时空了！

![空间翘曲](http://static.wulfric.me/space-wrap.png "空间翘曲")

alfred 就是 Mac 世界里穿越时空的飞船。

## Alfred

[Alfred](http://www.alfredapp.com/)作为 Mac 下较为常用的软件，已有大量教程，比如[教程1](http://macshuo.com/?p=625)，[教程2](http://wellsnake.com/jekyll/update/2014/06/15/001/)等。本文主要介绍 Alfred 的基础功能，workflow 以及与 dash 的集成。 

### 基础功能

打开`Alfred Preference`，在`Features`选项卡中的内容就是免费版提供的各项基础功能，如图。

![Alfred Preference](http://static.wulfric.me/alfred-preference.png)

把上述内容过一遍，就能对 Alfred 有一个比较全面的印象了。

最常用功能：定位 APP。`Option+Space`键打开后，键入 APP 名称，便可以轻松打开 APP。名称输入支持部分匹配，但不支持模糊匹配。

计算器：简单计算，也可输入`=`使用高级函数比如「sin, cos, log, ln」等。

定位文件：`Option+Space`键打开并输入`find`，空格之后输入文件或文件夹的部分名称，便可找到其位置，确定后将会在 Finder 中定位该文件；如果输入的是`open`，确定后定位并打开该文件；`in`，搜索文件中的信息。该部分配置项在`Features`选项卡中`File Search`部分。

剪贴板：Alfred 也可以操作剪贴板。获得剪贴板历史，合并剪贴板等。除此之外还可以自定义 snippet。snippet 可以操作的变量只有三个：time, date, clipboard。我做了一个 snippet，快捷键是`sj`，内容是`{date:medium} {time}`。输入`Option+Cmd+c`调出 Alfred 的剪贴板工具，输入`sj`，即可输出符合 Jekyll 格式的当前时间。PS: date 的 short, medium 和 long 的配置见系统设置-时间和日期-语言和地区中。

![时间设置](http://static.wulfric.me/systempreference-datetime-languageregion.png)

### workflow

workflow 作为 Alfred 的付费功能，提供了相当强大的自定义功能。workflow 支持调用脚本（shell/ruby/python/php），这为 Alfred 提供了无限可能。比如这个 workflow，就是调用 Python 脚本实现的一个有趣的功能。

![alfred workflow 真相](http://static.wulfric.me/R-alfred-zhenxiang.png)

![alfred workflow 示例](http://static.wulfric.me/alfred-workflow-example.png)

豆瓣 workflow：输入 movie/music/book，查找豆瓣信息。
![alfred workflow douban](http://static.wulfric.me/alfred-douban.png "alfred workflow douban")


## Dash

[dash](https://kapeli.com/dash)是 Mac 下的文档集合工具，支持海量语言和插件。其原理倒是比较简单，就是把各种语言和框架的文档下载到本地以供查找。一方面可以节约网络查询的成本，另一方面可以原生提供针对几乎每一个编辑器和 IDE 的支持，非常的便利。

![dash](https://kapeli.com/img/dash-s1.png)

### 和 Alfred 配合

配合 Alfred 下的 dash workflow，可以轻松的查找文档，如图所示。

![dash alfred](http://static.wulfric.me/dash-alfred.png)

## 结语

直方图一样丛生的石头森林，命运之线下苦苦挣扎的傀儡，远方朝露晨雾中若隐若现的灯塔，这是个纷纷扰扰的世界。想要成为一个穿越时空的 ~~少女~~ 少年。[^1]

![穿越时空的少女](http://static.wulfric.me/chuanyueshikongdeshaonv.png "穿越时空的少女")

[^1]: 图片来源于剧照，和 google 图片搜得。用 google 按图找图也没有找到原版权所有者，找到的全是素材站你懂的。侵权请通知我删除。