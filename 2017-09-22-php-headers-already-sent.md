---
title: PHP headers already sent 原因分析
date: 2017-09-22 18:48
categories: [技术]
tags: [ php, echo, print, ob_start, HTTP]
---

先上结论，为了避免 headers already sent 错误，你应该[^1]：

- 检查 PHP 代码，确认 <?php 前没有空格和空行
- 避免在业务代码中使用 echo 和 print 系函数，只在框架组织 HTTP body 输出的时候使用，这些函数包括

  - print, echo, printf, vprintf
  - trigger_error, ob_flush, ob_end_flush, var_dump, print_r
  - readfile, passthru, flush, imagepng, imagejpeg

## 原因分析

最近上线代码之后遇到了一个问题，在某些情况下会抛出异常：Uncaught Exception: ErrorException: Severity: 2; Message: Cannot modify header information - headers already sent by...。而且这个异常并非总是会出现，在不了解原因的情况下想要在测试环境重现比较困难，以下是分析步骤。

### 异常产生的原因

它本质上是一个 **E_WARNING**，被 error_handler 截获而抛出异常：

```php
<?php
function _error_handler($severity, $message, $filepath, $line)
{
    // ...
    if (($severity & error_reporting()) == $severity)
    {
      	// db rollback
        throw new ErrorException("Severity: $severity; Message: $message");
    }
}
```

在 index.php 中我们设置 error_reporting 要报告 E_WARNING 错误，所以会走到这里并抛出异常。也就是说，我们需要找到 E_WARNING 抛出的位置和原因。

### E_WARNING 产生的原因

```php
<p>Severity: Warning</p>
<p>Message:  Cannot modify header information - headers already sent by (output started at .../application/controllers/my_script.php:xxx)</p>
<p>Filename: libraries/Session.php</p>
```

这个错误从字面理解，就是设置 header() 的时候发现 header 中已经有内容了，那么，在异常信息中， headers already sent by () 括号里的内容就很重要了，它表明了是那一行的输出导致了这个问题。按照定位的位置，是脚本中的一个`printf`语句；继续看，是 Session 中的 setcookie() 方法发现这个 printf 语句已经输出内容了。

想要解决这个问题，可以使用 sprintf 来组装字符串，使用 fwrite 等标准输出将内容输出到控制台。

### 为什么会出现 headers already sent

在 PHP 中，不能在`header()`之前 echo 任何内容，一旦 echo，PHP 会发送已有的 header 内容，我们做一下实验。

在实验之前，你需要把`php.ini`中的 output_buffering 关闭或者设置一个很小的值。之后重启 php-fpm。

```shell
[PHP]
...
output_buffering = 3
...
```

这样设置表明输出的 buffer 不超过 3 个字符。

然后重现一下这个 bug：

```php
<?php
public function test()
{
  echo 'asd';
  header('a: b');
}
```

使用 curl 访问一下，返回的 HTTP body 是 asd 和一个 headers already sent 错误信息，`curl -I http://localhost/test`一下看看 header，发现 a: b 并没有输出到 header 中。

echo 的内容超出了缓冲区限制的长度，便会作为 HTTP body 输出给 WEB 服务器。一旦 echo，PHP 输出 header 的任务就等于结束了，那么此时调用`header()`就会抛出 headers already sent 的错误。

修改一下代码：

```php
<?php
public function test()
{
  header('b: c');
  echo 'asd';
  header('a: b');
}
```

此时输出的 HTTP body 内容是相同的，但是 curl -I 看到的 header 中多了 b: c，说明 echo 之前的`header()`正确的输出了内容。

setcookie 方法也会发送 header：`set-cookie: xxx`，所以一样会引起这个问题。

在上面的例子中，我们将 output_buffering 设置为 3，如果 echo 的内容小于 3，是不会引起问题的，因为缓冲区缓冲了 echo 的内容，会在 header 输出之后再输出缓冲内容。在实际的应用中，可以给 output_buffering 一个稍大一些的值。

但是，不能依赖 output_buffering 的大小，应该尽量避免在业务代码中使用 echo 和 print 系函数。

## 怎样使用 echo

echo 很方便，古董 PHP 开发还会使用 echo 调试大法，而且我们要输出  HTTP 内容肯定要用到 echo 或者 print，怎么可能避免使用呢？

### 业务代码中尽量避免

我们应该避免在业务中使用，而不是禁止使用。当使用 echo 的时候，因为上述原因出现 headers already sent 错误，要看 output_buffering 设置的大小和 echo 内容的长度，这给 debug 带来了很大的不确定性，测试环境很可能会漏掉这个 case。

在业务中，可能用到 echo 的原因有：1. 调试代码，查看变量；2. 命令行脚本的输出。对于 1，建议通过调试工具调试，或者使用插件 [clockwork](https://github.com/itsgoingd/clockwork)；对于 2，可以在脚本中通过标准输出来输出重要内容，并不需要使用 echo。

```php
<?php
fwrite(STDOUT, $content);
```

如果基于某种原因一定要使用，可以将一段输出用 ob_start 和 ob_end 包裹起来。被包裹的输出会进入内部缓冲区，在需要的时候再 flush 出来。

```php
<?php
// ob_start 的函数定义
bool ob_start ([ callable $output_callback = NULL [, int $chunk_size = 0 [, int $flags = PHP_OUTPUT_HANDLER_STDFLAGS ]]])
```

`$chunk_size=0`的时候，只有在关闭缓冲区的时候才会输出缓冲区的内容。[^3]


```php
<?php
public function test()
{
  ob_start(); // 打开缓冲区
  echo 'asd';
  header('a: b');
  ob_end_flush(); // 关闭缓冲区，将缓冲区的内容输出到 HTTP body
}
```

一般框架的输出都是这样设计的，echo 会包裹在 ob_start 和 ob_end 之间。

### ob_start 的问题

ob_start 不能解决 PHP 代码不规范导致的 headers already sent：

```php
           <?php
public function test()
{
  ob_start(); // 打开缓冲区
  echo 'asd';
  header('a: b');
  ob_end_flush(); // 关闭缓冲区，将缓冲区的内容输出到 HTTP body
}
// 这段代码也会报错
```

使用 ob_start 需要及时的将数据输出出去，否则可能会因为字符串拼接和二进制内容冲突：

```php
<?php
public function test()
{
  ob_start(); // 打开缓冲区
  echo 'asd';
  imagepng($resource);
  ob_end_flush(); // 关闭缓冲区，将缓冲区的内容输出到 HTTP body
}
// asd 和 imagepng() 的内容混在一起，输出的图片不可用
```

### 好的实践

综上所述，一个良好的实践是：

- output_buffering 关闭或者设置一个较小的数值[^2]
- 如非必要，不使用 echo 和 print 系函数
- 使用 echo 时，尽量用 ob_start 和 ob_end 包裹
- 使用 ob_start 和 ob_end 包裹时，对自己包裹的内容有清晰的认识，尽量不要跨函数使用 ob_start 和 ob_end

[^1]: 参见 [stackoverflow 回答](https://stackoverflow.com/questions/8028957/how-to-fix-headers-already-sent-error-in-php)，除此之外，还有 UTF-8 BOM 等其他原因
[^2]: 参见[PHP程序访问报错Warning: Cannot modify header information - headers already sent by](https://help.aliyun.com/knowledge_detail/36512.html) 和 [PHP: 运行时配置 - Manual](http://php.net/manual/zh/outcontrol.configuration.php)，开启 output_buffering 可能影响 PHP 执行效率
[^3]: 使用 ob_start 的时候不受 php.ini 中的 output_buffering 大小的影响