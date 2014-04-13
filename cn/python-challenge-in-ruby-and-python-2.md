title: python challenge in ruby and python 13-23
date: 2013-03-18 22:32
tags: [ruby, python, pythonchallenge]
categories: [技术]
---

继续 python challenge 游戏。这篇文章将努力更新到 23 题。

## 第 13-17 题

剩下的题目官方的答案和网上的答案都越来越少了，果然能坚持下来的人不多啊，希望我能坚持下来吧。

### 第 13 题

数字 5 的地方可以点击，进去后告诉我们这个 xml 有问题。我对于 xml 不是很熟，搜索得知，需要用到 XML-RPC。在 python 文档里是这样介绍的：

> XML-RPC is a Remote Procedure Call method that uses XML passed via HTTP as a transport. With it, a client can call methods with parameters on a remote server (the server is named by a URI) and get back structured data. This module supports writing XML-RPC client code; it handles all the details of translating between conformable Python objects and XML on the wire.

提示：这里需要用到 12 题的 evil: Bert。打电话给它吧！

{% gist 10590975 "Q13" %}
