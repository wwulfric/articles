---
title: 服务器和本地的文件同步
date: 2014-03-25 12:19
tags: [samba, 远程, remote, server, rsync, ubuntu, centos] 
categories: [技术]
---

在开发的时候经常会有这样的情形：在本地做 Git 操作和代码编辑工作，然后在远程服务器/虚拟机上做调试，这样可以保持本地环境的整洁。有以下几种待选方案：

- 利用 samba 将远程文件夹挂载到本地
- 利用 Rsync/sftp 或其它工具建立本地文件夹和远程文件夹的双向同步
- 利用 Rsync/sftp 将本地文件夹同步到远端

<!--more-->

## Samba

![Samba](http://wulfric.qiniudn.com/samba.png "Samba")

在网速比较好的情况下，samba 是一个比较优秀的解决方案。它将远程的文件夹加载到本地，可以保持本地和远程文件内容的一致性。

本文安装方法参考了[这篇文章](https://rbgeek.wordpress.com/2012/05/25/how-to-install-samba-server-on-centos-6/)。以下配置可以在虚拟机安装时通过脚本自动完成。

### 安装

~~~ bash
cd ~
# 安装 samba
sudo apt-get install samba
# yum: sudo yum install samba
# 备份 samba 配置文件
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf_back
# 修改配置文件，具体内容见下文。
sudo vim /etc/samba/smb.conf
# 或者 sudo rm /etc/samba/smb.conf && sudo touch /etc/samba/smb.conf 新建一个配置文件

# 关闭 selinux
sudo vim /etc/selinux/config
# 设置：SELINUX=disabled

# 配置 iptables
sudo iptables -I INPUT 4 -m state --state NEW -m udp -p udp --dport 137 -j ACCEPT
sudo iptables -I INPUT 5 -m state --state NEW -m udp -p udp --dport 138 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -m tcp -p tcp --dport 139 -j ACCEPT
sudo service iptables save
sudo service iptables restart
~~~

### 普通配置

如果服务器是在信任的局域网中，那么可以跳过权限验证，使用简单的guest 来挂载。

```shell
mkdir guest_share
ls -l
# 通过查看可以知道 guest_share 的权限是 775，guest 没有写权限，这会导致本地只能读而不能写
# 修改权限
sudo chmod -R 777 guest_share
```

编辑配置文件。

```shell
# /etc/samba/smb.conf

#======================= Global Settings =====================================
[global]
 workgroup = WORKGROUP
 security = share
 map to guest = bad user
#============================ Share Definitions ==============================
[MyShare]
 path = /home/username/guest_share
 browsable =yes
 writable = yes
 guest ok = yes
 read only = no
```

重启。

```shell
# 查看配置文件的内容有没有错误，如果有重要的配置丢失，可能是因为缩进不对
sudo testparm
sudo service smb restart
sudo service nmb restart
```

在本地获取该共享文件夹

- MAC: finder -> 前往 -> 连接服务器 -> smb://你的IP/share

- WINDOWS: 资源管理器 -> 输入 \\\你的IP\share

其中 share 是配置文件中的配置名：如上所述的[MyShare]

### 安全配置

如果是远程的服务器，安全起见，最好还是能够用帐号和密码登录。

```shell
# 创建组
sudo groupadd smbgrp
# 创建用户
sudo useradd smbuser
# 更改用户密码
sudo passwd smbuser
# 用户添加到组
sudo usermod -a -G smbgrp smbuser
# 添加 samba 的登录密码
sudo smbpasswd -a smbuser
# 创建要分享的文件夹
cd ~
mkdir secure_share
sudo chown -R smbuser:smbgrp secure_share
sudo chmod -R 770 secure_share
```

编辑配置文件，注意差别，security 从 share 改为 user[^share_level]。

```shell
# /etc/samba/smb.conf

#======================= Global Settings =====================================
[global]
 workgroup = WORKGROUP
 security = user
 map to guest = bad user
#============================ Share Definitions ==============================
[MyShare]
 path = /home/username/guest_share
 browsable =yes
 writable = yes
 guest ok = yes
 read only = no
[Secure]
 path = /home/username/secure_share
 valid users = @smbgrp
 guest ok = no
 writable = yes
 browsable = yes
```

然后检查参数并重启 samba。在本地选择使用注册用户登录 samba。

### 缺点

在本地新建和删除文件的时候略有卡顿，对网络的稳定性要求较高。

## Rsync

本地和服务器都需要安装 Rsync。这种工作方式对网络的实时性需求不是很高，而且相对于 sftp，rsync 是增量更新，对带宽的需求更小。

![rsync](http://wulfric.qiniudn.com/rsync.jpg)

一般编辑器都有 rsync 的插件，对于 sublime text 就是 [RSync](https://sublime.wbond.net/packages/RSync)。

值得注意的是，MAC 的系统是 Case-insenstive 的，因此使用 git 作版本控制时，重命名操作会出现问题。一个解决方案是，利用 MAC 自带的 Disk Utility 工具，创建一个 Case-senstive 的 new image[^case_sensitivity]。

[^case_sensitivity]: 参见[讨论](http://stackoverflow.com/questions/8904327/case-sensitivity-in-git)

[^share_level]: samba 有多种 [share level](https://www.samba.org/samba/docs/man/Samba-HOWTO-Collection/ServerType.html)