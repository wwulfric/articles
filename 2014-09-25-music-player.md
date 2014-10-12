---
title: Linux PC 网易云音乐下载方案
date:  2014-09-25 23：53
tags: [虾米, 网易, 落网, foobar, iTunes, btsync]
categories: [技术, 音乐]
---

这篇文章主要是关于我常用的音乐资源和播放器，以及怎样在网易云音乐没有提供客户端的终端获取资源的方法。

以前用 Windows PC 的时候，最喜欢的音乐组合是[虾米](http://www.xiami.com/)的资源和 [Foobar](http://www.foobar2000.org/) 播放器。虾米资源出众，Foobar 播放器无出其右。后来网易开始做云音乐，才渐渐把资源库转到网易。[网易云音乐](music.163.com)和虾米相比，目前的劣势还是内容太少，很多稍微冷门的资源在虾米都能有好几页的评论，在网易云音乐确只有寥寥几条评论，甚至有时候连资源都没有；另外，网易云音乐的标签也不够完善。当然，似乎也很少有音乐资源站会提供包括作词、作曲、流派等标签齐全的资源。优点嘛，网易云音乐的体验更好，下载更方便，版权问题似乎稍微少一点。所以我现在找资源都是二者结合。

前段时间切换到 Mac OS了，当时网易云音乐还没有出 Mac OS 的客户端，所以网易云音乐的下载就成了问题（现在 Mac OS 已经有[客户端](http://music.163.com/#/download)，可以直接下载，但 Linux 的客户端依然缺失，而且在可预见的将来应该会一直缺失……）。我找到的解决方案是，在手机端下载音乐，然后利用 [BitTorrent Sync](http://www.bittorrent.com/sync) 把音乐同步到 Mac。

> 1. 手机端下载安装 BitTorrent Sync：[安卓](https://play.google.com/store/apps/details?id=com.bittorrent.sync)，[IOS](https://itunes.apple.com/us/app/bittorrent-sync/id665156116?mt=8)；
> 2. 在手机端，将网易云音乐的下载目录`netease/cloudmusic/Music`添加到 BitTorrent Sync 的同步目录；
> 3. 在 Mac 端安装 BitTorrent Sync：`brew cask install bittorrent-sync`，把手机端的目录同步到本地目录；
> 4. 添加到本地音乐库

OK，完工啦。

PS: 强烈推荐一个优秀的音乐发现网站：[落网](http://www.luoo.net/)
