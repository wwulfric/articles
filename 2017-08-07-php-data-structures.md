---
title: PHP 的数据结构扩展
date: 2017-08-07 17:02
categories: [技术]
tags: [php, 数据结构, ds]
math: true
---

在 PHP 中表示集合的数据类型就一种：Array。相信每个初学 PHP 的都会对它感到疑惑。这个东西看起来应该和其他语言中的 Array 或者 List 一样，但在 PHP 中，它是一切，即是 List，也是 Map：

```php
<?php
$a = array(1, 2, 3);
$b = array('key1' => 1, 'key2' => 2);
```

这听起来似乎很好，反正大家都使用同一种数据结构，偶尔情况下才会有些性能问题，况且升级 PHP7 之后 Array 的性能也提高了，实在不济还可以加内存。但如果我们可以通过引入更便利的数据结构优化性能，同时写代码反而更方便了，那何乐而不为呢?

## Array 的缺点

有些时候我们需要保存一个集合（Set），但是 Array 并不能保证元素的唯一性，`array_unique` 有不可避免的性能损耗。一种折衷方案是，将元素当做 key，同时 value 为 true 来曲线实现 Unique Array 的功能：

```php
<?php
$users = User::find($ids);
$res = [];
foreach ($users as $user) {
  $res[$user->id] = true;
}
```

PHP 的 Array 访问不存在的 key 可以得到 null，不会产生 fatal error，但会有一个 E_NOTICE。这个 E_NOTICE 会被 set_error_handler 注册的函数截获[^4]。显然，这种代码上的不干净和性能上的无谓开销完全是可以避免的。

```php
<?php
$req = [];
$req['user_id']; // PHP Notice:  Undefined offset
```

可以用 array_key_exists 和 if else 来让代码干净一些，但这样就显得啰嗦了。

array 的一些函数式方法很难用，比如 array_map, array_walk 等，写起来也很丑陋。当然这一点原生 PHP 没什么好方法，毕竟 PHP 的面向对象的基因不是很强。

```php
<?php
array_map(function($user){
  return $user->is_deleted();
}, $users);
// 就是这么难看
```

```ruby
users.map { |user| user.is_deleted? }
# ruby 的就好看多了
```

在某些情况下，使用 Array 性能很差[^1]，比如下面这段代码：

```php
<?php
$a=[1,2,3,4,5,6,7];
echo $a[5];
// 6
array_unshift($a, 0);
// $a: [0,1,2,3,4,5,6,7];
echo $a[5];
// 5
```

看起来似乎没什么，但需要注意的是，Array 本质上是一个 Map，unshift 一个元素进来，将会改变每个元素的 key，这是一个 $O(n)$ 操作。另外，PHP 的 Array 将其 value(包括 key 和 它的 hash) 保存在一个 bucket 中，所以我们需要查看每一个 bucket 并更新 hash。PHP 内部其实是通过创建新的 array 来做 array_unshift 操作的，其性能问题可想可知[^2]。

其他缺点不一而足。

## PHP 数据结构插件

Array 饱受诟病，就会出现替代方案。PHP5 有[spl](http://php.net/manual/zh/spl.datastructures.php)，但是有些场景性能很差，且设计的很不好[^1]。laravel 的 [Collection](http://d.laravel-china.org/docs/5.4/collections) 提供了更好用的 Map，但毕竟只是一种单一的数据结构，而且对 orm 操作设计了不少特有的接口，其用途受到限制。

PHP7 新增的 [Data Structures](http://php.net/manual/zh/book.ds.php) 插件（简称 ds）是 PHP 下一个优秀的补充，它充分考虑了便利、安全和整洁的需求。其结构如下图所示：

![php data structure 插件结构](http://static.wulfric.me/php/phpds.png)

它提供了 3 个接口类：Collection, Sequence, Hashable 和 7 个实现类（final class）：Vector, Deque, Map, Set, Stack, Queue, PriorityQueue。

### 接口

Collection 是基础接口，定义了一个数据集合（这里的集合指的是 Collection，不是 Set） 的基本操作，比如 foreach, json_encode, var_dump 等。

```php
<?php
$sequence = new \Ds\Vector([1, 2, 3]);
json_encode($sequence);
```

Sequence 是类数组数据结构的基础接口，定义了很多重要且方便的方法，比如 contains, map, filter, reduce, find, first, last 等。从图中可知，Vector, Deque, Stack, Queue 都直接或者间接的实现了这个接口。

```php
<?php
$sequence = new \Ds\Vector([1, 2, 3]);

print_r($sequence->map(function($value) { return $value * 2; }));
print_r($sequence);
```

Hashable 在图中看起来比较孤立，但对于 Map 和 Set 很重要。一个 Object 如果实现了 Hashable，就可以作为 Map 的 key，可以作为 Set 的元素。这样 Map 和  Set 就能像 Java 一样方便的使用了。

### 实现类

Vector 应该是最为常用的数据结构之一了，可以把它当成 Ruby 的 Array 或者 Python 的 List。其元素的值的 index 就是它在 buffer 中的 index，所以效率很高。只要有使用数组的需求且不需要 insert, remove, shift 和 unshift 的都可以用它。

Deque([dek]) 是双端队列，在 Vector 的基础上增加了一个头指针，因此 shift 和 unshift 也是 $O(1)$ 复杂度了。但带来的性能损耗并不多，因此也有讨论是不是只需要一个 Deque 就够了，不需要 Vector（[讨论](https://github.com/php-ds/extension/issues/45)）[^3]。

Stack 栈，嗯没什么好说的，它继承自 Collection，但内部使用 Vector 实现。这样做的好处是实现方便，且同时可以屏蔽不需要的和不应该出现的方法。

Queue 队列，内部使用 Deque 实现。

PriorityQueue，最大堆实现。

Map。以前使用 Array 来实现 map 的地方，改用 Map 更好。二者性能几乎一致，但 Map 对内存的管理更好。而且，Map 的语法要更加友好。

```php
<?php
$req = [];
$req['user_id']; // PHP Notice:  Undefined offset

$req = new \Ds\Map(["a" => 1, "b" => 2, "c" => 3]);
$req->get('user_id');// OutOfBoundsException
$req->get('user_id', 0); // 0 是默认值
// 即可以方便的指定默认值，也可以选择抛出异常。不用 array，不会产生 E_NOTICE

$req->keys();

$req->map(function($key, $value) { return $value * 2; });
```

不仅如此，只要 object 继承了 Hashable，Map 还允许使用 object 作为 key。

```php
<?php
class Photo implements \Ds\Hashable {
    
    public function __construct($id) {
        $this->id = $id;
    }
    
    public function hash() {
        return $this->id;
    }

    public function equals($obj): bool {
        return $this->id === $obj->id;
    }
}

$p1 = new Photo(1);
$p2 = new Photo(2);

$map = new Ds\Map();
$map->put($p1, 1);
$map->put($p2, 2);
```

Set 集合是一种元素唯一的数据结构。和 array_unique 相比性能有很大提升，而且用法也更加优雅[^1]。

```php
<?php
$set = new Ds\Set();
$set->add($p1);
$set->add($p2);
```

[^1]: [php ds 插件性能测试](https://medium.com/@rtheunissen/efficient-data-structures-for-php-7-9dda7af674cd)
[^2]: 当然，这一点可能稍嫌牵强，毕竟即使是数据量很大的情况下，array_unshift 的耗时也没有那么大
[^3]: github 上还在讨论可以增加一个不可变类型 Tuple，以及取消 Vector 直接使用 Deque，讨论[地址](https://github.com/php-ds/extension/issues/52)和 [2.0API](https://github.com/php-ds/extension/issues/90) 计划
[^4]: 关于 PHP 的错误处理可以参考笔者的另一篇博文 [PHP 的错误和异常处理机制](/2017/08/php-error-exception/)