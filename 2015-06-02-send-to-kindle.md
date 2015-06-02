---
title:  个人电子书自动发送到 kindle
date: 2015-06-02 11:45
categories: [技术]
tags: [ifttt, kindle, dropbox, gmail, Klipme]
---

kindle 有个`send to kindle`服务，可以把个人文档发送到 kindle 的服务器上。kindle 的内容包括：电子书（从亚马逊购入的书）、个人文档（个人电子书，通过`send to kindle`服务得到）、字典(在任何终端和 APP 里下载的字典)。

![我的内容](http://wulfric.qiniudn.com/R-my_content.png "我的内容")

但亚马逊中国的`send to kindle`服务一直无法正常使用，我们只好另辟蹊径了 -_-! 参看[知乎讨论](http://www.zhihu.com/question/21174855)。

 一种方法是，通过 chrome 插件`Klip.me`，另一种是手动给亚马逊 kindle 的对应邮箱发送文件。其原理都是，从 A 邮箱（用户个人邮箱，安全起见，需要在亚马逊官网填写注册这个邮箱，否则无法发送成功）往 B 邮箱（kindle 绑定邮箱，每个 kindle 设备和 kindle APP 都对应一个邮箱）发送个人文档，该个人文档就会被发送到 B 邮箱对应的 kindle 设备或 APP。举例来说，`Klip.me`的原理就是，通过`kindle@klip.me`A 邮箱发送文档到`%s_%d@kindle.cn`B 邮箱。

![注册邮箱](http://wulfric.qiniudn.com/R-content_settings.png "注册邮箱")

![认证邮箱](http://wulfric.qiniudn.com/R-auth_email.png "认证邮箱")

我们还可以利用`ifttt`将这一过程自动化。 登录`ifttt`，激活`dropbox`和`gmail`，给`dropbox`设置 trigger。当`dropbox`的某个文件夹有文件添加时，触发`ifttt`。 

![ifttt-trigger](http://wulfric.qiniudn.com/R-ifttt_dropbox_gmail_triger.png "ifttt trigger")

`ifttt` 触发后，执行`gmail`的 action，将该文件通过`gmail`邮箱发送到 kindle 邮箱。

![ifttt-action](http://wulfric.qiniudn.com/R-ifttt_dropbox_gmail_action.png "ifttt action")

稍等片刻，该文档就会发送到 kindle 邮箱对应的设备上了。

注意：`ifttt`需要较高的权限，请自己判断是否愿意把`dropbox`和`gmail`的权限交给`ifttt`。`dropbox`似乎会经常回收权限，如果发送失败，请查看是否需要重新激活。至于`gmail`，可以注册一个全新的帐号，仅作发送文档之用。
