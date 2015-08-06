---
title: MAC 开发五大效率工具
date: 2015-07-24 14:58:14
categories: [技术]
tags: [MAC, 工具, brew, iterm2, mosh, zsh, git]
---

## 全局工具

包管理工具和软件安装工具 brew, brew cask。

[Homebrew](http://brew.sh/)是MAC下的包管理工具，可以当做`debian`下的`apt-get`，但要强大得多。按照官方的介绍：

- Homebrew installs [the stuff you need](https://github.com/Homebrew/homebrew/tree/master/Library/Formula) that Apple didn’t. 是MAC OS生态的重要补充，提供了大量在Linux下喜闻乐见的工具，比如`curl`, `rename`, `wget`等等。
- Homebrew installs packages to their own directory and then symlinks their files into `/usr/local`. 通过brew安装的软件可以和系统原软件完美兼容，互不影响。
- Homebrew won’t install files outside its prefix, and you can place a Homebrew installation wherever you like.
- Trivially create your own Homebrew packages. 方便的创建个人包。
- It's all git and ruby underneath, so hack away with the knowledge that you can easily revert your modifications and merge upstream updates. 通过git管理所有安装配置，极易创建'私服’，把官方repo作为上游分支，可以自由定制。

``` ruby
# brew edit wget 可以直接编辑wget的安装配置
class Wget < Formula
  homepage "https://www.gnu.org/software/wget/"
  url "https://ftp.gnu.org/gnu/wget/wget-1.15.tar.gz"
  sha256 "52126be8cf1bddd7536886e74c053ad7d0ed2aa89b4b630f76785bac21695fcd"

  def install
    system "./configure", "--prefix=#{prefix}"
    system "make", "install"
  end
end
```

[caskroom](http://caskroom.io/)(brew cask)构建于Homebrew，继承了其优雅和便捷性，提供安装 OS X 应用和二进制软件的新方法。可以这么简单理解，brew安装的是命令行工具，比如`curl`, `rename`, `wget`；brew cask安装的是应用软件，比如`google chrome`, `dropbox`等。通过brew cask 安装软件，只需要简单的一条命令`brew cask install google-chrome`即可，再也不需要以前的打开网页、找到链接、下载软件、解压包、放到程序目录，删除安装文件，再来启动它这么复杂的步骤了。一键完成！[^1]

虽然通过 Mac App Store 安装软件一样很简单，但是 Mac App Store生态圈远不完善，审核流程过长，限制太多，维护成本过高让很多应用开发者被迫离开。大量软件缺失。[^2]

你也可以像 brew 那样自由定制，假如你觉得某个软件全球的用户都需要，就可以向官方 repo 发起 pull request。

更有趣的是，你的重装系统将变得极为简单！把你的常用软件列表记下来，然后执行`brew cask install software-1 software-2 ... software-n`，然后睡一觉就一切搞定了~个人文档数据通过`dropbox`直接恢复~你的代码肯定都在`github`~

[Alfred](http://www.alfredapp.com/) [mac talk 的教程](http://macshuo.com/?p=625) [教程2](http://wellsnake.com/jekyll/update/2014/06/15/001/) 

[dash](https://kapeli.com/dash) 

文档集合工具

支持海量语言和插件

几乎支持每一个编辑器和IDE

和Alfred配合

## 终端工具

[iterm2](https://iterm2.com/)

[mosh](https://mosh.mit.edu/)，更好的 ssh 工具，更健壮，支持断续连接，支持除了 iPhone 之外的几乎任何平台。（iOS 让人爱不释手的优点，也正是它让人恨之入骨的缺点）

![mosh](http://wulfric.qiniudn.com/R-mosh.png "mosh")

从上面来看，mosh 最主要的优点就是，断网了，休眠了，mosh的连接不会断。

## 命令行工具

zsh, oh-my-zsh, j, d, ag, [htop](http://hisham.hm/htop/), [ccat](https://github.com/jingweno/ccat)

## git 工具

git [官网](https://git-scm.com/download/gui/linux)收集了几乎所有市面上的图像界面客户端。

[SourceTree](https://www.sourcetreeapp.com/)。在各种[测评](http://www.slant.co/topics/465/~what-are-the-best-git-clients-for-mac-os-x)中，SourceTree一般被评价为MAC下性价比最高的。

p4merge 也是[测评](http://www.slant.co/topics/48/~what-are-the-best-visual-merge-tools-for-git)中可视化合并工具中性价比最高的。可以将 p4merge 配置为 git 的默认可视化合并工具，参见[使用 P4Merge 作为 GIT 的可视化合并工具](http://wulfric.me/2015/01/git-merge-with-p4merge/)。

PS: p4merge 并没有针对 Retina 屏优化，请使用 [Retinizer](http://retinizer.mikelpr.com/) 工具修正一下。

[tig](https://github.com/jonas/tig)（git cli 的工具，效果和sourcetree很像，适用范围不同，不需要记 git log, git diff 等命令）

## 虚拟化工具

vagrant

## vim 插件等—或者叫编辑器更好

可以先介绍 Vim 的一些特别优秀的插件，然后介绍较为出名的配置，然后是emacs的较为出名的配置。

vundle, nerdtree, ctrlp,fugitive, git gutter, ctrlsf

vim-surround, vim-repeat

text object, kill ring, undo tree

## 其他可能有用的开发工具

boris, pry, ipython 分别是 PHP, ruby, python 下 **更好** 的 REPL，知道任何一个便知道另外两个的用处。需要一提的是，ipython 的功能则要强大的多得多。

[vdebug](https://github.com/joonty/vdebug) 是 Vim 下的 PHP 调试工具，在 xdebug 外包裹了一层壳，更加易用。其功能如下所示。

- F5: start/run (to next breakpoint/end of script)
- F2: step over
- F3: step into
- F4: step out
- F6: stop debugging
- F7: detach script from debugger
- F9: run to cursor
- F10: toggle line breakpoint
- F11: show context variables (e.g. after "eval")
- F12: evaluate variable under cursor
- :Breakpoint [type] [args]: set a breakpoint of any type (see :help VdebugBreakpoints)
- :VdebugEval [code]: evaluate some code and display the result
- [Leader]e: evaluate the expression under visual highlight and display the result

chrome vimium http://sspai.com/27723
滚动，查看所有标签，gg G，鼠标中键，shift f, 上一页下一页


postman, rest client http调试工具，缺乏前端接口的情况下调试后端输出是否正确。

pyenv, rbenv, rvm, nvm 语言版本管理


ruby 的工具
vagarnt，homebrew, 三个版本管理工具 rbenv,rvm,ruby-lang,部署工具 mina, capistrono,服务器 puma,passenger,unicorn,thin, 前端模板 haml, slim. coffeescript. 


[^1]: http://www.yangzhiping.com/tech/homebrew-cask.html
[^2]: http://ksmx.me/homebrew-cask-cli-workflow-to-install-mac-applications/
