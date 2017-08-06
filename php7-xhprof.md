php7 安装 xhprof





## 使用兼容的 xhprof

xhprof已经很久没有更新了，使用他人的repo

```shell
cd ~
git clone https://github.com/longxinH/xhprof
cd xhprof/extension/
```



## 按照常规方式安装，注意之前用PHP5的命令，都要改成PHP7

### 查找 php7 下的 phpize

```shell
which php71 # 查看 php71 的位置
ll /usr/bin/php71 # 查看软链接的位置
ls /opt/remi/php71/root/usr/bin/ # 查看所有命令
```

### 安装

```shell
/opt/remi/php71/root/usr/bin/phpize
./configure --with-php-config=/opt/remi/php71/root/usr/bin/php-config  --enable-xhprof
sudo make
sudo make install
```

### 重启

```shell
sudo service php71-php-fpm restart
php71 -m | grep xhprof
```



此时，安装成功。我们配置一下 xhprof 插件：

```shell
[xhprof]
extension=xhprof.so;
xhprof.output_dir=/var/tmp/xhprof
```



其中 xhprof.output_dir 是 xhprof 的输出目录，每次执行 xhprof 的 save_run 方法时都会生成一个 run_id.project_name.xhprof 文件。这个目录在哪里并不重要。

## nginx 配置访问

当生成 .xhprof 文件之后，我们就可以利用 xhprof_lib 来展示结果了。

创建文件夹 /var/www/html/xhprof，然后配置 nginx 如下：

```nginx
server {
    listen 80;
    root /var/www/html/xhprof/;
    server_name xhprof.fangcloud.net;
    location = / {
        index index.php;
    }
    location ~ \.php {
        fastcgi_pass 127.0.0.1:9090;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```





在 /var/www/html/xhprof/xhprof_html 下创建 index.php

```php
<?php
    echo phpinfo();
?>
```



访问 xhprof.fangcloud.net/index.php 查看是否正确显示 phpinfo。

当配置成功之后，将 xhprof 中的xhprof_lib和xhprof_html两个文件夹复制到 /var/www/html/xhprof/ 下，然后访问 [http://xhprof.fangcloud.net/xhprof_html/index.php](http://xhprof.fangcloud.net/xhprof_html/index.php) 即可。

安装 yum install graphviz 查看图形界面

修改 nginx 配置，root 改成`root /var/www/html/xhprof/xhprof_html;`，因为 xhprof_html 下已经有 index.php 了，所以可以直接访问  http://xhprof.fangcloud.net/

## 使用方法

和之前一样，将要检查性能的代码包裹起来就可以了。

```php
xhprof_enable(XHPROF_FLAGS_CPU + XHPROF_FLAGS_MEMORY);
 
// 要检查性能的代码
 
$xhprof_data = xhprof_disable();
include_once  '/var/www/html/xhprof/xhprof_lib/utils/xhprof_lib.php';
include_once  '/var/www/html/xhprof/xhprof_lib/utils/xhprof_runs.php';
$xhprof_runs = new \XHProfRuns_Default();
$run_id = $xhprof_runs->save_run($xhprof_data, 'cloudoffice');
```



当然这里也可以将前后的代码单独开一个php文件：header.php 和 footer.php，然后用这两个文件包裹代码。只是现在没有 htaccess 文件，不是很方便了。当然你也可以选择将

```
php_value[auto_prepend_file] = /var/www/html/xhprof/header.php
php_value[auto_append_file] = /var/www/html/xhprof/footer.php
```

```
加到 php-fpm.d/www.conf 中，不过依然不方便。
```

```

```

## 问题

### pango 错误

如果遇到错误failed to execute cmd: " dot -Tpng". stderr: ` ([process:24220](http://process:24220/)): Pango-WARNING **: Invalid UTF-8 string passed to pango_layout_set_text() '。暂时不清楚怎么解决，可以选择避开它。

将 xhprof_lib/utils/callgraph_utils.php 的 121，122 行注释掉。

 

## PS

```
下面是一些参数说明
Inclusive Time                 包括子函数所有执行时间。
Exclusive Time/Self Time       函数执行本身花费的时间，不包括子树执行时间。
Wall Time                      花去了的时间或挂钟时间。
CPU Time                       用户耗的时间+内核耗的时间
Inclusive CPU                  包括子函数一起所占用的CPU
Exclusive CPU                  函数自身所占用的CPU
```

 

更好看一点:

[https://lamosty.com/2015/03/19/profiling-wordpress-with-xhprof-on-mac-os-x-10-10/](https://lamosty.com/2015/03/19/profiling-wordpress-with-xhprof-on-mac-os-x-10-10/)

[http://blog.oneapm.com/apm-tech/235.html](http://blog.oneapm.com/apm-tech/235.html)

[https://tideways.io/profiler/xhprof-for-php7-php5.6](https://tideways.io/profiler/xhprof-for-php7-php5.6) (手动搭免费）

## 参考：

[https://github.com/phacility/xhprof/issues/82](https://github.com/phacility/xhprof/issues/82)

[http://www.jianshu.com/p/c420ebe6ce39](http://www.jianshu.com/p/c420ebe6ce39)

[https://wizardforcel.gitbooks.io/nginx-doc/content/Text/6.5_nginx_php_fpm.html](https://wizardforcel.gitbooks.io/nginx-doc/content/Text/6.5_nginx_php_fpm.html)

[https://www.nginx.com/resources/wiki/start/topics/examples/phpfcgi/](https://www.nginx.com/resources/wiki/start/topics/examples/phpfcgi/)