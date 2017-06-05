---
title: 遥远分支的 cherry-pick 方法
date: 2017-06-05 20:07
categories: [技术]
tags: [git, cherry-pick, diff]
---

在刚开始使用 git 作为代码托管工具的时候，还是一名学生，自己的项目也没啥冲突好解决的，那个时候的 git 感觉就是一个用起来还算方便的轻量级代码同步工具。之所以说它轻量，是因为彼时所需要的所有命令只有 4 个：

```shell
git pull # 换电脑的时候同步代码

git add
git commit
git push # 推送代码
```

工作之后经历了团队开发，多进度开发，废弃，提前上线等等情况之后，才越来越感受到 git 所谓「分布式版本控制系统」的威力。

然而随着使用的更加频繁，渐渐的感觉到 git 也有一些不那么如人意的地方，比如时间久的大项目，占用空间大，不小心 commit 进去了大文件，就再也拿不出来了；checkout 分支的动作越来越重等。

这次遇到的就是一个较为棘手的合并问题。如下图所示，当开发一个feature 分支的时候，突然有需求要将该 feature 分支的 B commit 放到 master 分支上。而此时，由于疏忽，feature 分支已经远远落后于 master 分支了，且该分支也提交了大量的 commit，可想而知合并将产生大量的冲突。（当然，feature 分支应该尽量频繁的 rebase master）

![两个分道扬镳的分支之间的 cherry pick 手动 git 修复](http://wulfric.qiniudn.com/git/R-far-away-branch.png "两个分道扬镳的分支之间的 cherry pick 手动 git 修复")



面对这种冲突一点办法都没有，如果直接 cherry pick，大量的难以理解的冲突将令人无所是从，解决这些冲突也会消耗大量的精力，即使有些冲突你很明白并不需要。我们可以通过一些原始的手段预先减少一些冲突和打一部分补丁，以下是常用的方法：

- 查看两个 commit 之间的文件变动，对变更的内容了如指掌
- 将某个 commit 的某个文件的内容拷贝到当前
- 查看某个 commit 的某个文件的 diff
- 查看两个 commit 直接的某个文件的 patch，并将该 patch 应用到当前分支
- 如果确认当前可以使用其他分支的某个文件的最新版本，可以 checkout 到该 commit 的那个文件

下面的命令清单记录了解决冲突要用到的 git 命令。

```shell
# 查看两个 commit 之间哪些文件有变动
git diff --name-only SHA1 SHA2
# http://stackoverflow.com/questions/1552340/how-to-list-the-file-names-only-that-changed-between-two-commits

# 显示某个 commit 的某个文件的内容：拷贝内容
git show SHA1:path/to/file
# http://stackoverflow.com/questions/338436/is-there-a-quick-git-command-to-see-an-old-version-of-a-file

# 显示某个 commit 的某个文件的 diff
git show SHA1 path/to/file

# 查看两个 commit 之间某个文件的 patch
git diff <SHA1>^..<SHA2> -- <filename>

# 将该 patch 内容应用到当前分支：因为分支之间差距比较大，往往很难成功 patch
git diff <SHA1>^..<SHA2> -- <filename> | git apply
# http://stackoverflow.com/questions/16068186/how-to-cherry-pick-only-changes-for-only-one-file-not-the-whole-commit

# 完全使用远程分支的文件：新文件，确定使用另外一个版本
git checkout feature-branch /path/to/file
```