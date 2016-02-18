---
title: 批量取关 instagram 好友
date: 2016-02-18 23:28
categories: [技术]
tags: [instagram, javascript, unfollowgram]
---


前段时间由于帐号密码被撞库，instagram 被人盗去关注了好多广告帐号，大概有 1000 多个。登录进去一看吓了一跳，赶紧改了密码。可是怎么把这 1000 多号人取消关注，却犯了难。

首先我想借助官方的力量，可是并没有在 instagram 里找到人工服务的地方，也没有发现提交工单的表单；去 twitter @ 也没人理。要是 instagram 的 web 页面能够有 unfollow 的按钮，那倒还好办，异步发一堆 unfollow 请求就行了（被盗的 twitter 就是这样取消所有关注的，其实就是简单的`$(something).click()`），可惜没有。移动端不是很了解，不知道如何将一堆点击操作换成代码。也曾经想利用 instagram 的 API 自己写一个批量 unfollow 的脚本，然而 instagram 的安全等级太高，给出的权限很少，申请高级的权限还被拒了，所以这条路也走不通了。只好另寻他法了。

于是在 quora 的帮助下 找到了 [unfollowgram](http://unfollowgram.com/)。

进入 unfollowgram 主页，按照要求使用 instagram 的帐号登录之后，选择「who doesn’t follow me back」选项，然后执行如下代码，获取所有的将要 unfollow 的用户的 id。

``` javascript
var uids = $("#thelist li a.unfollow").map(function(val, i) {
    return $(i).attr('id');
}).toArray()
```

unfollowgram 没有提供批量 unfollow 的功能，接下来就要一个个的 unfollow 这些 users。

注意请求的频率，instagram 对每个 client 的限速是每小时 60 个 relationship 请求，但是 unfollowgram 似乎自己也有限制，每次超过 20 个就会出错，所以我改成每 3 分钟一个请求。代码如下。

``` javascript
var unfollow = function(uids) {
  uid = uids.shift();
  $.ajax({
    type: 'POST',
    url: 'http://unfollowgram.com/en/unfollow.ajax',
    data: 'uid=' + uid,
    async: false
  });
  if (uids.length > 0) {
    setTimeout(unfollow, 180000, uids);
  }
}
```

接下来执行`unfollow(uids)` 即可。1000 个请求，每 3 分钟执行一个，没办法，只能放那儿跑了，还得注意 unfollowgram 可能会被登出。

仍然不够完美，如果你有更好的方法，请务必告诉我。

