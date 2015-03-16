---
title: 为 Jekyll 博客添加微搜索
date: 2015-01-28 16:47
tags: [jekyll, tinysou, search]
categories: [技术]
---

 作为静态博客，搜索是个不太好做的功能。一般的处理方式是，在 Jekyll build 的时候，把页面信息写入一个文件中，然后在搜索的时候用 js 作匹配。客户端的浏览器会较为吃力。[微搜索](http://tinysou.com/) 是一个提供静态搜索的服务，功能强大，可定制性强，还可支持拼音搜索，非常适于作为博客这样开放，静态的内容的搜索引擎。

在官方[指南](http://doc.tinysou.com/guides/overview.html)里提供了较为详细的安装过程。简单来说就是：

1. 创建引擎
2. 添加域名
3. 获取 engine key，并安装到页面中

我在 Jekyll 博客的根目录下新建了一个`search.html`，其内容如下：

~~~html
---
layout: page
title: Search
permalink: /search/
---

<form><input type='text' id='ts-search-input'></form>
<div id='ts-results-container'></div>
<script>
var option = {
engineKey: '这里是你的 engine key',
resultContainingElement: '#ts-results-container',
renderStyle: 'inline'
};
(function(w,d,t,u,n,s,e){
s = d.createElement(t);
s.src = u;
s.async = 1;
w[n] = function(r){
w[n].opts = r;
};
e = d.getElementsByTagName(t)[0];
e.parentNode.insertBefore(s, e);
})(window,document,'script','//tinysou-cdn.b0.upaiyun.com/ts.js','_ts');
_ts(option);
</script>
~~~

我还没找到合适的展示方式和样式，所以现在先这样啦~得改好了样式再来更新好了。
