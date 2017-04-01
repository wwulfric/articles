遥远分支的 cherry-pick 方法

![复杂 cherry-pick](http://wulfric.qiniudn.com/git/R-far-away-branch.png "复杂 cherry-pick")



两个分道扬镳的分支之间的 cherry pick 手动 git 修复



```
查看两个 commit 之间哪些文件有变动
git diff --name-only SHA1 SHA2
http://stackoverflow.com/questions/1552340/how-to-list-the-file-names-only-that-changed-between-two-commits

显示某个 commit 的某个文件的内容：拷贝内容
git show SHA1:path/to/file
http://stackoverflow.com/questions/338436/is-there-a-quick-git-command-to-see-an-old-version-of-a-file

显示某个 commit 的某个文件的 diff
git show SHA1 path/to/file

查看两个 commit 之间某个文件的 patch
git diff <SHA1>^..<SHA2> -- <filename>

将该 patch 内容应用到当前分支：因为分支之间差距比较大，往往很难成功 patch
git diff <SHA1>^..<SHA2> -- <filename> | git apply
http://stackoverflow.com/questions/16068186/how-to-cherry-pick-only-changes-for-only-one-file-not-the-whole-commit

完全使用远程分支的文件：新文件，确定使用另外一个版本
gco yaitem_permission application/libraries/Permission_Web_Client.php
```