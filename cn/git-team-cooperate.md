---
title: "Git 团队协作实践"
date: 2014-03-25 14:55
tags: [git, 协作, rebase, merge] 
categories: [技术]
---


### 编辑

从 master 分支上新建分支，通过 `git checkout -b new-feature` 创建 `new-feature` 分支，开始工作

- 编辑
- 提交
- 重复上述步骤
    
完成之后，提交到仓库 `git push origin new-feature`
    
### Code Review

提交之后，发 pull request，拉同事来做 code review。Code review 完成之后，在本地执行 `git fetch`，从远端下载最新的更新。

### 合并与提交

开始合并工作---使用 rebase 而不是 merge：使用 rebase 将自己的更改提交合并成一个功能性的提交，这样在 master 分支的历史就会清晰明了，而 merge 会将每一个 commit 插入到历史里，增加了 review 的复杂度。

> TODO rebase 参考 [rebase](http://segmentfault.com/a/1190000000456077)

``` bash
git rebase -i origin/master
# 该命令会新建一个未命名的分支，在该分支上执行合并
# 执行后会跳到编辑页面，保留第一个 pick，其他 pick 改成 squash，保存退出
git status
# 执行下 git status，会看到分支已经更改
# 此时一般会出现冲突，执行 git status 查看冲突
# 编辑冲突
  # 编辑冲突完成，继续，循环上述操作直到没有冲突
  # 编辑完可能需要使用 git add -u 加入
  git add -u
  git rebase --continue
  # 如果遇到问题，abort 掉这次rebase，重新执行
  git rebase --abort
# 冲突编辑完毕后
# 复制 git log 的最新 commit id
git checkout master
git reset --hard origin/master #得到远端最新内容
git cherry-pick commit_id #该 commit id 就是刚才复制的内容
# 修改 commit 信息，使用简洁的描述，比如 fixed #99: user management
git commit --amend 
# 检查一下，提交更改
git push origin master
```

