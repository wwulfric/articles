title: 服务器和本地的文件同步--samba
date: 2014-03-25 12:19
tags: [samba, 远程, 本地] 
categories: [技术]
---

![Samba](http://bcs.duapp.com/blog-image-bed/samba.png Samba)

在开发的时候经常会有这样的情形：在本地做 Git 操作和代码编辑工作，然后在远程服务器/虚拟机上做调试，这样可以保持本地环境的整洁。有以下几种待选方案：

- 将远程文件夹挂载到本地
- 利用 Rsync/sftp 或其它工具建立本地文件夹和远程文件夹的双向同步
- 利用 Rsync/sftp 将本地文件夹同步到远端

在网速比较好的情况下，samba 是一个比较优秀的解决方案。它将远程的文件夹加载到本地，可以保持本地和远程文件内容的一致性。

<!--more-->

## 安装

``` bash
cd ~
mkdir share
# 安装 samba
sudo apt-get install samba
# 修改 samba 配置文件
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf_back
sudo vim /etc/samba/smb.conf
```

## 配置

``` bash
# 注意添加的位置；注意和原文的缩进保持一致
# 在“# security = user”这行的下面添加： 
security = user 
username map = /etc/samba/smbusers
# 在 "workgroup = WORKGROUP" 下面添加如下三行： 
display charset = UTF-8 
unix charset = UTF-8 
dos charset = cp936
# 在文件的最后添加如下几行： 
[Share]
comment = Shared Folder with username and password 
path = /home/你的用户名/share 
public = yes 
writable = yes 
valid users = 你的用户名 
create mask = 0664
directory mask = 0775
force user = 你的用户名 
force group = 你的组 
available = yes 
browseable = yes
```

## 权限修正

``` bash
# 增加 samba 的对应用户的密码
sudo smbpasswd -a 你的用户名
# 把该用户加入 samba 管理
sudo vim /etc/samba/smbusers
# 创建用于同步的文件夹
你的用户名 = "network username"
# 改权限
sudo chmod 777 ~/share
# 最重要的是确保你在服务器中加到该文件夹的文件的权限应是对外可访问的
```

## 重启

``` bash
sudo testparm #测试 samba 参数是否正确
sudo restart smbd
sudo restart nmbd
```

## 本地获取该共享文件夹

- MAC: finder -> 前往 -> 连接服务器 -> smb://你的IP/share

- WINDOWS: 资源管理器 -> 输入 \\\你的IP\share

其中 share 是配置文件中的配置名：如上所述的[Share]

## 缺点

在本地新建和删除文件的时候略有卡顿

## 可选

一般在本地开发机器上编辑的时候，有时候操作系统或编辑器会生成一些对项目来说不必要的文件，比如 vim 的 .swp，mac 下的 ._.DS_Store 等，可用 gitignore 将之忽略

以上配置可以在虚拟机安装时通过脚本自动完成
