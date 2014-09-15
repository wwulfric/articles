git submodule

压缩 css: sass: :compressed
kramdown: input: GFM

用到的 Jekyll 插件：
压缩 HTML: https://github.com/penibelst/jekyll-compress-html
弃用原因：压缩 js，使其失效
原因：该压缩方法直接去除换行符，则 script 中的 // 注释会导致该段代码失效


kramdown-with-pygments https://github.com/mvdbos/kramdown-with-pygments
改用 kramdown 的 coderay 来渲染代码

syntax: github: https://github.com/mojombo/tpw/blob/master/css/syntax.css

image encode to base64: https://github.com/GSI/jekyll_image_encode
当你想隐藏你的图床时，比如，七牛云可以加水印，但是如果知道 url
的话，很容易就可以去掉水印，转成 base64 可以防止这种情况

自动给图片加宽度 读在线文件，读尺寸，加到 img 里。对于 R-图片，即 retina
图片，width 设为1/2。还可以同时给 img 加 figure 和 caption。

图片的显示，jquery.modal or fancybox or vex

对 albums 的显示

color scheme: https://kuler.adobe.com/main-cxcedar-color-theme-4237607/

压缩
http://davidensinger.com/2013/08/how-i-use-reduce-to-minify-and-optimize-assets-for-production/
https://github.com/grosser/reduce

assets: https://github.com/ixti/jekyll-assets

字体选择，宽度

轻交互 各种效果，比如鼠标经过的时候 border
变色，鼠标经过的时候图片旋转等（多说的经典评论样式）

jekyll tags: http://guojing.me/jekyll-tags-categories-and-archive/
http://stackoverflow.com/questions/1408824/an-easy-way-to-support-tags-in-a-jekyll-blog

多说评论比 disqus 好一点的是，它插入的是普通的 html 标签，也就可以通过 css
来自定义，而 disqus 不可以; 当然，disqus 以 iframe
方式引入，其样式不会被原样式搞乱掉。可以以 ajax 方式请求 disqus 的评论

多说评论的样式

评论可以延迟加载，等到页面浏览到底部时才发起请求。

date 样式

coderay 代码高亮线上不支持

break-word firefox 不支持
