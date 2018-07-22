---
title: nginx 502 和 504 超时演示
date: 2018-07-19 19:26
categories: [技术]
tags: [nginx, php-fpm, php]
---

最近线上 nginx 遇到了一些较难排查的 502 和 504 错误，顺便了解了一下 nginx 的相关配置。我发现网上很多介绍 nginx 超时配置只是列了这几个配置的含义和数值，并没有解释什么原因会触发哪个配置。因此趁这个机会演示一下，如何让 nginx 符合预期正确出现 502 和 504。

## 502 和 504 的解释

在 http status 的 [定义](https://www.wikiwand.com/en/List_of_HTTP_status_codes) 中：

502 Bad Gateway: The server was acting as a [gateway](https://www.wikiwand.com/en/Gateway_(telecommunications)) or proxy and received an invalid response from the upstream server. 

504: he server was acting as a gateway or proxy and did not receive a timely response from the upstream server. 

502 的错误原因是 Bad Gateway，一般是由于上游服务器的故障引起的；而 504 则是 nginx 访问上游服务超时，二者完全是两个意思。但在某些情况下，上游服务的超时（触发 tcp reset）也可能引发 502，我们会在之后详述。

## 演示环境

你需要 3 个逻辑组件：nginx 服务器，php-fpm，client 访问客户端。3 个组件可以在同一台机器中，我用的是 docker 来配置 PHP 和 nginx 环境，在宿主机上访问。如果你很熟悉这 3 个组件，这部分可以跳过。用 docker 来做各种测试和实验非常方便，这里就不展开了。docker-compose 的配置参考了这篇[文章](http://geekyplatypus.com/dockerise-your-php-application-with-nginx-and-php7-fpm/)。我的 docker composer 文件如下：

```yaml
version: '3'
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./code:/code
      - ./nginx/site.conf:/etc/nginx/conf.d/site.conf
    depends_on:
      - php
  php:
    image: php:7.1-fpm-alpine
    volumes:
      - ./code:/code
      - ./php/php-fpm.conf:/usr/local/etc/php-fpm.conf
```

使用的镜像都是基于 [alpine](https://hub.docker.com/_/alpine/) 制作的，非常小巧：

```shell
REPOSITORY  TAG               SIZE
php         7.1-fpm-alpin     69.5MB
nginx       alpine            18.6MB
```

nginx 的配置：

```nginx
server {
  index index.php index.html;
  server_name php-docker.local;
  error_log  /var/log/nginx/error.log;
  access_log /var/log/nginx/access.log;
  root /code;

  location ~ \.php$ {
    try_files $uri =404;
    fastcgi_split_path_info ^(.+\.php)(/.+)$;
    fastcgi_pass 127.0.0.1:9000;
    fastcgi_index index.php;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param PATH_INFO $fastcgi_path_info;
    fastcgi_connect_timeout 5s;
    fastcgi_read_timeout 8s;
    fastcgi_send_timeout 10s;
  }
}
```

php-fpm 的配置

```shell
[global]
include=etc/php-fpm.d/*.conf
request_terminate_timeout=3s
```

代码放在 [github](https://github.com/wwulfric/nginx-timeout-demo)。

## 关键参数

在这个演示中，PHP 的关键参数有两个，一个是 PHP 脚本的 max_execution_time，这个配置在`php.ini`中；另一个是 php-fpm 的 request_terminate_timeout，在`php-fpm.conf`中。当以 php-fpm 提供服务时，request_terminate_timeout 设置会覆盖 max_execution_time 的设置，因此我们这里只测试 request_terminate_timeout。

request_terminate_timeout 的意思是 php-fpm 接受的请求的超时时间，超过这个时间 php-fpm 会 kill 掉执行脚本的 worker 进程。

nginx的关键参数是 fastcgi 相关的 timeout，即：fastcgi_connect_timeout，fastcgi_read_timeout，fastcgi_send_timeout。

这几个 nginx 参数的主语都是 nginx，所以 fastcgi_connect_timeout 的意思是 nginx 连接到 fastcgi 的超时时间，fastcgi_read_timeout 是 nginx 读取 fastcgi 的内容的超时时间，fastcgi_send_timeout 是 nginx 发送内容到 fastcgi 的超时时间。

## 演示过程

首先启动 nginx 和 PHP：

```sh
docker-compose up
```

在 code 文件夹下添加一个 index.php 文件：

```php
<?php
sleep(70);
echo 'hello world';
```

### 上游服务主动 reset

访问 php-docker.local:8080/index.php，报错 502 bad gateway。而且是在 3s 之后报的错，说明触发了 request_terminate_timeout 设置，php-fpm 关闭了连接。

通过观察 `ps aux | grep php` 可以发现，php-fpm 是通过杀掉超时的进程来解决进程超时问题的（pid 每次有一个会变化，说明一个进程杀掉了，并启动了另一个进程。这和 php-fpm 的进程池设定有关，你的设定未必会重新启动一个新的进程）。

```bash
/var/www/html # ps aux | grep php
    1 root       0:00 php-fpm: master process (/usr/local/etc/php-fpm.conf)
    6 www-data   0:00 php-fpm: pool www
    7 www-data   0:00 php-fpm: pool www
/var/www/html # ps aux | grep php
    1 root       0:00 php-fpm: master process (/usr/local/etc/php-fpm.conf)
    7 www-data   0:00 php-fpm: pool www
   17 www-data   0:00 php-fpm: pool www
/var/www/html # ps aux | grep php
    1 root       0:00 php-fpm: master process (/usr/local/etc/php-fpm.conf)
   17 www-data   0:00 php-fpm: pool www
   20 www-data   0:00 php-fpm: pool www
```

在这种情况下，nginx 日志中的错误是：

````
recv() failed (104: Connection reset by peer) while reading response header from upstream
````

即连接被服务端（PHP）reset 了，也就很好理解了。

注意，在这种情况下，php-fpm 的日志中也会记录的：

```
php_1  | [18-Jul-2018 16:33:42] WARNING: [pool www] child 5, script '/code/index.php' (request: "GET /index.php") execution timed out (3.040130 sec), terminating
php_1  | [18-Jul-2018 16:33:42] WARNING: [pool www] child 5 exited on signal 15 (SIGTERM) after 30.035736 seconds from start
php_1  | [18-Jul-2018 16:33:42] NOTICE: [pool www] child 8 started
```

这也是可以发现问题的一个地方。

### nginx 读取上游服务超时

删掉 request_terminate_timeout 配置，重启应用：

```sh
docker-compose down && docker-compose up
```

此时，PHP 脚本将要执行 70s，肯定超过 nginx 设置的超时时间，get 一下发现确实如此，8s 之后抛出 504 Gateway Time-out 错误，nginx 日志是：

```
upstream timed out (110: Operation timed out) while reading response header from upstream
```

说明触发了 fastcgi_read_timeout 设置。

### 关闭上游服务

关掉 PHP 服务：

```sh
docker-composer stop php
```

PHP 服务停掉之后第一次访问，得到 504 错误，错误是：

```
upstream timed out (110: Operation timed out) while connecting to upstream
```

超时时间为 fastcgi_connect_timeout 的设置。说明这个时候 tcp 连接还在，但是尝试连接的时候失败了。

再次访问，得到 502 错误，错误是：

```
connect() failed (113: Host is unreachable) while connecting to upstream
```

502 的原因很容易理解，上游服务挂了，同时因为之前访问的时候发现连接不上就把连接断掉了，再次连接的时候便无法找到 host 了。

我曾怀疑第一次访问 504 是由于 keepalive。但我停掉 PHP 之后隔了好久才发第一个请求，仍然是这个结果。

如果将 nginx fastcgi_pass 配置为 127.0.0.1:9000（本地没有这个端口），则马上就会抛出 502 错误，错误为：

```
connect() failed (111: Connection refused) while connecting to upstream
```

登入 nginx 服务，使用 tcpdump 监控 9000 上的通信：

```sh
tcpdump -i eth0 -nnA tcp port 9000
# 如果你的 PHP 在本地，eth0 应该改成 lo
```

我们发现，当 PHP 关闭之后第一次访问，nginx 会尝试向 PHP 发起若干次 TCP SYN 请求，但 PHP 显然不会响应，这个时候 nginx 就返回了 504。第二次访问的时候 nginx 根本不会发起任何请求，直接 502 了[^2]。如果我们这个时候执行`nginx -t`会发现，nginx 已经认为配置文件有问题了：nginx: configuration file /etc/nginx/nginx.conf test failed。

### 换一种配置

[这篇文章](https://sandro-keil.de/blog/2017/07/24/let-nginx-start-if-upstream-host-is-unavailable-or-down/) 提到，我们之前的 nginx 配置并不合理[^1]，我们重新设置 nginx：

```nginx
server {
  index index.php index.html;
  server_name php-docker.local;
  error_log  /var/log/nginx/error.log;
  access_log /var/log/nginx/access.log;
  root /code;
  resolver 127.0.0.11;  # here
  location ~ \.php$ {
    set $upstream php:9000; # here
    try_files $uri =404;
    fastcgi_split_path_info ^(.+\.php)(/.+)$;
    fastcgi_pass $upstream;  # here
    fastcgi_index index.php;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param PATH_INFO $fastcgi_path_info;
    fastcgi_connect_timeout 5s;
    fastcgi_read_timeout 8s;
    fastcgi_send_timeout 10s;
  }
}
```

其中 127.0.0.11 是 docker 的内网 dns resolver。该配置动态指定 fastcgi pass，所以 nginx 不会检查该连接能否建立起来。

按照这个配置启动，先访问 index.php 建立连接，然后关闭 PHP，表现为：

在 keepalive 期间，抛出 504 错误，超时时间为 fastcgi_connect_timeout，错误是：

```
upstream timed out (110: Operation timed out) while connecting to upstream
```

keepalive 断线之后，抛出 502 错误，超时时间不定，错误是：

```
connect() failed (113: Host is unreachable) while connecting to upstream
```

按照[这篇文章](https://sandro-keil.de/blog/2017/07/24/let-nginx-start-if-upstream-host-is-unavailable-or-down/)所说，这种配置 nginx 不会认为有问题，执行`nginx -t`确实如此。在 **一段时间** 内，每次请求 nginx 都会向 upstream 发送 SYN，这段时间的状态码都是 504，之后再访问就不再发 TCP 包，状态码也变成 502。

### 其他

除此之外，PHP 脚本还有一个超时时间的设置：max_execution_time。它是限制 PHP 脚本的执行时间，但这个时间不会计算系统调用（比如 sleep，io，等）。因为该原因导致 PHP 杀掉进程时，会抛出 fatal error，而 php-fpm 不会有 fatal error。

这里实验使用的是 PHP 的 fastcgi 工作方式，如果是 nginx 连接上游服务器的话，fastcgi_connect_timeout，fastcgi_read_timeout，fastcgi_send_timeout 都需要替换成对应的 proxy_connect_timeout，proxy_read_timeout，proxy_send_timeout。

## 结论

504 的原因比较简单，一般都是上游服务的执行时间超过了 nginx 的等待时间，这种情况是由于上游服务的业务太过耗时导致的，或者连接到上游服务器超时。从上面的实验来看，后者的原因比较难以追踪，因为这种情况下连接是存在的，但是却连不上，好在这种 504 一般都会在一段时间后转为 502。

502 的原因是由于上游服务器的故障，比如停机，进程被杀死，上游服务 reset 了连接，进程僵死等各种原因。在 nginx 的日志中我们能够发现 502 错误的具体原因，分别为：`104: Connection reset by peer`，`113: Host is unreachable`，`111: Connection refused`。

有一些细节上的差别和 nginx 的工作原理有关，这部分尚未深挖。

[^1]: [这篇文章](https://sandro-keil.de/blog/2017/07/24/let-nginx-start-if-upstream-host-is-unavailable-or-down/) 表明，我们之前的设置中，如果 PHP 没有先启动起来，那么 nginx 也是启动不起来的，这种设置并不合理：nginx 的一台上游服务有问题，结果 nginx 就无法提供服务了。但这和我们的演示关系不大，因此并没有在正文中过多描述。
[^2]: 按理说，既然 nginx 已经知道 PHP 不可达，不去发 TCP 请求了，那么应该立即 502 才是。实验中发现，这种情况下的 502 有 3s 左右的延时，不知何故。