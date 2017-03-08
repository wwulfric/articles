---
title: Mac OS 使用 jenv 管理 java 版本
date: 2017-03-07 21:05
categories: [技术]
tags: [brew, jenv, rbenv, java]
---

[jenv](http://www.jenv.be/) 是跨平台的 java 版本管理工具。当然，pyenv 仿的 rbenv，jenv 也是仿的 rbenv，功能和用法也很类似。

```shell
$ brew install jenv
# 添加 path
$ echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.zshrc
$ echo 'eval "$(jenv init -)"' >> ~/.zshrc

# 安装成功，添加 java 版本(需自行下载安装)
$ brew tap caskroom/versions
$ brew cask install java7
$ brew cask install java8
# 需要注意的是，这里仅仅安装了 java 的 pkg 文件，你还需要进入对应的目录，执行这个 pkg 文件来完成安装。
$ cd /usr/local/Caskroom/java7/1.7.xxx
$ open xxx.pkg

# 将安装好的 java 添加到 jenv，注意路径和版本可能稍有不同
$ jenv add /Library/Java/JavaVirtualMachines/jdk1.8.0_66.jdk/Contents/Home/
$ jenv add /Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home/
```

安装好后，大部分的使用方法都和 rbenv/pyenv 类似，无非就是 /versions/local/global 等，当然还有一些特殊的配置，比如 java 的 options：

```shell
# 使用 1.7 版本
$ jenv local 1.7
# 设置编译参数选项
$ jenv local-options "-Xmx512m"
# 查看 所使用的 java 的信息
$ jenv info java
```

查看版本是否更改成功：

```shell
$ java -version
java version "1.8.0_66"
Java(TM) SE Runtime Environment (build 1.8.0_66-b17)
Java HotSpot(TM) 64-Bit Server VM (build 25.66-b17, mixed mode)

$ jenv local 1.7

$ jenv info java
Jenv will exec : /Users/xxx/.jenv/versions/1.7/bin/java
Exported variables :
  JAVA_HOME=/Users/xxx/.jenv/versions/1.7

$ java -version
java version "1.7.0_80"
Java(TM) SE Runtime Environment (build 1.7.0_80-b15)
Java HotSpot(TM) 64-Bit Server VM (build 24.80-b11, mixed mode)

$ jenv which java
/Users/xxx/.jenv/versions/1.7/bin/java

$ jenv enable-plugin maven
jenv: no such command `enable-plugin'
```

我们发现虽然 jenv 的 version 对了，但是 java -version 的结果还是不对，而且尝试开启 maven 插件也出错。执行`jenv doctor`查看原因：

```shell
$ jenv doctor
[OK]   	No JAVA_HOME set
[ERROR]	Java binary in path is not in the jenv shims.
[ERROR]	Please check your path, or try using /path/to/java/home is not a valid path to java installation.
       	PATH : ...
[ERROR]	Jenv is not loaded in your zsh
[ERROR]	To fix :       	cat eval "$(jenv init -)" >> /Users/xxx/.zshrc
```

原来是因为终端开了多个标签页，在另一个标签页编辑完 .zshrc 文件后直接到这个标签页执行了，应该先 source 一下：`source ~/.zshrc`。

```shell
$ jenv enable-plugin maven
maven plugin activated

$ jenv disable-plugin maven
maven disabled
```

成功开启。需要注意，插件的支持是全局的，和 local/shell 无关，只需要开启一次就行了。jenv 的所有插件可以查看[列表](https://github.com/gcuisinier/jenv#plugins)。