---
title: github pages 302 redirect 跳转
date: 2015-05-31 00:28 
tags: [github pages, 302 redirect]
categories: [技术]
---

使用静态博客，部署在 github pages，再绑定一个域名，是现在比较流行的博客撰写方案。github [官网](https://help.github.com/articles/tips-for-configuring-an-a-record-with-your-dns-provider/)和网上的教程都是在域名管理中把域名的 A 记录指向 github 给的 IP 地址。

这样设置已经基本上可以工作了。但如果你仔细观察 HTTP 的返回数据会发现，github 返回的是 302 跳转而不是 200，这样在做 SEO 的时候，Google 和百度都无法正确识别网站。参考这篇[文章](http://subosito.com/github-hosted-redirect/)。

解决方法是，换用可以支持 ALIAS 的 DNS 服务。注意这里的 ALIAS 不是 CNAME(ALIAS)，大多数域名服务不提供 ALIAS，所以需要好好选择，推荐的有[Pointhq](https://pointhq.com/)，支持一个免费域名；[DNSimple](https://dnsimple.com/)，ALIAS 需要收费。

![ALIAS 配置](http://qiniu-wulfric.lufeihaidao.top/conf.png "ALIAS 配置")

改完之后再看一下：

~~~bash
dig wulfric.me +nocomments +nocmd +nostats
~~~

~~~
; <<>> DiG 9.8.3-P1 <<>> wulfric.me +nocomments +nocmd +nostats
;; global options: +cmd
;wulfric.me.      IN A
wulfric.me.  3600 IN A xxx.xxx.xxx.xxx
~~~

这里的 IP 地址不再是 Github 给出的 IP 了

~~~bash
curl -I wulfric.me/archive/
~~~

~~~
HTTP/1.1 200 OK
Server: GitHub.com
Content-Type: text/html; charset=utf-8
Last-Modified: Fri, 17 Oct 2014 16:29:11 GMT
Expires: Sat, 18 Oct 2014 11:55:16 GMT
Cache-Control: max-age=600
Content-Length: 4786
Accept-Ranges: bytes
Date: Sat, 18 Oct 2014 11:45:16 GMT
Via: 1.1 varnish
Age: 0
Connection: keep-alive
X-Served-By: cache-lcy1125-LCY
X-Cache: MISS
X-Cache-Hits: 0
X-Timer: S1413632716.122644,VS0,VE85
Vary: Accept-Encoding
~~~

从返回来看，结果已经是正确的 200(OK) 了。Google Master 也能正确识别网站了。
