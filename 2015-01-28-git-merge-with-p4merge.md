---
title: 使用 P4Merge 作为 GIT 的可视化合并工具
date:  2015-01-28 23:43
tags: [p4merge, git, merge]
categories: [技术]
---

P4Merge 是一款非常优秀的 git merge 工具，且跨平台兼容。尽管 git 亦有内部实现的 merge 工具，但并不如 P4Merge 易用。我们可以通过配置`.gitconfig`文件来设置 git 使用外部 merge 工具。

![P4Merge](http://www.perforce.com/sites/default/files/p4merge_three_pane_1.jpg)

首先，下载安装 [P4Merge](http://www.perforce.com/product/components/perforce-visual-merge-and-diff-tools)。MAC 下可以通过`brew cask install p4merge`来安装。

在系统可访问的目录下（我们这里使用`/usr/local/bin/`）创建两个可执行文件`extMerge`和`extDiff`，其内容如下。

~~~bash
$ cat /usr/local/bin/extMerge
#!/bin/sh
/Applications/p4merge.app/Contents/MacOS/p4merge $*
~~~

~~~bash
$ cat /usr/local/bin/extDiff
#!/bin/sh
[ $# -eq 7 ] && /usr/local/bin/extMerge "$2" "$5"
~~~

别忘了添加可执行权限：

~~~bash
$ sudo chmod +x /usr/local/bin/extMerge
$ sudo chmod +x /usr/local/bin/extDiff
~~~

使用 `extMerge`和`extDiff`的好处是，我们可以很方便的切换 merge 工具。

最后，在你的`.gitconfig`文件里添加如下配置：

~~~bash
[merge]
    tool = extMerge
[mergetool "extMerge"]
    cmd = extMerge "$BASE" "$LOCAL" "$REMOTE" "$MERGED"
[diff]
    external = extDiff
[mergetool]
    trustExitCode = false
    keepTemporaries = false
    keepBackup = false
    prompt = false
~~~

好了，当合并（merge/rebase） 出现冲突时，执行`git mergetool`，即可调出 P4Merge 来解决冲突了。唯一的缺点是，目前还不支持 Retina 屏，看起来有些糊。(PS：可以用工具[Retinizer](http://retinizer.mikelpr.com/)来解决这一问题)

更多细节参见官网[链接](http://git-scm.com/book/en/v2/Customizing-Git-Git-Configuration#External-Merge-and-Diff-Tools)。
