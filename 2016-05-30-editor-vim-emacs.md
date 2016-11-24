---
title: Vim、Emacs 和 IDE
date: 2016-05-30 15:04
categories: [技术]
tags: [vim, emacs, ide, spf13, kvim, fisa, prelude, purcell, spacemacs]
---

这篇文章想要介绍一下 vim 和 emacs 两个编辑器。包括怎样组装插件打造成近似 IDE 的功能，使用场合，优点和缺点等。本文并不致力于手把手教你将 vim/emacs 打造成 IDE，也不会详细的介绍某些具体的功能和插件。

![vim and emacs](http://wulfric.qiniudn.com/vim-and-emacs.png "vim and emacs")

## vim 和 emacs 的优缺点

vim 和 emacs 分别被称为「 编辑器之神」与「 神之编辑器」，自有其独到之处。但是陡峭的学习曲线却吓跑了好多潜在用户。那么我们是否还有必要学习这两个编辑器呢？

![editor-learning-curve](http://wulfric.qiniudn.com/editor-learning-curve.jpg "编辑器学习曲线")

### 快捷键的无差别延续

vim 和 emacs 诞生于 30 年前，快捷键基本没什么变化。这意味着，一旦你学会使用这两个编辑器，无论以后软件怎么更新，都不需要学习别的快捷键了。因为历史较长，加上快捷键变化不大，新兴编辑器大多提供模拟 vim/emacs 操作的插件。这也方便了用户迁移到其他编辑器，无需学习更多的同质快捷键。

### 简单和适用的默认配置

vim 和 emacs 都可以运行在终端，也有图形化的软件，非常适合快速编辑文件。当需要在无法运行图形界面的服务器上编辑代码的时候，二者也足以胜任。虽然在终端也有 nano 这样的编辑器，但毕竟 too simple ⚯，无法方便的完成较为复杂的编辑工作。而且，这两个编辑器的默认配置的功能就已经很强大了，语法着色、补全、缩进等功能也表现不俗。

### 上手难度大默认配置不好看

虽然默认的功能很强大，但不得不说，默认的配色真是难看，相比于 sublime text 和 atom 这样开箱即用又美观的编辑器（sublime text 的默认配色在其他编辑器里也很流行，可见一斑），这等于直接拒绝了一批颜控。

为了实现强大的功能，vim 选择了多模式编辑（Normal, Insert, Visual 模式），emacs 则选择了复杂的[快捷键](https://www.gnu.org/software/emacs/refcards/pdf/refcard.pdf)。这些因素导致了这两个编辑器学习曲线陡峭，使用体验不够友好。对于一个刚上手的 vim 用户，他的内心一共有三个疑问：为什么 vim 只帮助乌干达的可怜儿童？怎么输入？怎么关掉？相比而言，一个刚上手 emacs 的用户心中的疑问就比较少：好了，我试着敲了一些字母了，现在，该怎么关掉？

![看到这样的快捷键，我的内心是拒绝的](http://wulfric.qiniudn.com/emacs-bad-shortcut.png "看到这样的快捷键，我的内心是拒绝的")

### vim 和 emacs 的区别

无论是日常感知还是做一些简单的[调查](http://www.google.com/trends/explore#q=Emacs%2C%20Vim&cmpt=q&tz=)，大概都能得出 vim 用户多于 emacs 用户的结论。而且，对大部分 Linux 发行版来说，vim 都是内置的，emacs 则不是。也就是说，某种程度上，vim 比 emacs 更容易被接受。

vim 的基础快捷键非常简洁，比如移动的`hjklweb`，删除的`dx`，复制粘贴相关的`yp`，配合 vim 独有的 text object 属性（i 表示 in，a 表示 around），可以组合出非常强大的快捷操作。比如：

|    组合键    |               快捷操作               |
| :-------: | :------------------------------: |
|    5j     |             向下移动 5 行             |
|    5w     |            向后移动 5 个单词            |
|    5dd    |              删除 5 行              |
|    di"    |   删除"包裹中的字符串， "word" 会删除 word    |
|    da"    | 删除"包裹和包裹中的字符串， "word" 会删除 "word" |
|    vi"    |        选中，"word" 会选中 word        |
|    va"    |       选中，"word" 会选中 "word"       |
| yi" 和 ya" |               你猜？                |

大家在做`git rebase` 的时候经常遇到下面这种情况吧，在 vim 下，把第二到第十行的 pick 改成 squash 就非常简单：`:2,10s/pick/squash`。

``` bash
pick 4aad920 f
pick 322877a f
pick 34dacff f
pick f1c9311 f
pick 8cca93d f
pick 497c2c9 f
pick b2d6921 f
pick cafe1d4 f
pick 9b70e6c f
pick 5c3747b f

# ...
```

除此之外，vim 的矩形编辑也非常犀利。比如上面的示例，也可以在矩形编辑下，选中第二到第十行的 pick，删除，然后 I 插入 squash，继而退出 insert mode 即可。

而 emacs 没有输入上的 mode 差别，所以需要依赖复杂的快捷键来实现强大的编辑功能，正如上图所示。emacs 插件想象力更加丰富，有「伪装成编辑器的操作系统」之称。插件的 major mode 和 minor mode 的设计很出彩，对一个文件，只有一个 major mode，但是可以有多个 minor mode，这样一个文件一个主插件，多个附加插件，可以实现很多有趣的效果。在 vim 中，是通过`set filetype=python`或者在`filetype.vim`文件中自定义来决定 vim 使用哪种语法渲染，其他比如自动补全这样的插件通过判断`filetype`来实现相关功能，并没有 mode 一说，针对同一种文件类型的插件可以非常分散。而在 emacs 中，如果我们选中`pythonA-mode`作为`.py`文件的 major mode，那么`pythonB-mode`就不会起作用，除非它是 minor mode。这有利于大而优秀的特定 major mode 脱颖而出，同时使用多个 minor mode 提供通用编辑功能。

![emacs major mode](http://wulfric.qiniudn.com/emacs-major-mode.png "emacs major mode")


## 打造 IDE 的尝试

有很多人试图将 vim/emacs 打造成 IDE，也有一些比较著名的配置。比如，对于 vim，比较优秀的 IDE 配置有 [spf13](http://vim.spf13.com/)，[kvim](https://github.com/wklken/k-vim)，[fisa](https://github.com/fisadev/fisa-vim-config)（这个并不著名但我很喜欢，我的配置也是从这个开始的）等。emacs 有 [prelude](https://github.com/bbatsov/prelude)，[purcell](https://github.com/purcell/emacs.d) 和我现在在用的 [spacemacs](https://github.com/syl20bnr/spacemacs)。如果有兴趣，可以去这些项目主页看一看，然后选择一个尝试一下。

为了实现类似 IDE 的功能，这些配置通常包括了项目结构列表，文件结构列表，自动跳转，自动提示和补全，插件管理，语法检查，版本控制等插件。如果上面每个配置项目你都过了一遍，会发现大家要做的事情其实是差不多的。对于 vim，可以看下这个 [vimawesone](http://vimawesome.com/)，其实最受欢迎的插件也大概是这些，对着 vimawesone 你也能拼起来一个很优秀的配置。

### 项目结构浏览插件

于编辑器而言，这个插件的功能通常都比较简陋，一般只能浏览和导航文件，加上简单的文件操作（增删改复制）。不像 IDE，提供的功能非常多，多到右键弹出功能列表的时候都会卡顿（没错，我并没有说 JetBrains 家的 IDE）。vim 中比较优秀的是 [nerdtree](http://github.com/scrooloose/nerdtree)，emacs 下是 [neotree](https://github.com/jaypei/emacs-neotree)，其实就是仿的 nerdtree。对于 vim/emacs 用户而言，不会通过在文件树中点击来跳转，使用此类插件其实仅仅是为了浏览项目结构，所以往往不会做的功能特别强大。

### 快速定位

得益于 sublime text 的 go to anywhere 思想，ctrlp 几乎成为了现代编辑器的标配功能。所谓 go to anywhere，就是通过一个快捷键（一般是 ctrl+p）能够通过模糊查找快速到达项目中的任意文件、类、方法。毫无疑问，在编辑器中，这个功能 sublime 做的最好。在 sublime 中，ctrlp 会弹出一个输入框，直接输入，会查找文件，先输入`@`，则会查找方法，先输入`:`，则会跳到这一行。而且还支持组合查找。

![sublime-text-ctrlp](http://wulfric.qiniudn.com/sublime-text-ctrlp.png "sublime-text-ctrlp")

其实 JetBrains 系 IDE 的 go to anywhere 功能更加强大，可以同时搜索文件、类、方法、IDE 动作。代价就是性能太差---每次`⇧⇧`都会卡顿，所以只好使用`⌘+O`查找文件，查找到文件之后再查找方法，或跳转到具体行。这意味着，在这一方面，更强大的 IDE，反而比编辑器更不方便。这倒不是因为它是 IDE，而是软件设计的一个问题---哪些功能应该合在一起，哪些功能应该分开。

vim 下的 go to anywhere 插件名字就叫 [ctrlp](http://ctrlpvim.github.io/ctrlp.vim/)，仅仅实现了查找文件功能，需要通过[插件](https://github.com/ctrlpvim/ctrlp.vim/tree/extensions)来扩展功能，比如查找 vim 命令的 [ctrlp-funky](https://github.com/tacahiroy/ctrlp-funky)。而且，不知是不是技术限制，其搜索精确度还是无法和 sublime 相比。不仅如此，搜索速度也不够好，需要借助插件 [ctrlp-py-matcher](https://github.com/FelikZ/ctrlp-py-matcher) 。这一切搭配好之后，ctrlp 还是可以用得很好的。（PS：[vim-ctrlspace](https://github.com/vim-ctrlspace/vim-ctrlspace) 提供了一种新型的文件编辑管理方式，使用 go 写了模糊查询，并没有使用过，感兴趣的可以尝试下）

emacs 下的 go to anywhere 插件有好几个，spacemacs 默认使用的是 [projectile](https://github.com/bbatsov/projectile)。使用感觉和 vim 的 ctrlp 很像，中规中矩。通过 [helm-projectile](https://github.com/bbatsov/helm-projectile) 扩展实现对 emacs 内置命令的模糊查询。

总而言之，在这一功能上，vim/emacs 的实现比较分散，消耗过多认知资源，sublime 实现的最好，JetBrains 则过于集中。

### 补全和跳转

自动补全和跳转，这两个功能就是 IDE 的强项了。IDE 解析语法树，可以实现相当精准的补全和跳转，而编辑器基于字符串匹配，效果就要大打折扣了。当然了，我说的是静态语言🙄。对于动态语言，即使是 IDE，也总有力所不及的地方， 而编辑器开一个语言解释器进程实时解析也能实现不错的效果。二者的差别没那么明显。对于静态语言，编辑器竟也有和 IDE 相抗的野心：[eclim](http://eclim.org/)，也就是 eclipse+vim（当然也有 emacs 插件），在后台开一个 eclipse 进程，然后在 vim 中利用 eclipse 来做补全和跳转……

![说得好，我选择死亡](http://wulfric.qiniudn.com/woxuanzesiwang.jpeg "说得好，我选择死亡")

众生皆苦，何必苦上加苦？

### 其他

|    功能    |             vim             | emacs                 |
| :------: | :-------------------------: | --------------------- |
|   版本管理   |     fugitive/git gutter     | maggit                |
|   语法检查   |          Syntastic          | Flycheck              |
| Snippets |  vim-snippets/vim-snipmate  | yasnippets            |
|    补全    | NeoComplCache/youcompleteme | company/auto-complete |

然而比起这些来，我更喜欢的是针对编辑的一些插件，比如 [vim-surround](https://github.com/tpope/vim-surround)，能够快速的将字符串的包裹修改或者删除。

``` bash
"Hello world!"  
# cs"'
'Hello world!'
# cs'<q>
<q>Hello world!</q>
# cst"
"Hello world!"
# ds"
Hello world!
```

[vim-repeat](https://github.com/tpope/vim-repeat) 重复上一个动作。[Gundo](https://github.com/vim-scripts/Gundo) 树形撤销历史。[vim-exchange](https://github.com/tommcdo/vim-exchange) 交换行，或者交换选中区域。

更多有趣的 vim 编辑插件可以看[这里](http://spacemacs.org/doc/DOCUMENTATION#evil-plugins)。


## 使用哲学

对于这种将 vim/emacs 打造成 IDE 的尝试，有的人很热衷，有的人则很反对。热衷的抱着一股热忱，相信经过自己的打造，vim/emacs 的使用体验可以不输 IDE，甚至在很多细节要优秀得多。反对的人认为无论如何编辑器的自动补全和跳转都无法达到 IDE 的精度，何必徒劳。嘛，其实都有道理，看你是什么样的性格咯。

但是学习这两个编辑器还是非常有好处的。就像一开始说的，很多编辑器、IDE 都提供 vim 插件，学会了 vim 可以一套快捷键吃遍天。另外，bash 默认是 emacs 模式，所以熟悉 emacs 还可以提高 bash 下的效率。比如 emacs 的[移动命令](https://www.gnu.org/software/emacs/manual/html_node/emacs/Moving-Point.html)和[删除命令](https://www.gnu.org/software/emacs/manual/html_node/emacs/Erasing.html)就可以部分用在 bash 上。最常用的是这几个[快捷键](https://www.gnu.org/software/emacs/manual/html_node/emacs/Words.html)。而且，不得不说，在纯粹的编辑上，vim/emacs 效率更高。

在具体的项目上，当然还是 JetBrains 家的 IDE 更好用，尤其是静态语言。而动态语言，选自己比较喜欢的就好了。（然而 JetBrains 家的调试工具太好用已经离不开了☹️）

PS: vim 快捷键一览 [vimgifs.com](http://vimgifs.com/)
