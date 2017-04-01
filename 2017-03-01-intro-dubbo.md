---
title: dubbo 简介与 dubbo demo 运行
date: 2017-03-01 18:40
categories: [技术]
tags: [ 分布式, 微服务, dubbo, zookeeper, dubbo-admin, dubbo-monitor, dubbo-demo]
---

dubbo是阿里巴巴推出的分布式服务治理框架，是国内实现微服务较为常见的框架。关于微服务的介绍可以参见[原文](http://martinfowler.com/articles/microservices.html)，[infoq 文章](http://www.infoq.com/cn/articles/basis-frameworkto-implement-micro-service)，[dubbo 架构设计详解](http://shiyanjun.cn/archives/325.html)。

dubbo 的架构图如下[^dubbo-architecture]：

![dubbo 架构](http://wulfric.qiniudn.com/dubbo/dubbo-architecture.png "dubbo 架构")

如上图所示，简单来说 dubbo 架构包括如下几部分：服务注册和服务发现中心，对外暴露服务的服务提供方和运行该服务的容器，调用远程服务的服务消费方，统计服务调用时间和次数的监控中心（非必需）。其调用关系为：

```
0. 服务方的容器启动服务
1. 服务提供方向注册中心注册自己的地址
2. 访问消费方向注册中心订阅所需的服务
3. 注册中心通知服务消费方所订阅服务的变更
4. 根据获取到的提供方地址列表，服务消费方直接调用服务提供方
5. 监控中心会监控消费方和提供方的调用时间和次数等信息
```

## 安装与准备

dubbo 采用全 spring 配置方式，透明化接入应用，对应用没有侵入。

如上框图所示，你需要准备注册中心，dubbo 服务的提供者和消费者，以及监控中心。运行一个 demo 程序需要准备这些：

- 注册中心 zookeeper
- 当当的 dubbox
- tomcat
- 示例  dubbo demo 的 provider，consumer
- dubbo admin 管理程序
- dubbo simple monitor 监控程序

其中 dubbo 的部分都在 dubbox 项目中。这些软件的下载只需要下载源码，不会安装到本地。

在 zookeeper 官网选择国内镜像下载 [zookeeper](https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/)，解压到本地。我下载的是 3.4.9 版本。

在 tomcat 官网下载国内镜像 [tomcat](https://tomcat.apache.org/download-80.cgi)，解压到本地。我下载的是 8.5.11 版本。

clone 下当当的 [dubbox 项目](https://github.com/dangdangdotcom/dubbox)，执行：

```shell
mvn install -Dmaven.test.skip=true #dubbox 的测试 url 无法访问
```

安装。

查看 dubbox 的文件结构，我们在后面还需要用到 dubbo-admin，dubbo-demo 和 dubbo-simple/dubbo-monitor-simple。

## 运行

### zookeeper 注册中心

进入解压后的 zookeeper 目录，创建 conf/zoo.cfg（可参考同目录下的 sample 文件），其内容如下：

```shell
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/tmp/zookeeper/tmp/data # 自己定义
dataLogDir=/tmp/zookeeper/tmp/log # 自己定义
clientPort=2181
server.1=localhost:2287:3387
```

执行`bin/zkServer.sh start`启动 zookeeper，执行`bin/zkCli.sh -server 127.0.0.1:2181`查看是否启动成功。

### dubbo 提供者和消费者服务

Demo 在 dubbo-demo 文件夹下，因为前面在 dubbox 下执行过 mvn install，都已经打包好了，这里可以直接执行来启动 demo provider, consumer，如下图所示。

![dubbo demo consumer](http://wulfric.qiniudn.com/dubbo/R-dubbo-demo-consumer.png "dubbo demo consumer")

当 provider 执行后看到

```shell
[01/03/17 06:16:26:026 CST] main  INFO container.Main:  [DUBBO] Dubbo SpringContainer started!, dubbo version: 2.0.0, current host: 127.0.0.1
[2017-03-01 18:16:26] Dubbo service server started!
```

consumer 执行后看到

```shell
Client response filter invoked
Reader interceptor invoked
Dynamic reader interceptor invoked
[01/03/17 06:18:11:011 CST] main  INFO support.LoggingFilter:  [DUBBO] The contents of request body is: 
{"id":1,"username":"username1"}
, dubbo version: 2.0.0, current host: 192.168.1.33
SUCCESS: got user User (id=1, name='username1')
[01/03/17 06:18:11:011 CST] main  INFO container.Main:  [DUBBO] Dubbo SpringContainer started!, dubbo version: 2.0.0, current host: 192.168.1.33
[2017-03-01 18:18:11] Dubbo service server started!
```

说明启动成功了。

### tomcat 运行 dubbo admin

在 dubbo-admin 的 target 目录下找到 war 包（没有的话就 mvn package 一下），放到 tomcat 目录下的webapps 下。

```shell
~/tmp/zookeeper/apache-tomcat-8.5.11

> ls webapps

ROOT                  docs                  dubbo-admin-2.8.4     dubbo-admin-2.8.4.war examples              host-manager          manager
```

执行`bin/startup.sh`启动 tomcat，打开 localhost:8080，一开始会有点慢，tomcat 要解压 war 文件，解压完毕后 webapps 下出现对应的文件夹（如上面 ls 所示），进入该文件夹，

```shell
~/tmp/zookeeper/apache-tomcat-8.5.11

> vim webapps/dubbo-admin-2.8.4/WEB-INF/dubbo.properties
```

编辑 dubbo-admin 的 zookeeper 配置。

```shell
dubbo.registry.address=zookeeper://127.0.0.1:2181
# 帐号对应的密码，初次访问的时候需要用到
dubbo.admin.root.password=root
dubbo.admin.guest.password=guest
```

重启 tomcat，打开 http://localhost:8080/dubbo-admin-2.8.4，输入帐号密码即可看到 dubbo-admin 的管理页面。上面运行的 provider 和 consumer demo 在服务治理下。

![dubbo admin](http://wulfric.qiniudn.com/dubbo/dubbo-admin.png "dubbo admin")

注意：有些链接可能 404，因为 dubbo-admin 默认使用了`/`路径，而挂在 tomcat 下的时候路径中包含了`/dubbo-admin-2.8.4`。

### 启动 dubbo monitor

除了这三者之外，dubbo 还提供了监控面板。在 dubbo-simple/dubbo-monitor-simple 的 target 目录下，解压 .tar.gz 文件，得到 dubbo-monitor-simple-2.8.4 文件夹，编辑其中的 conf/dubbo.properties 文件。

```shell
dubbo.container=log4j,spring,registry,jetty
dubbo.application.name=simple-monitor
dubbo.application.owner=
# 选择 zookeeper 作为注册中心
# dubbo.registry.address=multicast://224.5.6.7:1234
dubbo.registry.address=zookeeper://127.0.0.1:2181
#dubbo.registry.address=redis://127.0.0.1:6379
#dubbo.registry.address=dubbo://127.0.0.1:9090
dubbo.protocol.port=7070
# 监控中心的端口号
dubbo.jetty.port=8083
dubbo.jetty.directory=${user.home}/monitor
dubbo.charts.directory=${dubbo.jetty.directory}/charts
dubbo.statistics.directory=${user.home}/monitor/statistics
dubbo.log4j.file=logs/dubbo-monitor-simple.log
dubbo.log4j.level=WARN
```

需要修改的是 registry，端口号和各种文件的保存位置。

执行`bin/start.sh`，当看到

```shell
Starting the simple-monitor .........................OK!
PID: 21032
STDOUT: logs/stdout.log
```

即执行成功，打开 http://localhost:8083/ 即可看到监控面板。

![dubbo monitor](http://wulfric.qiniudn.com/dubbo/dubbo-monitor.png "dubbo monitor")

如果看不到 charts 和 statitics，检查下配置中的`dubbo.charts.directory`和`dubbo.statistics.directory`是否提前创建成功，dubbo-monitor 可能不会自动创建该目录的。

自带 monitor 比较简单，可以参见 monitor 的其他实现：[韩都衣舍/dubbo-monitor](http://git.oschina.net/handu/dubbo-monitor)，[dubboclub/dubbokeeper](https://github.com/dubboclub/dubbokeeper)。

PS: 从零开始搭一个 dubbo 的 demo 示例可以参见[下一篇文章](/2017/03/dubbo-demo-test/)。

[^dubbo-architecture]: 见官方[原图](http://dubbo.io/dubbo-architecture.jpg-version=1&modificationDate=1330892870000.jpg)