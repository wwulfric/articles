---
title: Jekyll 博客的一些优化插件与配置
date:  2014-09-17 01:01
tags: [jekyll, compress, highlight, fontawesome, colorscheme, font]
categories: [技术]
---

在停用了一段时间后，最终还是回归博客写作了。之前使用的是 Octopress，现在换成更简单的 Jekyll。本文记录了搭建 Jekyll 博客的一些优化细节和搜集到的一些有趣的插件。

## 工作方式

[Jekyll](http://jekyllrb.com/) 的工作方式是，服务器端（对于 github pages，就是 github 的服务器）将特定文件夹下（根目录，`_posts`）的文件编译成静态文件，我们只需要在该文件夹下写 md 文件就好了，无需生成 html。

另一种工作方式则是，我们创建一个新的分支（比如 source）来存放 Jekyll 代码，生成的静态站点（在`_site`下）作为 master 主分支，这样每次生成静态站点，然后部署到 github。

显然，前者更加方便，但基于安全考虑，github 不支持过于自由的操作，故大量自定义的插件无法在 github pages 使用。所以只好分成两个分支，master 分支是生成站点，source 是 Jekyll 主代码。这也就是 [Octopress](http://octopress.org/) 的工作方式了。相比于 Jekyll 的简陋，Octopress 提供了更丰富的功能，比如分类、标签、更强大的代码高亮等。与 Octopress 类似的还有基于 nodejs 的 [Hexo](http://hexo.io/)，同样是先编译后部署，因此可以提供更强大的功能。

## 代码分离

使用 [Git submodule](http://git-scm.com/book/zh/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97) 分离博客的重要部分。我把文章专门放到一个 repo 里，然后通过 git submodule 的方式引入到`_posts`文件夹下。一些插件也可以以这种方式引入到`_plugins`里。

## 静态文件优化

静态文件就是浏览器渲染网页必须的 html, css, js 和图片文件。我对 Jekyll 博客的静态文件优化主要集中在两个方面，即压缩和打包。

Jekyll 提供了内置的对 Sass 和 CoffeeScript 的支持，见 [Assets](http://jekyllrb.com/docs/assets/)。Jekyll 已支持对 css 的打包和压缩，但为了统一管理，还是使用了 [jekyll-assets](http://ixti.net/jekyll-assets/) 来处理合并压缩相关的任务。该插件功能强大，参考其文档可以大大简化静态文件的管理。

原本我也曾经使用 [jekyll-compress-html](https://github.com/penibelst/jekyll-compress-html) 来压缩 html 文件。但因为该压缩方法会直接去除换行符，则 script 中的 // 注释会导致该段代码失效（只要有一行代码的注释是以 // 开始的，该行之后的代码都将失效）；同时，kramdown 生成的 mathjax 会比较依赖换行（[CDATA](https://github.com/gettalong/kramdown/issues/100)），在对比了生成的 html 和压缩后的差别之后发现，压缩率并不高，故最终还是取消了该插件的使用。

还有一些压缩静态文件的策略，比如该[博文](http://davidensinger.com/2013/08/how-i-use-reduce-to-minify-and-optimize-assets-for-production/)中提到的 [reduce](https://github.com/grosser/reduce)。它甚至可以压缩图片文件。我尝试了下图片压缩，比率并没有太大的惊喜（毕竟图片文件本身已经经过了高效的压缩），故最终没有使用。

还有一个插件 [jekyll_image_encode](https://github.com/GSI/jekyll_image_encode)，可以将图片转成 base64。其实本质上和把图片下载到本地一样，只是更加自动化。base64 转化后的 img 标签是这样的：

~~~html
<img src="data:image;base64, iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAAAXNSR0IArs4c
6QAAAAxJREFUCNdj+P//PwAF/gL+3MxZ5wAAAABJRU5ErkJggg== " />
~~~

可见，src 处已经没有 url 了，而只是图片的 base64 格式源码。当你想隐藏你的图床时，比如，七牛云可以加水印，但是如果知道 url 的话，很容易就可以去掉水印，转成 base64 可以防止这种情况。

除此之外，我还利用 minimagick 库自动给图片加了宽度和高度属性[^1]。即读在线文件，获取尺寸，加到 img 里。对于 R-图片（我给在 rMBP 上制作的图片都加了 R- 前缀），即 retina 图片，width 设为1/2。另外还同时给 img 加上了 figure  wrapper 和 figcaption。代码如下，打开类`Kramdown::Converter::Html`并重写`convert_img`方法。

~~~ruby
def convert_img(el, indent)
  image = MiniMagick::Image.open el.attr['src']
  width = image[:width]
  height = image[:height]
  unless el.attr['src'].match(/R-/).nil?
    width /= 2
    height /= 2
  end
  "<figure>
    <a class='post-image' rel='post-image' href='#{el.attr['src']}'>
      <img#{html_attributes(el.attr)} width=#{width} height=#{height} />
    </a>
    <figcaption>
      <i class='icon-pencil'></i>
      #{el.attr['alt']}
    </figcaption>
  </figure>"
end
~~~

对于博客中常用的图标，我是用的是 [Font Awesome](http://fortawesome.github.io/Font-Awesome/)。Font Awesome 图标比较多，我们可以[按需引用](http://www.w3cplus.com/preprocessor/create-font-awesome-font-icons-with-sass.html)，或者直接[定制](http://fontello.com/)需要的图标。前者通用性更好，但实现复杂；对于博客而言，直接定制即可。

其他可供参考的优化方法：[seo](http://pizn.github.io/2012/01/16/the-seo-for-jekyll-blog.html)。我把每篇文章的 tags 作为 keywords 放在 html 的 header 里，把每篇文章的第一段作为 description 也放在 html 的 header 里，尽量对搜索引擎友好。

## 样式

使用 markdown 写博客，我比较坚持的一点就是，写出的 markdown 文件尽最大可能保持自文档特性，即语法简洁容易理解，尽量不包含 github  不支持显示的语法。换句话说，就是尽量以 [GFM--GitHub Flavored Markdown](https://help.github.com/articles/github-flavored-markdown) 为标准。所以我避免使用一切 Jekyll tag，除了数学公式（github 现在还不支持常见的以`$`符识别 tex 的特性，但数学公式实在不可避免）。

Jekyll 的代码高亮默认是使用 highlight tag。我则只使用 [fanced codes block](https://help.github.com/articles/github-flavored-markdown#fenced-code-blocks) 来标识代码，而 kramdown 不支持 pygments。为代码高亮带来不少麻烦。可以使用[该插件](https://github.com/mvdbos/kramdown-with-pygments)来 hack kramdown 使之支持 pygments。配合 github 风格的代码高亮 [syntax](https://github.com/mojombo/tpw/blob/master/css/syntax.css)，神清气爽。当然，kramdown 自带 coderay 也可以完成代码高亮，两种方法任选一即可。需要注意的是，基于安全原因，大量自定义的插件无法在 github pages 使用，请注意你的 Jekyll 的工作方式。

对于图片的显示，我选择的是[fancybox](http://fancyapps.com/fancybox/)。

对于网页的配色（color scheme），我在[这个方案](https://kuler.adobe.com/main-cxcedar-color-theme-4237607/)的基础上做了些修改，得到了[我的 color scheme](https://kuler.adobe.com/Copy-of-the-main-cxcedar-color-theme-4361434/)。这五种颜色分别用作链接色、重要标题背景色、mark 背景色、强调色和主文字颜色。

![color scheme](http://qiniu-wulfric.lufeihaidao.top/R-colorscheme.png "color scheme")

对于字体的选择，我使用的是和 [slash 主题](https://github.com/tommy351/Octopress-Theme-Slash/)相同的 font-family，尚未对 Windows 字体做兼容性处理。关于字体的选择，可参考[这篇文章](https://ruby-china.org/topics/14005)。

在做博客样式的时候，我坚持轻交互。即如果不是没有交互就不行的情况，就不加交互。比如利用输入框边框颜色的变化来表示获得焦点和失去焦点的差别是非常合理的，鼠标经过代码块时边框变色就是不必要的。链接 hover 的时候有变化（下划线，变色等）是必要的，鼠标经过的时候头像旋转就是不必要的，等等。

最终的样式如图所示：

![Preview](http://qiniu-wulfric.lufeihaidao.top/R-preview.png "Preview")

如果对不同平台下的样式有疑问，欢迎通过评论告诉我~

PS: 本博客将在一千年以后兼容 IE，敬请期待~

[^1]: http://stackoverflow.com/questions/1247685/should-i-specify-height-and-width-attributes-for-my-imgs-in-html
