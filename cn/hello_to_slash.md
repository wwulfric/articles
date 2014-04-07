title: Use slash theme
date: 2013-01-04 14:27
tags: [octopress, theme, slash]
categories: [技术]
---

{% img http://www.lufeipic.tk/f/l/?.jpg 'preview of slash theme' 'preview of slash theme' %}

After I have had my own octopress blog, I start to look for a different theme. And fortunately I've find it. It is slash. Simple and cool. Thanks, [zespia](http://zespia.tw/).

<!--more-->

### Install

Type the code below and experience Slash!

```bash
cd octopress
git clone git://github.com/tommy351/Octopress-Theme-Slash.git .themes/slash
rake install['slash']
rake generate
```

Have problems when installing with zsh? Try `rake install\['slash'\]` instead.

See the features and Q&As [here](http://zespia.tw/Octopress-Theme-Slash/).

### Personal Change

It's not so comfortable to have the exectly same appearence as others, so I decide to change it :)

#### Switch Left and Right

This is quite easy. Just edit the **margin** attribute value of div.post and div.meta in `sass/parts/_post.scss`. 

#### Create Index

I want to display the content of my article in the right side. Let's use jQery to add the chapter index.

``` html
<script type="text/javascript">
  tmp = '<div><a href="#">CONTENTS</a></div>';
  $('.post-index').append(tmp);
  $.each($('h3, h4'), function(index, item){
      item.id = 'menuindex'+index;
      tmpText = $(item).text();
      if(item.tagName.toLowerCase() == 'h3'){
        tmpClass = 'menuindexh3';  
      }else{
        tmpClass = 'menuindexh4';
      }
      tmp = '<div class="' + tmpClass +'"><a href="#' + item.id +
            '">' + tmpText + '</a></div>';  
      $('.post-index').append(tmp);
  });
</script>
```

Put this code snippet to `source/_layouts/post.html`,
and append `<div class="post-index"></div>` to div.meta in `souce/_includes/article.html`.
Then edit its style:

``` scss
.post-index{
  position: fixed;
  top: 250px;  
  padding-top: 20px;
  a{@include link-colors($color-main, $color-main-darken);}
  .menuindexh4{
    padding-left: 15px;
  }
}
```

And put this snippet to the .meta block of `sass/parts/_post.scss`.


