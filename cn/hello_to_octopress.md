title: 使用 Github Page 和 Octopress搭建个人博客
date: 2012-11-30 00:19
tags: [git, octopress] 
categories: [技术]
---

{% img http://www.lufeipic.tk/f/m/?.png 'Github and Octopress' 'Github and Octopress' %}

**Octopress Blog: 为极客准备的博客！**本文将简单介绍使用 github page 和 octopress搭建博客的几个步骤。

<!-- more -->

### 准备工作

假设你已经安装了如下软件：

* git
* ruby with RVM

#### 克隆 Octopress Code

``` bash
git clone git://github.com/imathis/octopress.git octopress
cd octopress    # If you use RVM, You'll be asked if you trust the .rvmrc file (say yes).
ruby --version  # Should report Ruby 1.9.3
bundle install  # Install what octopress need
rake install    # Install the default Octopress theme
```

执行 `rake preview` 预览默认的博客页面。[这里](localhost:4000).

#### 为 Github Page 做准备

创建一个新的 Github 仓库，用你的用户名 `username.github.com` 或组织名 `organization.github.com` 来命名。如下：   

{% img http://www.lufeipic.tk/f/k/?.jpg 'github page' 'github page' %}

执行 `rake setup_github_pages`，填入 github page 的 URL。 你将在工作目录中看到 `_deploy` 这个文件夹，它是用来部署到远端的静态文件。

#### Deploy and Push

部署到 github page by:

```bash
rake generate
rake deploy
# You can use `rake gen_deploy` instead of them
```

这会产生博客内容，并将产生的文件拷贝到`_deploy`文件夹，添加到 git，提交并 push 到 master 分支。不久之后你就会收到来自 Github 的邮件通知你博客内容部署成功。

**切记** 将你的代码 push 到 source 分支。保存你的文章和样式。 

```bash
git add .
git commit -m 'my first commit'
git push origin source
```
你将拥有两条分支：’master’ 分支用于部署，’source’ 分支用于编辑文章。

  
 
### 发表你的第一篇文章

你已经拥有了一个搭建在 github page 上的博客了。你一定注意到了默认的页面并非我们想要的，修改它。

运行 `vi _config.yaml` 编辑 `_config.yaml` 文件。

```
url: http://lufeihaidao.github.com
title: Wulfric's Blog
subtitle: FEEL FREE TO CHANGE THE WORLD!
author: Wulfric Wang
simple_search: https://google.com/search?q=
description:
```

#### 写第一篇文章

文章保存在 `source/_posts` 目录内，根据 Jekyll 的命名规则：YYYY-MM-DD-post-title.markdown。文章的名字将会用作 url ，日期将帮助文章区分和排序。

Octopress 提供了一个 rake task 帮助你新建符合命名规则的文章：

```
rake new_post['My first article']
```

你会在 `source/_posts/` 文件夹下找到一个 markdown 文件，修改它：

```
---
layout: post
title: "Personal Blog with Github Page and Octopress"
date: 2012-11-30 00:19
comments: true
categories: git 
---

Octopress Blog: Blog for geek!
```

添加一个或多个分类：

```
# One category
categories: Sass

# Multiple categories example 1
categories: [CSS3, Sass, Media Queries]

# Multiple categories example 2
categories:
- CSS3
- Sass
- Media Queries
```

写博文的时候，可以使用 HTML comment `<!--more-->` 来截断文章。只有前面部分可以显示在 index 中。

#### Push 到 Github

既然已经写好一篇文章了，那么就部署到远端吧！

```
# Do this to preview whether it works well or not
rake preview
# Deploy it to github page
rake gen_deploy
# Do these to commit source
git add . # add all useful files you want to commit
git commit -m 'My first article' # commit it locally
git push origin source # push source remotely
```

三个重要的文件夹：

* source: 源文件文件夹，主要的编辑工作在这里
* public: 本地预览
* _deploy: 远端部署

通常一个 workflow 是这样的：

1. 添加新文章
2. 编辑文章
3. 预览
4. Deploy
5. 提交 source 分支



### Deploy 到其他 PC

通常你会想在另外一台 PC 上继续你的博客大业：

``` bash
git clone git@github.com:lufeihaidao/lufeihaidao.github.com.git # clone your code
bundle install
git checkout source # switch to source branch
git clone git@github.com:lufeihaidao/lufeihaidao.github.com.git # clone your code
mv lufeihaidao.github.com/ _deploy # for gen_deploy
```

或者拷贝一份 octopress 代码，并执行 `rake setup_github_pages`。

然后按照 workflow 执行就好了。
