---
title: Ruby on Rails 开发和生产环境搭建
date: 2016-03-21 16:37
categories: [技术]
tags: [ubuntu, ruby, rbenv, nginx, passenger, mina, rails, postgresql]
---



准备搭生产环境做一些玩具（ubuntu ruby rails nginx passenger mina postgresql），参考了阿里云，linode 和 digital ocean。DO 两三百的延迟实在不能忍受，ping 了下阿里只有 2ms，那就选这个吧！

![使用优惠码可以打九折 dj7mxg](http://static.wulfric.me/aliyun-youhuima.jpg "aliyun-youhuima")

因为是自己的个人机器，所以就按照自己熟悉的 ubuntu 来配了。用 rbenv 来管理安装 ruby，用 passenger with nginx 作服务器，postgresql 作数据库，mina 来部署应用。


## 生产环境配置

### ssh 配置

先在本地开发环境下`ssh-keygen`，生成登录服务器的公钥私钥[^4]。然后在服务器端

``` bash
mkdir .ssh
vi ~/.ssh/authorized_keys
# 把刚才生成的公钥内容复制进去
```

尝试在本地登录，如果跳过了密码验证，说明配置正确。如果没有跳过，检查一下服务器上`.ssh`和`.ssh/authorized_keys`的权限：

``` bash
chmod 700 .ssh
chmod 600 .ssh/authorized_keys
```

为了让服务器更加安全，我们可以更改ssh登录端口，并拒绝密码登录。在服务器端`vi /etc/ssh/sshd_config`做如下修改

``` bash
Port xxx
PasswordAuthentication no
```

然后执行`service ssh restart`使配置生效。

PS: 可以在本地配置`.ssh/config`如下，方便登录。

``` bash
Host nick-name
  HostName        xxx.xxx.xxx.xxx
  Port            xxx
  User            someuser
  IdentityFile    ~/.ssh/some_private_key
```

之后本地只要`ssh nick-name`即可登录服务器。

更详细的配置可以参考[这里](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04)。

### 创建生产环境用户

创建一个用户，作为生产环境的操作者，生产环境的工具都通过这个用户来安装和配置[^1][^3]。`adduser wwwroot`，按照提示输入密码等信息。

`vi /etc/sudoers`，在文件中添加一行`wwwroot ALL=(ALL:ALL) ALL`然后`wq!`保存退出，给 wwwroot 用户添加 sudo 权限。

`su - wwwroot`切换到 wwwroot 帐号，同上创建 `~/.ssh/authorized_keys` 文件并复制公钥。

### 安装 git

一般发行版都会自带 git，不过我选择安装较新的版本。

``` bash
sudo add-apt-repository ppa:git-core/ppa
sudo apt-get update
sudo apt-get install git
```

### rbenv 安装 ruby
rbenv 是一个非常优秀的 ruby 版本管理工具，每个 ruby 版本都是编译安装在用户家目录下`.rbenv`文件夹下，相关 gem 也是安装在这个文件夹下，便于移动和删除。在 mac 下只需简单的`brew install rbenv`即可，在 linux 下需要 clone 代码并配置你的`.bashrc`或`.zshrc`文件，安装命令如下（参考 [rbenv install](https://github.com/rbenv/rbenv#installation) 和 [ruby-build install](https://github.com/rbenv/ruby-build#installation)）：

``` bash
# 安装 rbenv
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
# 加速 rbenv，非必需
cd ~/.rbenv && src/configure && make -C src
# 加入 PATH （ubuntu）
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
# 安装 ruby-build
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
# source .bashrc or source .zshrc
source .bashrc
# rbenv install -l
ruby_version="$(curl -sSL http://ruby.thoughtbot.com/latest)"
# install ruby 需要依赖一些库（如果缺失，执行 rbenv install 的时候会提示）
sudo apt-get install -y libssl-dev libreadline-dev zlib1g-dev
rbenv install "$ruby_version"
rbenv global "$ruby_version"
# 安装 bundler
gem install bundler
```

至此，ruby 安装完毕。`which ruby && ruby --version`查看安装结果。

PS：如果你的服务器在国内（比如用的阿里云国内节点），修改 `.bundle/config` 和 `.gemrc` 使用国内镜像。

```bash
# .bundle/config
---
BUNDLE_MIRROR__HTTPS://RUBYGEMS__ORG/: https://ruby.taobao.org

# .gemrc
---
:backtrace: false
:bulk_threshold: 1000
:sources:
- https://ruby.taobao.org/
:update_sources: true
:verbose: true
gem: "--no-document"
```

### 安装 passenger with nginx

passenger 官方有很详细的安装[教程](https://www.phusionpassenger.com/library/install/nginx/install/oss/trusty/)，可以作为主要的参考。官方教程主要分为三部分，standalone, with nginx 和 with apache，区别见[博文](https://www.phusionpassenger.com/library/indepth/integration_modes.html)。我们选择安装 passenger with nginx。

要安装 passenger with nginx，nginx 需要加上 passenger 模块重新编译安装，所以无论是否已经安装过 nginx，nginx 都需要重新安装。首先按照官方教程安装，如下：

```bash
# Install our PGP key and add HTTPS support for APT
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 561F9B9CAC40B2F7
sudo apt-get install -y apt-transport-https ca-certificates

# Add our APT repository
sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger trusty main > /etc/apt/sources.list.d/passenger.list'
sudo apt-get update

# Install Passenger + Nginx
sudo apt-get install -y nginx-extras passenger
# 网络不太好，可以多 update 几次
```

完成之后，执行`passenger-install-nginx-module`，它会提醒你重新编译安装 nginx，按照提示一路安装过去即可。安装完毕，执行`sudo /usr/bin/passenger-config validate-install`检查是否安装成功。

新安装的 nginx 位于`/opt/nginx`，查看帮助`sudo /opt/nginx/sbin/nginx -h`。

参照[官方说明](https://www.phusionpassenger.com/library/deploy/nginx/deploy/ruby/)配置 nginx 部署网站。我的例子如下。


``` bash
server {
    listen 80;
    # 如果不支持 ipv6，这里需要注释掉
    listen [::]:80;

    server_name domain_name.com;

    # Tell Nginx and Passenger where your app's 'public' directory is
    root /var/www/html/mywebsite/current/public;

    # Turn on Passenger
    passenger_enabled on;
    passenger_ruby /home/wwwroot/.rbenv/versions/2.3.0/bin/ruby;
    location ~ ^(/assets) {
      access_log off;
# 设置 assets 下面的浏览器缓存时间为最大值（由于 Rails Assets Pipline 的文件名是根据文件修改产生的 MD5 digest 文件名，所以此处可以放心开启）
	  expires max;
	}
}
```

虽然配置好了，但是还需等待本地配置完成并部署成功之后才能看到效果。


### 安装 postgresql

ubuntu 14.04 的默认版本是 9.3，我要安装的版本是 9.4[^5]，参考 [How install postgresql 9.4](http://askubuntu.com/questions/633919/how-install-postgresql-9-4)。

``` bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install postgresql-9.4 libpq-dev
# 后者在 gem install pg 的时候需要用到
```

安装完毕开始配置[^2]。

``` bash
# 当服务器登录用户名和 postgresql 用户名相同时，postgresql 默认可以无需密码登录，所以我们先创建一个 wwwroot 用户
sudo -u postgres createuser --superuser $USER
# 以 postgres 身份进入数据库
sudo -u postgres psql
# 给刚刚生成的 wwwroot 用户设置密码。注意这个时候是在数据库的终端下，没有 $USER 变量，需要指定为 wwwroot
postgres=# \password wwwroot
# 创建同名数据库
sudo -u postgres createdb $USER
# 这个时候登录数据库就非常容易了
psql
# 创建一个新的数据库
create database xxxxxxdb;
```

## 开发环境配置

本地配置 Ruby on Rails，同开发环境。

### 安装 mina 部署网站

本地安装 mina`gem install mina`，调整 Gemfile。执行`mina init`生成 `deploy.rb`并编辑。默认生成的文件里有很多注释，参照更改即可。其他可以参考的[配置](https://github.com/mina-deploy/mina/blob/master/test_env/config/deploy.rb)。

编辑完成之后执行`mina setup`，远程服务器会完成文件结构等的设置。远程目录结构为`current  last_version  releases  scm  shared  tmp`，其中 releases 是发布的所有版本，current 是软链接，指向 releases 中当前版本。shared 文件夹放的是一些共享的配置文件。根据提示到远程创建`shared/config/database.yml`和`shared/config/secrets.yml`。

``` bash
# database.yml
default: &default
  adapter: postgresql
  pool: 5
production:
  <<: *default
  database: xxxxxxdb
```

``` bash
# secrets.yml
production:
  secret_key_base: xxxx...
```

接下来就可以执行`mina deploy`把代码部署到服务器了。deploy 在远程服务器安装 gem 的时候可能会报错，多半是服务器没有 js 运行时或者相关依赖库导致，按照提示安装就好了。如果是 pg 安装问题，可以配置 bundler：

``` bash
bundle config build.pg --with-pg-config=/usr/pgsql-9.4/bin/pg_config
```

PS: 更新了[Rails 应用生产环境利用  Active Job 和 Action Cable
实现消息推送](/2016/09/active-job-action-cable-in-rails)

[^1]: 参考[在 Aliyun 上快速部署 Ruby on Rails](https://ruby-china.org/topics/17553)
[^2]: 参考[PostgreSQL 服务器设置](https://help.ubuntu.com/community/PostgreSQL#Basic_Server_Setup)
[^3]: 参考[Deploying rails app to DO ubuntu droplet with Nginx/Passenger/Mina](http://railsr.github.io/posts/nginx-passenger-mina-deploy)
[^4]: 参考 [github](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/)、[archlinux](https://wiki.archlinux.org/index.php/SSH_keys_(简体中文)) 的文档
[^5]: centos 下安装 passenger 和 postgresql 的方法稍有不同，参见 [CentOS6下最新版PostgreSQL的安装及设置-Racksam](http://www.racksam.com/2016/04/02/install-postgresql9-with-yum-on-centos6)、[Installing PostgreSQL 9.4 And phpPgAdmin In CentOS 7/6.5/6.4-Unixmen](https://www.unixmen.com/postgresql-9-4-released-install-centos-7)、[Installing Passenger-Nginx on CentOS 6 with RPM](https://www.phusionpassenger.com/library/install/nginx/install/oss/el6)。

