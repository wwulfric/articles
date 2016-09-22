---
title: Mac 下 pyenv 安装的 Python3 报 KeyError PYTHONPATH 错
date: 2016-09-22 22:37
categories: [技术]
tags: [miniconda, python, brew, path]
---

Error in sitecustomize; set PYTHONVERBOSE for traceback: KeyError: 'PYTHONPATH'

系统不知道更改了什么之后就遇到了这个问题，每当切换目录(`cd`)的时候都会跳出来错误提示，非常恼人。在 stackoverflow[^stackoverflow_links] 上没找到解决方法，倒是在一个日文[博客](http://qiita.com/Asakage/items/690ce9048e708de41166)里找到了。我也不懂日文，但是看他罗列的文件和代码，大概能了解他的思路，按照思路走，才发现了这个问题。

## 背景

我在 mac 系统上 使用 brew 安装了 Python2.7.10 替代了系统的 Python，之后又使用 pyenv 来管理 Python 的版本，pyenv 中安装了 miniconda 和 miniconda3，默认使用的是miniconda3，也就是 Python3。

## 挖坑

首先，`cd`会出现这个问题的原因是我在`.zshrc`中执行了`eval "$(pyenv init -)`，如果删除这一行，就不会有这个问题了。

其次，这个问题不仅在 cd 的时候会出现，执行 python 的时候也会。

```python
> python
Error in sitecustomize; set PYTHONVERBOSE for traceback:
KeyError: 'PYTHONPATH'
Python 3.5.2 |Continuum Analytics, Inc.| (default, Jul  2 2016, 17:52:12)
[GCC 4.2.1 Compatible Apple LLVM 4.2 (clang-425.0.28)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import sys
>>> sys.path
# 返回一个 list。着重关心其中一个元素：'/usr/local/lib/python2.7/site-packages'
```

这里其实就发现问题了，Python3.5 怎么会有 2.7 的 site-packages 呢？

`/usr/local/`是 brew 的默认安装路径，继续挖。

### brew doctor 看诊断

既然可能是 brew 安装的问题，执行`brew doctor`。

```shell
Warning: Your default Python does not recognize the Homebrew site-packages
directory as a special site-packages directory, which means that .pth
files will not be followed. This means you will not be able to import
some modules after installing them with Homebrew, like wxpython. To fix
this for the current user, you can run:
  mkdir -p /Users/username/.local/lib/python3.5/site-packages
  echo 'import site; site.addsitedir("/usr/local/lib/python2.7/site-packages")' >> /Users/username/.local/lib/python3.5/site-packages/homebrew.pth
```

这里就是问题所在！brew 在安装某些软件之后，可能需要用户手动做一些后续操作。想起来 brew 更新 Python 的时候，它曾经提示我这样做，于是就直接复制代码执行，结果就导致了这个问题。把homebrew.pth 的内容改成

```python
import site; site.addsitedir("/usr/local/lib/python3.5/site-packages")
```

我分析，其原因应该是 brew 没有安装 Python3，但是通过 pyenv 安装了 Python3，所以 brew 还是按照自己的意思建议你在homebrew.pth 导入 2.7 的site-packages。

[^stackoverflow_links]: [python - PYTHONPATH error when trying to activate a virtual environment - Stack Overflow](http://stackoverflow.com/questions/34981284/pythonpath-error-when-trying-to-activate-a-virtual-environment), [python - "KeyError: 'PYTHONPATH'" message when updating Anaconda packahes on Mac OS X - Stack Overflow](http://stackoverflow.com/questions/31601078/keyerror-pythonpath-message-when-updating-anaconda-packahes-on-mac-os-x), [python - After installing Anaconda, I get constant "KeyError: 'PYTHONPATH'" messages - Stack Overflow](http://stackoverflow.com/questions/32321973/after-installing-anaconda-i-get-constant-keyerror-pythonpath-messages)
