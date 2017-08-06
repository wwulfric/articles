php 的数据结构扩展



```php
$req = [];
$req['user_id']; // PHP error:  Undefined offset
$req = \Ds\Vector([]);
$req.get('user_id');// OutOfBoundsException
$req.get('user_id', 0); // 0 是默认值
```

即可以方便的指定默认值，也可以选择抛出异常。

不用array，不会产生 E_NOTICE