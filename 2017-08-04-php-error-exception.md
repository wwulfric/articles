---
title: PHP 的错误和异常处理机制
date: 2017-08-04 14:42
categories: [技术]
tags: [php, error, exception, throwable]
---

原先的 PHP 只有错误没有异常。看一些老的文档你能看到不少错误输出是直接 echo html 标签的。而现代一点的框架早已经包裹好了一切，直接抛出异常就可以有比较漂亮的错误显示页面，比如 rails 的 [better errors](https://github.com/charliesome/better_errors)。当然，PHP 的现代框架也已经做的不错了，比如 laravel。然而我司目前还是用 codeigniter 2，它的错误和异常处理还比较简陋。借着升级到 PHP7 的契机梳理了一下 PHP 的错误和异常处理的机制。

## PHP 的错误和异常

PHP5 已经实现了异常的处理，这和其他语言差别不大，无非就是 try, catch, uncaught，按下不表，先说错误。

### PHP 的错误

除了异常 PHP5 常见的就是抛出错误。你可以在官方[文档](http://php.net/manual/zh/errorfunc.constants.php)找到所有的错误的定义，这些错误可以大致分为 WARNING, ERROR(fatal error), NOTICE 等[^1]。[PHP的错误机制总结](http://www.cnblogs.com/yjf512/p/5314345.html)一文中给出了每种错误出现的场景。

> E_DEPRECATED(8192)  运行时通知,启用后将会对在未来版本中可能无法正常工作的代码给出警告。
>
> E_USER_DEPRECATED(16384)  是由用户自己在代码中使用PHP函数 trigger_error() 来产生的

> E_NOTICE(8)  运行时通知。表示脚本遇到可能会表现为错误的情况  
>
> E_USER_NOTICE(1024)  是用户自己在代码中使用PHP的trigger_error() 函数来产生的通知信息

> E_WARNING(2)  运行时警告 (非致命错误)
>
> E_USER_WARNING(512)  用户自己在代码中使用PHP的 trigger_error() 函数来产生的
>
> E_CORE_WARNING(32)  PHP初始化启动过程中由PHP引擎核心产生的警告 
>
> E_COMPILE_WARNING(128)  Zend脚本引擎产生编译时警告

> E_ERROR(1)  致命的运行时错误
>
> E_USER_ERROR(256)  用户自己在代码中使用PHP的 trigger_error()函数来产生的
>
> E_CORE_ERROR(16)  在PHP初始化启动过程中由PHP引擎核心产生的致命错误
>
> E_COMPILE_ERROR(64)  Zend脚本引擎产生的致命编译时错误

> E_PARSE(4)  编译时语法解析错误。解析错误仅仅由分析器产生

> E_STRICT(2048)  启用 PHP 对代码的修改建议，以确保代码具有最佳的互操作性和向前兼容性

> E_RECOVERABLE_ERROR(4096)  可被捕捉的致命错误。 它表示发生了一个可能非常危险的错误，但是还没有导致PHP引擎处于不稳定的状态。 如果该错误没有被用户自定义句柄捕获 (参见 set_error_handler() )，将成为一个 E_ERROR 　从而脚本会终止运行。

> E_ALL(30719) 所有错误和警告信息(手册上说不包含E_STRICT, 经过测试其实是包含E_STRICT的)。

常见的有：

```php
<?php
// E_ERROR
nonexist(); // PHP Fatal error:  Call to undefined function nonexist()
throw new Exception(''); // 未捕获异常也是 fatal error

// E_NOTICE
$a = $b; //  PHP Notice:  Undefined variable
$a = []; $a[2]; // PHP Notice:  Undefined offset: 2

// E_WARNNING
require 'nonexist.php' // warning and fatal error
```

由于历史原因，这个老旧的 ci2 框架有不少不合理的地方，比如会读取不存在的 log 文件；我们对 PHP 也有一些不规范的使用，比如：

```php
<?php
$req = [];
$user_id = $req['user_id']; // PHP error:  Undefined offset
if (null === $user_id) { /* do something */}
```

我们的代码不少地方较为依赖这种获取不存在 key 得到 null 的表现，而每次这样使用都是会有一个 E_NOTICE 错误的。虽然可以通过 array_exists 来做 if else，但毕竟比较麻烦。PHP7 之后可以通过[数据结构](http://php.net/manual/zh/book.ds.php)插件来使用 Map, Set, Vector 等明确的数据结构，从而较好的解决这个问题。

### PHP 对错误的处理

如果没有做任何配置，PHP 的错误是会直接打印出来的。古老的 PHP 应用也确实有这么做的。但现代应用显然不能这样，现代应用的错误应该遵循一下规则[^2]：

> 一定要让 PHP 报告错误；
>
> 在开发环境中要显示错误；
>
> 在生产环境中不能显示错误；
>
> 在开发和生产环境中都要记录错误。

在生产环境下，错误不能直接打印出来，应该记到 log 文件中，并返回用户一个笼统的错误信息。[set_error_handler 函数](http://php.net/manual/zh/function.set-error-handler.php)就是设置用户自定义的错误处理函数，以处理脚本中出现的错误。我们可以在这个函数中将错误信息打到 log 文件中，并统一返回错误信息。

本来这个函数是搭配 [trigger_error 函数](http://php.net/manual/zh/function.trigger-error.php)使用的。用户通过 trigger_error 产生 error，然后用 error_handler 来处理错误。只是在这种场景下往往「异常」更好用，所以这么用的并不多。

在前述的系统自带的 16 种错误中，有一部分相当重要的错误并不能被 error_handler 捕获[^3]：

> 以下级别的错误不能由用户定义的函数来处理： E_ERROR、 E_PARSE、 E_CORE_ERROR、 E_CORE_WARNING、 E_COMPILE_ERROR、E_COMPILE_WARNING，和在调用 set_error_handler() 函数所在文件中产生的大多数 E_STRICT。

这些错误将无法记录下来，同时也不方便统一处理[^4]。在 PHP7 之前的 PHP 版本一个很大的痛点就是：发生了 E_ERROR 错误，无法捕获，导致数据库的事务无法回滚造成数据不一致[^5]。

另外一个需要注意的是， error_handler 处理完毕，脚本将会继续执行发生错误的后一行。在某些情况下，你可能希望遇到某些错误可以中断脚本的执行。在[官方文档](http://php.net/manual/zh/function.set-error-handler.php)中已说明，

> 同时注意，在需要时你有责任使用 [die()](http://php.net/manual/zh/function.die.php)。 如果错误处理程序返回了，脚本将会继续执行发生错误的后一行。

也就是说，某些情况下，我们处理完 E_WARNING 之后，需要及时退出脚本（即 die() 或者 exit()）。

### PHP 异常

[异常](https://www.wikiwand.com/zh-hans/%E5%BC%82%E5%B8%B8%E5%A4%84%E7%90%86)是对程序错误的一种优秀的处理方式，较于错误，异常的优点是默认打印调用栈，便于调试，可控等，可以参考一下鸟哥的文章[我们什么时候应该使用异常](http://www.laruence.com/2012/02/02/2515.html)，清晰的点明了错误码和异常的优缺点。

对异常的处理也要遵循前述的错误处理规则[^2]。在我们的日常开发中，不可能保证可以 catch 所有的异常，而未被 catch 的异常将以 fatal error 的形式中断脚本的执行并输出错误信息。所以要借助 [set_exception_handler](http://php.net/manual/zh/function.set-exception-handler.php)，统一处理所有未被 catch 的异常。我们可以像 error_handler 那样，在 exception_handler 中处理 log，将数据库的事务回滚。

前面提到，error_handler 需要在必要的时候手动中断脚本， PHP 文档中给出的一种实践是，在 error_handler 中 throw [ErrorException](http://php.net/manual/en/class.errorexception.php)，代码示例如下：

```php
<?php
function exception_error_handler($severity, $message, $file, $line) {
    if (!(error_reporting() & $severity)) {
        // This error code is not included in error_reporting
        return;
    }
    throw new ErrorException($message, 0, $severity, $file, $line);
}
set_error_handler("exception_error_handler");

/* Trigger exception */
strpos();
```

这样凡是不想忽略的 error，都会以 Uncaught ErrorException 的形式返回并中断脚本。

### PHP 异常机制

鸟哥通过一个[例子](http://www.laruence.com/2010/08/03/1697.html)讲解了 PHP 的异常的处理机制，在这里转述一下。

```php
<?php
function onError($errCode, $errMesg, $errFile, $errLine) {
    echo "Error Occurred\n";
    throw new Exception($errMesg);
}
 
function onException($e) {
    echo '********exception: ' . $e->getMessage();
}
 
set_error_handler("onError");
 
set_exception_handler("onException");

require("nonexist.php");
```

其运行结果为

1. Error Occurred
2. PHP Fatal error

而 onException 并没有执行到，说明在 error_handler 中 throw exception 不会被 exception_handler 截获。

require 不存在的文件会抛出两个错误，

1. WARNING : 在PHP试图打开这个文件的时候抛出
2. E_COMPILE_ERROR : 从PHP打开文件的函数返回失败以后抛出

PHP 中的异常处理机制如下：

![](http://laruence-wordpress.stor.sinaapp.com/uploads/PHP-exception-cycle.png)

PHP在遇到 Fatal Error 的时候，会直接 zend_bailout，而 zend_bailout 会导致程序流程直接跳过上面代码段，也可以理解为直接 exit 了(longjmp)，这就导致了 user_exception_handler 没有机会发生作用。

### PHP 错误分类

综上所述，在 PHP 中，错误和异常可以分为以下 3 个类别：异常，可截获错误，不可截获错误。异常和可截获错误虽然机理不同，但可以当做是同一种处理方式，而不可截获错误是另一种，是一种较为棘手的错误类型。马上将会讲到，PHP7  中的 fatal error 是一种继承自 Throwable 的 Error，是可以被 try catch 住的。通过这一方式 PHP7 解决了这一难题。

## PHP7 的错误和异常

PHP 7 改变了大多数错误的报告方式。不同于传统（PHP 5）的错误报告机制，现在大多数错误被作为 **Error** 异常抛出（在 PHP7 中，只有 fatal error 和 recoverable error 抛出异常，其他 error 比如 warning 和 notice 的表现不变[^7]）。PHP7 中的 Error 和 Exception 的关系如图[^7]：

```
interface Throwable
    |- Exception implements Throwable
        |- ...
    |- Error implements Throwable
        |- TypeError extends Error
        |- ParseError extends Error
        |- ArithmeticError extends Error
            |- DivisionByZeroError extends ArithmeticError
        |- AssertionError extends Error
```

值得注意的是，Error 类表现上和 Exception 基本一致，可以像 [Exception](http://php.net/manual/zh/class.exception.php) 异常一样被第一个匹配的 try / catch 块所捕获，如果没有匹配的 [catch](http://php.net/manual/zh/language.exceptions.php#language.exceptions.catch) 块，则调用异常处理函数（事先通过 set_exception_handler() 注册[^6]）进行处理。 如果尚未注册异常处理函数，则按照传统方式处理，被报告为一个致命错误（Fatal Error）。但并非继承自 [Exception](http://php.net/manual/zh/class.exception.php) 类（要考虑到和 PHP5 的兼容性），所以不能用 `catch (Exception $e) { ... }` 来捕获，而需要使用 `catch (Error $e) { ... }`，当然，也可以使用 set_exception_handler 来捕获。

但是，用户不能自己定义类实现 Throwable，这是为了保证只有 Exception 和 Error 才可以抛出。

### PHP7 的 ERROR 处理

PHP7 中的 fatal error 会抛出 Error，且可以被正常 catch 到：

```php
<?php
$a = 1;
try {
  $a->nonexist();
} catch (Error $e) {
  // Handle error
}
```

也有些错误场景下会抛出更加详细的错误，比如：

```php
<?php
// TypeError
function test(int $i) {
  echo $i;
}
try {
  test('test');
} catch (TypeError $e) {
  // Handle error
}

// ParseError
try{
  eval('i=1;');
} catch (ParseError $e) { 
  echo $e->getMessage(), "\n";
}

// ArithmeticError
try {
    $value = 1 << -1;
} catch (ArithmeticError $e) {
    echo $e->getMessage(), "\n";
}

// DivisionByZeroError
try {
    $value = 1 % 0;
} catch (DivisionByZeroError $e) {
    echo $e->getMessage(), "\n";
}
```

### Error 和 Exception 的选择

当需要自定义处理错误的时候，应该选择继承 Error 还是 Exception 呢？

我们注意到，PHP7 中是将曾经的 fatal error 变成了 Error 抛出，而 fatal error 一般都是一些不需要在运行时处理的错误，这种错误旨在提醒程序员，这里的代码写的有问题，需要修复，而不是逻辑上要 catch 它做某些业务。

因此，绝大多数情况下，我们并不需要继承 Error，甚至 catch Error 也不常见，只在某些需要 log，回滚数据库，清理现场等场合才需要这样做。

### 对错误和异常的一种实践

根据以上所述，我们提炼了一个对错误和异常处理较好的实践。

对于业务中不应该出现错误的地方，抛出 InternalException，而不是 Error。

```php
<?php
class InternalException extends Exception { /*...*/ }

function find(Array $ids) {
  if (empty($ids)) {
    throw new InternalException('ids should not be empty');
  }
  ...
}
```

只在需要清理现场的时候 catch Error。

```php
<?php
try { /*...*/ }
catch (Throwable $t) {
  // log, transaction rollback, cleanup...
}
```

未捕获的 Error 和 Exception 通过 set_exception_handler 做后续清理和 log。

其他错误仍然通过 set_error_handler 来处理，在处理的时候使用更加明确的 FriendlyErrorType，并抛出 ErrorException 记录调用栈。

FriendlyErrorType：

```php
<?php
function FriendlyErrorType($type) 
{ 
    switch($type) 
    { 
        case E_ERROR: // 1 // 
            return 'E_ERROR'; 
        case E_WARNING: // 2 // 
            return 'E_WARNING'; 
        case E_PARSE: // 4 // 
            return 'E_PARSE'; 
        case E_NOTICE: // 8 // 
            return 'E_NOTICE'; 
        case E_CORE_ERROR: // 16 // 
            return 'E_CORE_ERROR'; 
        case E_CORE_WARNING: // 32 // 
            return 'E_CORE_WARNING'; 
        case E_COMPILE_ERROR: // 64 // 
            return 'E_COMPILE_ERROR'; 
        case E_COMPILE_WARNING: // 128 // 
            return 'E_COMPILE_WARNING'; 
        case E_USER_ERROR: // 256 // 
            return 'E_USER_ERROR'; 
        case E_USER_WARNING: // 512 // 
            return 'E_USER_WARNING'; 
        case E_USER_NOTICE: // 1024 // 
            return 'E_USER_NOTICE'; 
        case E_STRICT: // 2048 // 
            return 'E_STRICT'; 
        case E_RECOVERABLE_ERROR: // 4096 // 
            return 'E_RECOVERABLE_ERROR'; 
        case E_DEPRECATED: // 8192 // 
            return 'E_DEPRECATED'; 
        case E_USER_DEPRECATED: // 16384 // 
            return 'E_USER_DEPRECATED'; 
    } 
    return ""; 
}
```

error_handler:

```php
<?php
function exception_error_handler($severity, $message, $file, $line) {
    if (!(error_reporting() & $severity)) {
        // This error code is not included in error_reporting
        return;
    }
 	log FriendlyErrorType($severity);
    throw new ErrorException($message, 0, $severity, $file, $line);
}
set_error_handler("exception_error_handler");
```

PHP 会把所有的错误都交给错误处理程序，甚至包括错误报告设置中排除的错误。因此，我们要检查每个错误代码`$severity`，然后做适当的处理[^8]。



[^1]: [PHP中的错误级别与具体报错信息分类](http://www.jianshu.com/p/1b6004a94eb8)
[^2]: [modern php](https://book.douban.com/subject/26635862/) 第五章，P115
[^3]: E_ERROR 无法捕获，E_RECOVERABLE_ERROR 可以，后者默认输出 Catachable fatal error
[^4]: fatal error 会记录到 web 服务器的 error.log，这一点需要注意，因为这个 log 的位置往往不是 PHP 应用定义的，而是 web 服务器定义的。
[^5]: PHP 中还有一个 [register_shutdown_function](http://php.net/manual/zh/function.register-shutdown-function.php) 函数，它允许注册一个会在 PHP 中止时执行的函数，这个函数可以捕获 fatal error，毕竟是只要是脚本中断就可以捕获的。ci2 并没有使用这个方法，所以相关问题一直没有得到很好的解决，当时也没有意识到这个函数的存在，升级 PHP7 之后可以通过 catch Error 来解决，便不再需要这样处理了。
[^6]: 在 PHP7 中，传入 exception_handler 的参数从 [Exception](http://php.net/manual/zh/class.exception.php) 改为 Throwable，这意味着 exception_handler 可以截获 Error。
[^7]: [Throwable Exceptions and Errors in PHP 7](https://trowski.com/2015/06/24/throwable-exceptions-and-errors-in-php7/)
[^8]: modern php 第五章，P116