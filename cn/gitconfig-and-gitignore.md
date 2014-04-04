---
title: "Git 配置：gitconfig 和 gitignore"
date: 2014-04-04 11:32
tags: [git, gitignore, gitconfig] 
categories: [技术]
---

目的：更方便的使用 git。通过全局的 gitignore 和项目下的 gitignore，使得 git 仓库尽量整洁。

在家目录下创建 .gitconfig 文件，

``` bash
[user]
	name = 你的名字
	email = 你的邮箱
[core]
    editor = vim ;或其他编辑器
    excludesfile = 你的全局 ignore 文件地址（绝对地址）：.gitignore_global
[merge]
    tool = 你的 merge 工具，默认的是 vimdiff
[alias]
    ci = commit -a -v
    co = checkout
    st = status
    br = branch
    lg = log --graph --pretty=mt:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit --
    throw = reset --hard HEAD
[color]
    ui = true
```

配置好后，你就可以使用 `git ci`, `git co`, `git st` 等命令了。

在家目录下创建 .gitignore_global，这个文件中放的是全局的 git ignore 文件，比如编辑器的配置文件，缓存文件，编译的文件等，下为例子

``` bash
# Compiled source #
###################
*.com
*.class
*.dll
*.exe
*.o
*.so
*.pyc

# Packages #
############
# it's better to unpack these files and commit the raw source
# git has its own built in compression methods
*.7z
*.dmg
*.gz
*.iso
*.jar
*.rar
*.tar
*.zip

# Logs and databases #
######################
*.log
*.sql
*.sqlite

# OS generated files #
######################
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

# VIM swp files #
#####################
*.swp
# sublime project files #
#####################
*.sublime-project
```

在项目下的 .gitignore 文件应该放和项目紧密相关的 ignore 文件，比如项目的配置，数据库的配置等，如：

```bash
*/conf_set.php
database.yml
```
