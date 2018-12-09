---
title: python 数学绘图工具 matplotlib 的优化配置
date: 2015-10-19 23:37
tags: [ipython, matplotlib, matlab, retina, 中文, ggplot]
categories: [技术]
---

matplotlib 是 python 下的 2D 数学绘图工具，仿 matlab 编写而成，功能强大，是 python 数值计算库中非常重要的一员。安装很简单，`pip install matplotlib` 即可。如果安装时遇到什么问题，一般是依赖没有安装完全，按照错误提示一路安装过去便是，参照 [Installing — Matplotlib](http://matplotlib.org/users/installing.html)。强烈建议同时安装 ipython: `pip install "ipython[notebook]"`。ipython 的依赖关系较多，请耐心查看[文档](http://ipython.readthedocs.org/en/stable/)，所依赖的一些非 python 的程序可以通过系统的包管理工具安装，比如 `brew install zeromq`。

![ipython xkcd](http://jakevdp.github.com/figures/xkcd_version.png "ipython xkcd")

安装完成之后，先在 python 下执行如下命令，这个命令第一次调用的时候会生成 cache 文件，速度较慢，生成之后就不会出现卡顿的情况了。

``` python
import matplotlib.pyplot as plt
```

在 bash 中查看生成的 cache：

``` bash
> ls .matplotlib
fontList.cache matplotlibrc   tex.cache
```

 matplotlib 默认的颜色配置不好看。为了使 matlab 用户易于上手，matplotlib 的默认配色采用了与之相同的配色方案。这种对比明显的配色方案在出版物上观看时效果很好，但是并不适于在屏幕上观看[^1]。

一位女程序员 [olgabot (Olga Botvinnik)](https://github.com/olgabot) 表示不能忍，于是写了一个库改善 matplotlib 的配色: [olgabot/prettyplotlib](https://github.com/olgabot/prettyplotlib)。我颤抖着进入她的主页，然后跪着看完了她的 CV: MIT 本科双学位（数学和生物），生物信息学博士，玩的了设计，写的了代码，还能做俄-英的医疗口译……好吧，我们还是继续说配色的事情。对于 python 和 R 语言在数值计算领域孰优孰劣的争论中，其中一种观点就是，matplotlib 的配色和 ggplot 相比，太哔丑了。

![](http://qiniu-wulfric.lufeihaidao.top/shuo-de-hao-you-dao-li.jpg)

好嘛好嘛，学习别人的优点就好啦。matplotlib 提供对其配色等的个性化配置，参见[官方文档](http://matplotlib.org/users/customizing.html)，注意系统不同，配置文件的位置也不同。这篇[文章](https://www.huyng.com/posts/sane-color-scheme-for-matplotlib)中就给出了模仿 ggplot 配色的 [gist](https://gist.github.com/huyng/816622)。还有这个库 [daler/matplotlibrc](https://github.com/daler/matplotlibrc)，给出了几个不同的配色。按照官方文档的提示，把喜欢的配色文件的内容复制到你的 matplotlibrc 文件中即可。（配色问题有所**更新**，现在你不再需要自己修改配置文件了，官方已经内置了优雅的配色，请查看到最后）

最好的使用 matplotlib 的环境是 `ipython notebook`，这是一个类似于 Matlab 交互式编程的界面。在 bash 中执行 `ipython notebook`，会在后台启动一个本地 tornado 服务，之后便可在 `http://localhost:8888/notebooks` 中使用 notebook。效果可以查看 nbviewer 的任何一个链接，比如[这个](http://nbviewer.ipython.org/url/jakevdp.github.com/downloads/notebooks/XKCD_plots.ipynb)。

ipython 移除了 pylab 启动参数，所以现在你不能通过 `ipython notebook --pylab=inline` 来获得内联查看画图的功能。需要在启动之后执行 `%matplotlib inline`。另外，为了使画出来的图支持 retina，你还需要执行 `%config InlineBackend.figure_format='retina'`，你可以把下面的代码作为你 .ipynb 文件的初始脚本。更多配置可参考这篇[文章](http://blog.invibe.net/posts/2015-01-07-the-right-imports-in-a-notebook.html)。

``` python
import numpy as np
import matplotlib.pyplot as plt
import matplotlib
%config InlineBackend.figure_format='svg'
#config InlineBackend.figure_format='retina'
%matplotlib inline
```

matplotlib 默认不支持中文显示，查看[这里](http://hyry.dip.jp/tech/book/page/scipy/matplotlib_fast_plot.html#id6)的设置。

PS1: 配图为[XKCD](https://zh.wikipedia.org/zh/Xkcd)[^2]。

PS2: 1.5 版本的 matplotlib 已经内置了 ggplot 配色，参见[文档](http://matplotlib.org/users/style_sheets.html)，[示例](http://matplotlib.org/examples/style_sheets/plot_ggplot.html)。

```python
import matplotlib.pyplot as plt
print plt.style.available
# [u'dark_background', u'bmh', u'grayscale', u'ggplot', u'fivethirtyeight']
plt.style.use('ggplot')
```

![ggplot](http://matplotlib.org/_images/plot_ggplot.png)

2.0 版本的 matplotlib 已经要抛弃传统 matlab 配色了，改用「viridis」配色，参见[文档](http://matplotlib.org/style_changes.html)，不妨看下那个视频，蛮有趣的，他们是怎么做决定使用哪个内置配色的~顺便到[这里](http://bids.github.io/colormap/)看看新增配色。

[^1]: [Sane color scheme for Matplotlib](http://www.huyng.com/posts/sane-color-scheme-for-matplotlib/)

[^2]: [XKCD-style plots in Matplotlib](http://jakevdp.github.io/blog/2012/10/07/xkcd-style-plots-in-matplotlib/)
