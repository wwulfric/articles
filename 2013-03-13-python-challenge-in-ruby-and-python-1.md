---
title: python challenge in ruby and python 0~12
date: 2013-03-13 20:36
tags: [ruby, python, pythonchallenge]
categories: [技术]
---

[Python Challenge](http://www.pythonchallenge.com/) 是一个非常有趣的闯关游戏，通过猜谜和编程得到关键词，并使用关键词更改网页的 URL 进入到下一关。现在共有 33 关。这篇文章（或者更多）将会专注于使用 python 和 ruby 解决这些问题。

![python challenge](http://wulfric.qiniudn.com/python-challenge.png "python challenge")

<!--more-->


## 第 0-6 题

第 0 题很简单（不过我第一次做的时候完全不了解，所以第 0 题都是看提示做的……），有三个数字，现在的 URL 是 /0.html, 我们根据这三个数字得到一个数字（看看能得到什么数字），然后更改 0，就进入下一题了。

### 第 1 题

从这一题开始就需要一些编程的技能了。请仔细观察[图中](http://www.pythonchallenge.com/pc/def/map.jpg)的 6 个字母的关系。 
提示：将题目给出的每一个字母向后移动两个位置。

~~~python
# python
# 这是我的 python 解法，相当的 C 呢
astring='''g fmnc wms bgblr rpylqjyrc gr zw fylb. rfyrq ufyr amknsrcpq ypc dmp.
 bmgle gr gl zw fylb gq glcddgagclr ylb rfyr'q ufw rfgq rcvr gq qm jmle. sqgle
 qrpgle.kyicrpylq() gq pcamkkclbcb. lmu ynnjw ml rfc spj.'''
import re
bstring=''
for a in astring:
    if re.match('\w',a):
        a=chr((ord(a)-95)%26+97)
    bstring=bstring+a
print (bstring)
~~~

~~~python
# python
# 推荐的做法
import string
text='''g fmnc wms bgblr rpylqjyrc gr zw fylb. rfyrq ufyr amknsrcpq ypc dmp.
 bmgle gr gl zw fylb gq glcddgagclr ylb rfyr'q ufw rfgq rcvr gq qm jmle.
 sqgle qrpgle.kyicrpylq() gq pcamkkclbcb. lmu ynnjw ml rfc spj.'''
table = string.maketrans(string.ascii_lowercase,
                         string.ascii_lowercase[2:]+string.ascii_lowercase[:2])
text.translate(table)
~~~
~~~ruby
# ruby
gets.tr "a-z", "c-zab"
# 就这么简单！然后输入字符串就好了！真教人喜欢！
~~~

在官网的解答上还有利用 *nix 系统的 tr 函数的方法，本质上和 ruby 的方法是一样的：`curl http://www.pythonchallenge.com/pc/def/map.html | tr a-z c-za-b`。

### 第 2 题

按照提示，我们到源代码中找到了线索：从一大串乱码中找到所有的稀有字符。一开始我以为是找出所有的字母，确实也能得到正确答案。不过官方答案中给出的方法是按照字符出现的频率来查找的。看来我误解了 character 的含义了。但是观察这段乱码应该也能猜到只有字母才是最稀少的。

提示：因为这段乱码较长，因此在交互工具中无法得到正确的结果(我在 bpython 中失败)，请在编辑器中完成代码，然后运行得到结果。

我选取的代码都是识别字母的方法，判断频率的方法也类似，不再赘述。

~~~python
# python
# 我的代码，判断每个字符是否是字母
import re
astring = '''copy from the source page'''
bstring = ''
for a in astring:
if re.match('[a-zA-Z]',a):
    bstring = bstring+a
~~~

其中最后判断的部分可以换成`print re.sub('[^a-z]','', s)`。

~~~ruby
# ruby
s = <<'EOF'
# copy from the source page
EOF
puts s.scan(/[a-z]/).join
~~~

### 第 3 题

这一题给的提示很明显了，就是找出所有左右被三个大写字母包围的小写字母。方法和第 2 题基本是一样的。

提示：用正则表达式查找更方便。前面虽然也用到了正则，但是这里不用正则的话就会很麻烦。

~~~python
# python
# 我的代码比较笨，但是方法是一样的，就不贴上来了。
import re
s = '''
copy from the source page
'''
print ''.join(re.findall('[^A-Z][A-Z]{3}([a-z])[A-Z]{3}[^A-Z]', s))
~~~

~~~ruby
# ruby
s = <<'EOF'
copy from the source page
EOF
puts s.scan(/[^A-Z][A-Z]{3}([a-z])[A-Z]{3}[^A-Z]/).join
~~~

### 第 4 题

进入源代码页可以发现可点击的链接，点进去再结合说明就知道了，你得不断换 nothing 后面的数字，直到有人告诉你该停下来了 :)

提示：当遇到问题退出时，将最后得到的数字复制到网址上，看看有什么提示，按照提示做就行了。

请记住这一题的图片的样子，以后会用到~

~~~python
# python
import re
from urllib import urlopen
astring='http://www.pythonchallenge.com/pc/def/linkedlist.php?nothing='
temp='12345'
f = urlopen(astring+temp)
a=f.read()
restring=re.compile('and the next nothing is (\d{3,5})')
for i in range(400):
    temp=re.findall(restring,a)[0]
    print(temp)
    f=urlopen(astring+temp)
    a=f.read()
~~~

~~~ruby
# ruby
require "open-uri"
n=12345
loop do
    flag = ''
    open("http://www.pythonchallenge.com/pc/def/linkedlist.php?nothing=#{n}") do |f|
        s=f.read
        flag = s if s=~/htm/
        n=(s=~/div/ ? n/2 : s.split[-1].to_i)
    end
    if !flag.empty?
        puts flag
        break
    end
end
# 需要注意的是，我本来是把 break 放在 open 这个 block 里的，我以为当 s 匹配到 htm 时，
# 接着 break 就可以退出了。但是 ruby 的 break 是退出最临近的 block，所以 loop 还是会
# 死循环下去。因此不得不加了一个 flag。
~~~

或者同时借助 ruby 和 bash 命令，在 命令行如下输入，也可以得到需要的结果。

~~~ruby
# ruby
ruby -wle 'n = 12345; print n = $1 while 
          `GET "http://www.pythonchallenge.com/pc/def/linkedlist.php?nothing=#{n}"`
          =~ /(\d+)$/'
~~~

### 第 5 题

这一题我以前一直不明白，现在也是……问题的关键肯定是在源代码中的 banner.p，看了别人的介绍才知道要用 python 的 pickle 模块。这是一个将 python 里的对象持久化到磁盘的一个工具。

提示：将该对象通过 pickle 模块 load 出来后，仔细观察，不妨把 data 按行打印出来看看。每一行的数字加起来都是 95，猜测每一行由 95 个字符构成，剩下的事情就简单了。

~~~python
# python
# 推荐的做法
import urllib2, sys, pickle
pickle_data = urllib2.urlopen('http://www.pythonchallenge.com/pc/def/banner.p').read()
data = pickle.loads(pickle_data)
for line in data:
    for char, count in line: sys.stdout.write(char * count)
    sys.stdout.write("\n")
~~~

这个真的就只能 python 做了……

### 第 6 题

这一题的提示表明上看起来相当少，但是当你直到是什么的时候就会觉得，嘿，其实提示还不少！（坑爹呢这是！）注意这幅图片，像不像什么软件的图标？还有源代码的注释部分……

提示：和 zip 相关。zip 的图标就是一个拉链。注意到源码的 html 标签后面有一个 zip 注释，还等什么，把网址的 html 后缀改成 zip 吧。

进入 zip，首先看到的就是 readme.txt，它告诉我们从 90052 开始。那就开始吧，和第 4 题一样。直到找不到数字的时候，输出最后一个文件的内容，它告诉我们去收集 comment，OK！收集 comment 然后输出就行了！

~~~python
# python
f = "channel.zip"
z = zipfile.ZipFile(f)
name = '90052.txt'
while 1:
    print z.getinfo(name).comment,
    name = z.read(name).split()[-1] + '.txt'
~~~

~~~ruby
# ruby
require "zip/zip"
zipfilename = "/home/haidao/Downloads/channel.zip"
Zip::ZipFile.open(zipfilename) do |zipfile|
  s = zipfile.get_input_stream("readme.txt").read
  num = s.match(/\d{2,6}/)
  output = Array.new
  while num do
    entry = zipfile.find_entry("#{num}.txt")
    s = entry.get_input_stream.read
    num = s.match(/\d{2,6}/)
    comment = entry.comment
    output.push comment
  end
  puts output.join
end
~~~

[0](http://www.pythonchallenge.com/pc/def/0.html), [1](http://www.pythonchallenge.com/pc/def/map.html), [2](http://www.pythonchallenge.com/pc/def/ocr.html), [3](http://www.pythonchallenge.com/pc/def/equality.html), [4](http://www.pythonchallenge.com/pc/def/linkedlist.php), [5](http://www.pythonchallenge.com/pc/def/peak.html), [6](http://www.pythonchallenge.com/pc/def/channel.html)

## 第 7-12 题

接下来的几个涉及图像的题目开始有难度了。先从一个简单开始 :)

### 第 7 题

一进去就感到中间那条线很可疑！全是灰色的块块，我们知道灰色的 RGB 是相等的，且从 0-255，不正是 ASCII 码吗。好的，把这些灰色块块换成数字，再换成 ASCII 码吧。

提示：python 需要 PIL 模块，ruby 需要 Rmagick 模块。python 通过 pip 安装 PIL；ruby 安装 Rmagick 需要先安装`sudo apt-get install libmagickwand-dev`，然后直接`gem install rmagick`。

~~~python
# python
import re, Image
im = Image.open("oxygen.png")
row = [im.getpixel((x, 47)) for x in range(0, im.size[0], 7)]
chrs = [chr(r) for r, g, b, a in row if r == g == b]
s = ''.join(chrs)
print ''.join(map(chr, map(int, re.findall(r'\d+',s))))
~~~

~~~ruby
# ruby
require "RMagick"
Magick::ImageList.new("oxygen.png").get_pixels(0,45,629,1).map.with_index
  {|p,i| (p.red/255).chr if i%7==0)}.join.scan(/\d+/){|c| print c.to_i.chr }
~~~

### 第 8 题

蜜蜂（bee）！源代码中一串可疑的字符串（BZ 开头）！其实我也是看提示才知道用到了 python 的 bz2 模块……

python 中的模块叫 bz2，是内置的；ruby 的需要先下载：`gem install bzip2-ruby`。

~~~python
# python
import bz2
un = "un 的字符串内容，表示 user name"
pw = "pw 的字符串内容，表示 password"
print bz2.decompress(un), bz2.decompress(pw)
~~~

~~~ruby
# ruby
require 'bzip2'
un = "同上，需要用双引号括起来"
pw = "同上，需要用双引号括起来"
puts Bzip2.uncompress(un), Bzip2.uncompress(pw)
~~~

### 第 9 题

个人认为这一题的提示不够明显。标题告诉我们要把点连起来。我以为说的是图上的点呢。first 和 second 也是一串数字，很难想到是两组点的集合。

提示：源代码给的 first 每两个数字一组，是一个点的位置，从第一个点开始画，一直画到最后一个点，seoncd 也是这样。原图上的点是要告诉你，连好后看点线的外观。这是一头牛。

~~~python
# python
from PIL import Image, ImageDraw
img = Image.new("RGB",(640,480), 'white')
draw = ImageDraw.Draw(img)
draw.polygon(first_list, 'black')
draw.polygon(second_list, 'grey')
img.show()
~~~

~~~ruby
# ruby
require "RMagick"
img = Magick::ImageList.new 'good.jpg'
gc = Magick::Draw.new
gc.fill('#CC6600')
gc.polygon(*first_list)
gc.fill('white')
gc.polygon(*second_list)
gc.draw(img)
img.display
~~~

### 第 10 题

这一题和图片基本没关系了（当然也可能是我眼拙）。tilte: what are you looking at? 是一个非常好的提示。再结合给出的序列……反正我是没想到……

这是一个 [look and say sequence](http://en.wikipedia.org/wiki/Look-and-say_sequence)，它是一个比较特殊的序列。后者是前者读出的结果，比如：前一个数是 11，则后一个数就是 21，表示 2 个 1；再后一个数就是 1 个 2，1 个1，即 1211。

~~~python
# python
from re import findall
def f(n):
    return "1" if n == 0 else ''.join(
        ["%d%s"%(len(j)+1, i) for i,j in findall(r'(\d)(\1*)', f(n-1))])
print len(f(30))
~~~

~~~ruby
# ruby
def f n
  n == 0 ? "1":
    f(n-1).scan(/(\d)(\1*)/).collect{|i| "#{i[1].size+1}#{i[0]}"}.join
end
puts f(30).size
~~~

额，感觉难度越来越大了……

### 第 11 题

奇和偶，应该和图片的细节有关。在 PhotoShop 里放大观察可以发现两两间隔有不少黑色的方块。问题的关键点肯定就在这里了。

提示：用下一行的黑色像素覆盖上一行的非黑色像素。

~~~python
# python
from PIL import Image
img = Image.open("cave.jpg")
c,r = img.size
pix = img.load()
for j in range(r-1):
    for i in range(c):
        if (i+j)%2:
            pix[i,j] = pix[i,j+1]
img.show()
# 注意也可以用 img.getpixel 和 img.putpixel 来操作像素。
~~~

或者直接`Image.open('cave.jpg').resize((320, 240)).show()`这样也可以。本来原理就是把黑色的像素块拼起来。你也可以在 PS 里选择以邻近的方法将图像大小重新设置为 320*240。

~~~ruby
# ruby
require "RMagick"
img = Magick::ImageList.new 'cave.jpg'
r, c = img.rows, img.columns
for i in 0..r-2
  for j in 0..c-1
    if (i+j).odd?
      p = img.get_pixels(j, i+1, 1, 1)
      img.store_pixels(j, i, 1, 1, p)
    end
  end
end
img.display
~~~

### 第 12 题

这一题真难……首先仔细观察源码（源码才几行有什么好观察的！），注意到图片的名字叫 evil1.jpg，诶～title 告诉我们要 dealing evil，那么就先看看 evil2.jpg 是什么。有惊喜吧！它告诉我们不是 jpg，是 gfx。先不管，继续 dealing evil，换 evil3.jpg，没啦！不管，还是继续，evil4.jpg：一张坏掉的图？用文本编辑器打开它，`bert is evil, go back!`

好吧，先记住 bert，url 改成 bert.html 会有一张图，不过看起来没什么用，还是回到 evil2 吧！把 jpg 改成 gfx，好，得到一个文件。剩下的所有工作都是在这个文件里进行。仔细观察这个文件的头部，会发现它很奇怪。有点熟悉，但是又不是那么熟悉。 ~~好吧我是看了别人的提示才知道的。~~ 原来这个文件中，隐藏着若干个图片！每隔 5 个字符看，很容易的就能找到 GIF 和 PNG。还等什么，马上写代码把他们分开吧！

提示，用可以查看二进制文件的编辑器查看这个 .gfx 文件。在 vim 里是输入`:%!xxd`即可。

~~~python
# python
raw = open('evil2.gfx','rb').read()
for i in range(5):
    open('out%d'%i, 'wb').write(raw[i::5])
~~~

~~~ruby
# ruby
# 一定要注意，读写文件要强调是二进制
evil =  File.open("evil2.gfx", 'rb').read
outputs = (0..4).map { |i| File.open("out#{i}", "wb") }
evil.each_char.with_index { |c, i| outputs[i%5].write(c) }
~~~
