---
title: "Personal Blog with Github Page and Octopress"
date: 2012-11-30 00:19
categories: [git, octopress] 
---

{% img http://www.lufeipic.tk/f/m/?.png 'Github and Octopress' 'Github and Octopress' %}

**Octopress Blog: Blog for geek!** This article focus on some simple steps to create a blog with github page and octopress.

<!-- more -->

### Preparation Work

I suppose that you've installed such softwares before you start:

* git
* ruby with RVM

#### Clone Octopress Code

``` bash
git clone git://github.com/imathis/octopress.git octopress
cd octopress    # If you use RVM, You'll be asked if you trust the .rvmrc file (say yes).
ruby --version  # Should report Ruby 1.9.3
bundle install  # Install what octopress need
rake install    # Install the default Octopress theme
```

Now you can type `rake preview` to preview the default octopress blog [here](localhost:4000).

#### Prepare Your Github Page

Create a new Github repository and name the repository with your user name or organization name `username.github.com` or `organization.github.com`. Just like this:   

{% img http://www.lufeipic.tk/f/k/?.jpg 'github page' 'github page' %}

Execute `rake setup_github_pages` and fill in your github page URL. You will then find a _deploy folder in your working directory which would be deployed remotely.

#### Deploy and Push

Deploy it to github page by:

```bash
rake generate
rake deploy
# You can use `rake gen_deploy` instead of them
```

This will generate your blog, copy the generated files into _deploy folder, add them to git, commit and push them up to the master branch. In a few seconds you should get an email from Github telling you that your commit has been received and will be published on your site.

**DO REMEMBER** to commit the source for your blog. 

```bash
git add .
git commit -m 'my first commit'
git push origin source
# You will have two branches: 'master' to deploy your code and 'source' to write and change your blog.
```

  
 
### Post Your First Article



Now you've already owned your blog on github page. You may have noticed that the default page is not what you perfer. Let's modify it.

Run `vi _config.yaml` to edit the _config.yaml file (Feel free to use any editor you like). Edit it like this:  
```
url: http://lufeihaidao.github.com
title: Wulfric's Blog
subtitle: FEEL FREE TO CHANGE THE WORLD!
author: Wulfric Wang
simple_search: https://google.com/search?q=
description:
```

#### Write Your First Article

Blog posts must be stored in the source/_posts directory and named according to Jekyll’s naming conventions: YYYY-MM-DD-post-title.markdown. The name of the file will be used as the url slug, and the date helps with file distinction and determines the sorting order for post loops.

Octopress provides a rake task to create new blog posts with the right naming conventions, with sensible yaml metadata.

```
rake new_post['My first article']
```

You will find a new markdown file in your source/_posts/ folder. Edit it!

```
---
layout: post
title: "Personal Blog with Github Page and Octopress"
date: 2012-11-30 00:19
comments: true
categories: git 
---

Octopress Blog: Blog for geek!
```

You can add a single category or multiple categories like this.

```
# One category
categories: Sass

# Multiple categories example 1
categories: [CSS3, Sass, Media Queries]

# Multiple categories example 2
categories:
- CSS3
- Sass
- Media Queries
```

Please remember that when writing a post, you can add an HTML comment `<!--more-->` to split the post for an excerpt. Only the first section of the post, before the comment, will show up on the blog index.

#### Push It to Github

Now you've add a new article. Push it remotely:

```
# Do this to preview whether it works well or not
rake preview
# Deploy it to github page
rake gen_deploy
# Do these to commit source
git add . # add all useful files you want to commit
git commit -m 'My first article' # commit it locally
git push origin source # push source remotely
```

Three important folders:

* source: source files, edit in this folder
* public: local preview
* _deploy: remote repository

A common workflow is like that:

1. Add new article
2. Edit this article
3. Preview it
4. Deploy it
5. Commit the source branch



### Deploy somewhere else



It is very common that you want to post articles on another computer. You can follow this:

``` bash
git clone git@github.com:lufeihaidao/lufeihaidao.github.com.git # clone your code
bundle install
git checkout source # switch to source branch
git clone git@github.com:lufeihaidao/lufeihaidao.github.com.git # clone your code
mv lufeihaidao.github.com/ _deploy # for gen_deploy
```

Or you can simply run `rake setup_github_pages` in an octopress copy and fill in your github pages repository url.

Then please feel free to follow the workflow.

Next possible post: octopress plugins or themes.



