---
title: jekyll 和 octopress 博客程序的选择
date: 2016-05-11 17:13
categories: [技术]
tags: [jekyll, octopress, hexo]
---

首先，你需要理解 jekyll 和 octopress 的本质区别。jekyll 是运行在 github 的服务器上的，他可以 **自动** 把你上传的 md 文件转成 html。也就是说，在一个配置好的 repo 中，你只需要更改或新增 md 文件，你的博客内容就自动更改好了。但是，octopress 不一样。github 的服务器上没有这么一个服务，所以你需要在本地先把 md 文件转换成 html，然后把 html 文件发布出去。

```
+----+
     |
     +-+article1.md
     |
     |
     +-+article2.md
     |
     |
     +-+article3.md
     |
     |
     +-+article4.md
```

上图是 jekyll 的 master 分支的结构（隐去了其他必要的文件，只留下 md），github 服务器上运行的 `jekyll server` 命令会监控这里的变化，并自动把 md 文件转成 html host 出去。

```
+-----+                     +----+
      |                          |
      +-+source folder           +-+index.html
      |                          |
      |                          |
      +-+public folder           +-+2016 folder （里面是各种 html）
      |                          |
      |                          |
      +-+css folder              +-+2015 folder （里面是各种 html）
      |                          |
      |                          |
      +-+others                  +-+others
```
      
上图是 octopress 的目录结构，左边是 source 分支，右边是 master 分支。octopress 的逻辑就是，在 source 分支的 source 文件夹里写 md 文件，然后执行一个转换命令，将生成的 html 文件放在 public 文件夹里。而 public 文件夹的内容就是该 repo 的 master 分支，即右边树就是 public文件夹的内容。

PS: jekyll 的目录中同样有类似 octopress 的 source folder 和 public folder，这个暂时按下不表。

这样，我们就可以总结出二者的优缺点：
- jekyll 极为便利，比如你在移动中突然有了灵感，你可以直接在手机上登录 github，修改 md 文件，马上你的博客就更新好了。但是如果使用 octopress，你虽然也能改 md 文件，但是你不能在手机上直接把 md 转成 html，所以无法实时更新博客。
- octopress 定制性更好。因为 jekyll server 是运行在 github 的服务器上，考虑到安全因素，github 禁止了几乎所有的 jekyll plugins，所以这种 host on fly 的部署方式受到很大的限制，而 octopress 是在本地转换成 html 再发布，任何插件都可以使用，而且从目录结构上来说，octopress 模板和插件制作也更方便一点。
- 虽然 jekyll 比 octopress 粗糙，但是 jekyll 官网可比 octopress 逼格高多了，旧 jekyll 主页是一片苍茫白色背景中央，一个超简洁的黑白配色的英文名，大概是这种[感觉](https://olivermak.es/2015/10/jekyll-mix-blend-mode/)。也许，这就叫，less is more?

在实际使用中，jekyll 的这种限制太大，几乎很难作为自己的个性化定制的博客。所以，如果不是确信自己的博客很简单且不会有插件需求，那么并不推荐 host on fly 的 jekyll。

还记得之前所说的，jekyll 的目录中同样有类似 octopress 的 source folder 和 public folder 吗（[jekyll 目录结构](https://jekyllrb.com/docs/structure/)）？jekyll 和 octopress 一样，都有一个 build 命令，可以将 source 中的 md 文件转成 html 放到 public 文件夹下，只要把 public 文件夹的内容设为 master 分支，就可以将 jekyll 变成 octopress 了。（两种工作方式截然不同，所以文件夹目录结构和分支也有很大的变化，我当时改动的时候也很痛苦）

现在我在 source 分支下建立了一个 `_deploy.sh` 文件（ `_` 开头的文件不会被自动放到 public 下），这个命令其实就等于把 jekyll 变成 octopress 了。这种情况下，jekyll 不再 host md 文件，而和 octopress 一样 host html 文件了。当然这种情况下，你可以使用任何插件了，我就用到了 jekyll-assets, jekyll-sitemap 等插件，还自己修改了 kramdown 的 convert 函数。

```bash
#!/bin/sh
# 编辑 md 文件
git submodule foreach git pull # 你可以忽略这里，我把 md 文件放在另一个 repo 中了
jekyll b
cd _site # 等同于 octopress 的 public 文件夹
git add .
now="$(date +"%Y-%m-%d %H:%M")"
git commit -m "${now}"
git push origin master
```

当然 jekyll 的 host on fly 的模式并非一无是处，单页面的个人主页，项目主页就非常适合。

顺便说下，hexo 本质上是和 octopress 一样的，hexo 作者说 octopress 在文章数多的情况下转换速度太慢，所以自己用 nodejs 重新写了一个博客程序，也就是 hexo。事实上，如果把代码渲染这部份转移到前端，octopress 速度也没那么慢。

最后，写出高质量的文章，设计优秀的 UI，和你强相关，和博客程序几乎无关。