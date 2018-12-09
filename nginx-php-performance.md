压一下 php 机器

https://www.jianshu.com/p/3e3fb0459ca3 proxy timeout 需要两次读有间隔时间，用 ob flush 和 flush 一起。

https://shaohualee.com/article/760 php-fpm 从 tcp 到 unix socket。nginx 到 php-fpm 的 tcp 连接是频繁的创建和关闭 tcp 连接的。

php-fpm nginx 长连接 http://www.xtgxiso.com/nginx%E7%9A%84upstream%E8%BF%9E%E6%8E%A5%E6%B1%A0/

http://mark311.github.io/2014/11/01/nginx-fastcgi-keepalive.html

https://blog.csdn.net/wangkai_123456/article/details/71715852 连接池设置

https://blog.csdn.net/qq624202120/article/details/60957634 图，同时说明 unix socket 不稳定，不如 tcp 这样面向连接的协议好



https://my.oschina.net/eechen/blog/541139  fpm 进程数，backlog 数

http://blog.51cto.com/benpaozhe/1846784 fpm 连接模型

https://www.jianshu.com/p/542935a3bfa8  FPM一个worker可同时只能处理一个个连接 这是PHP FPM 和 Nginx和Tomcat 的重大区别

https://www.fanhaobai.com/2017/10/internal-php-fpm.html fpm 分析

https://www.zhihu.com/question/64414628  php fpm 进程数和并发数是什么关系

https://my.oschina.net/eechen/blog/369470 PHP 和 nodejs 压测

http://www.fzb.me/2017-7-18-php7-internal-some-questions.html   并发受限于进程数，高并发请求时，fpm-worker不够用，nginx只能响应502

http://rango.swoole.com/archives/508  PHP并发IO编程之路



https://www.if-not-true-then-false.com/2011/nginx-and-php-fpm-configuration-and-optimizing-tips-and-tricks/ nginx + php-fpm 优化点

https://segmentfault.com/a/1190000007223278 同上

测试单连接，多次循环

测试多连接压测

验证长连接 用 tcpdump