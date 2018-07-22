压一下 php 机器

https://www.jianshu.com/p/3e3fb0459ca3 proxy timeout 需要两次读有间隔时间，用 ob flush 和 flush 一起。

https://shaohualee.com/article/760 php-fpm 从 tcp 到 unix socket。nginx 到 php-fpm 的 tcp 连接是频繁的创建和关闭 tcp 连接的。

php-fpm nginx 长连接 http://www.xtgxiso.com/nginx%E7%9A%84upstream%E8%BF%9E%E6%8E%A5%E6%B1%A0/

http://mark311.github.io/2014/11/01/nginx-fastcgi-keepalive.html

https://blog.csdn.net/wangkai_123456/article/details/71715852 连接池设置

https://blog.csdn.net/qq624202120/article/details/60957634 图，同时说明 unix socket 不稳定，不如 tcp 这样面向连接的协议好

测试单连接，多次循环

测试多连接压测

验证长连接 用 tcpdump