---
title: "gitcafe 与 车库咖啡"
date: 2014-04-06 21:57
tags: [git, 咖啡]
categories: [扯淡, 产品]
---

> 本文是在 gitcafe 实习时的一些想法。

我认为，gitcafe 除了做代码托管外，还可以利用品牌名称的优势，做线下的实体咖啡店。主要面向学生、创业者和自由职业者，定位则类似于“车库咖啡”。

首先，需以 gitcafe 的代码托管业务为中心，整合出一套云端开发范例（由 gitcafe 提供电脑）。我以为此开发范例应该包括如下几个方面：

* 本地开发环境云端化，包括编辑器，编译环境，浏览器及其调试用插件等。对于编译环境、浏览器及其调试用插件等，可能的解决方案是 vagrant 虚拟机的自动部署。Ubuntu 能够记录系统安装的软件，将这个记录存在 gitcafe 上就可以在开机的时候自动同步到个人编译环境。对于编辑器，vim 可以通过一个 .vimrc 文件同步所有的配置和插件，想来 emacs 和 sublime 应该也可以。
* 代码托管：自然就是 gitcafe 了。
* 其他。比如数据的同步（云端数据库，不知现在市面上有没有比较有名的云端数据库，小开发者可以将数据的安全托管在云端数据库上）。

开发环境的初始化在安装各种软件的时候会比较耗时，这个时候 gitcafe 线下实体店的作用就体现出来了：一杯咖啡的时间完成整个系统的初始化，工作不要太惬意～连广告我都想好了：

> 一人坐在咖啡厅内，镜头切在咖啡杯内，重点表现咖啡加牛奶后黑白颜色的交融与变化，旁白：时间瞬息万变，杯中也是，街头也是，而我又会存留在多少人的记忆里？
> 
> 镜头从咖啡转到街景，再转向一个象征美好未来的景（比如日出的阳光打在街道上），然后出 logo 和标语： Gitcafe，找到你的精彩未来。

可能的合作单位：

* PC 厂商。我觉得要是整个 gitcafe 提供的电脑都是 Mac 那简直太酷了，但不知苹果是否愿意做这样的合作，Mac 是否足够的开放能允许这种定制。如果 gitcafe 里全是一种电脑并且这种实体店能打出名声的话，应该会有 PC 厂商愿意赞助的。
* IaaS 服务提供商 / Paas 服务提供商 / 其他云端产品服务提供商

可能的盈利方向：

* 实体店本身就是对 gitcafe 各种在线服务的最好的广告
* 注册会员的月费
* 咖啡费（可能包括在上一条中）

