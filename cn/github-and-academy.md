﻿---
title: "学术型 github 畅想"
date: 2014-04-06 22:04
tags: [git, github, SNS, 学术, 写作, 版本控制]
categories: [扯淡, 产品]
---

> 本文是在 gitcafe 实习时的一些想法。

#### 什么是 github？

Github 是编程项目的托管服务，在这里，既有老鸟的牛逼项目，也有新手的实验性项目。表面上，github 是项目的集合，本质上则是代码的集合。同时 github 也有社交功能，你可以关注某人/组织，也可以关注某个项目，但本质上，github 还是以代码为中心的。

#### 学术论文撰写现状

现在学术论文的写作其实有很多需要改进的地方：

* 文章多是孤立分散的，虽然多数论文数据库都提供推荐功能，但推荐文章之间并不一定具有很强的联系性（比如，当跟进某课题组的项目时，需要查找所有该项目的文献，但推荐系统一般无法达成这一目的）。
* 文章发表之后才能看得到，看到文章的时候文章的撰写已经完成，不利于其他人参与。
* 很多数据库的论文的投稿和下载都需要收取不菲的费用，不利于知识共享。
* 文章无法为作者带来直接的利益。文章往往只会给作者带来间接的利益，但其实，文章也应该可以为作者带来直接的利益。
* 不同作者撰写文章的方法千差万别，互相之间交流困难。对于没有 MS Office 的读者，所见即所得的 MS Office 需要费时费力安装软件；对于不熟悉 tex 的读者，tex 又很难阅读。同时，虽然 MS Office 已有历史记录的功能，但这只是简化版的版本控制，无法进行比较、自由回溯等操作。
* 投稿不能做到格式无关。一篇文章的本质应该是由章节、公式、配图和参考文献等语义化组件组织起来的，不应该包含格式，但投稿的期刊、杂志、会议等都有不同的格式要求，增加了作者不必要的工作，甚至有写文章 20% 改格式 80% 时间的说法。
* 现有文献管理方式混乱而低效。现有的文献搜集方式是搜索-下载-归类放置，高级一点的则使用 endnote/jabref 等软件来管理。但这种方式需要维护所有的文献，万一一不小心删除了某篇文章找起来就会很麻烦，找到之后还要重复执行下载-归类的操作。

#### 学术型 github 可以是什么样的呢

我们能不能更酷的组织文章的写作与管理呢？不妨看看学术型 github 可以是什么样子的。

学术型 github 本质上将会是文章的集合，它同样以项目为组织形式。同时也具有社交元素，你可以关注感兴趣的人、组织和机构。这样一来，文章不再是孤立的一篇一篇，而是属于某个项目中，方便了查找与阅读。除此之外，还可以参考 github 的 watch/关注、fork/派生、Pull Request/合并请求、issues/工单，方便其他人参与到项目中来。

秉持开放与自由的哲学。如同软件开发曾经由大公司主导一样，学术的学院派气息很重，普通人难以负担论文下载的费用，参与到学术中来困难重重。如果学术界也能有一股如同编程领域内开源社区的清风，取消投稿与下载收费，另谋商业模式，想来应该是一件功在千秋之事。另外，可以仿效开源软件的协议（license），定义文章的再使用权限。作者可以借助较为严格的商业协议获得直接利益。

撰写论文使用 markdown + pandoc。其优点为：

- 版本控制，回溯历史更加方便；
- 纯文本，便于比较不同版本的差别；
- 便于交流。这种书写方式可以实现语义与格式的分离，将格式和布局完全交给第三方。如同网页文件由 html 和 css 实现语义和布局的分离一样；
- markdown + pandoc 可以自由的转换为 tex 和 word。

传统文献管理是管理文章，维护的也是所有的文章，引用的时候也需要指出很多其实并不重要的内容（比如第几卷第几页，这在以前很重要，但在数字时代，并不需要根据这些信息来唯一的标识一篇文章）。学术型 github 的文献管理方式则是维护 ID 列表（比如 URL），每个 ID 唯一标识一篇文章，大大方便维护和管理，引用也更加方便。同时还易于添加社交属性，具有相同兴趣的研究者可以共享文献列表。

#### 意义

一直以来，科学都是学院派的地盘，民科因缺乏完善的基础知识和异想天开一直被广为嘲讽。但在当今互联网大潮下，普通人对很多行业的参与度越来越高，各行各业无不经历或即将经历颠覆的浪潮。今天，名校公开课大行其道。也许，在能够自由获取文献和提交文章的未来，会是民间科学家的弄潮时代？既然传统的出版业已经渐渐萎缩，那么现在的那些牛气冲天的期刊杂志，会不会也有这样的一天呢？