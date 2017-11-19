Builder 模式和 invariant 释义

Effective Java 的 [item 2](/2017/11/effetive-java-2#Item2) 的 Builder 模式可以在 Builder 的 build 中检查参数是否符合约束条件，原文和中文版译文是这样的：

- Like a constructor, a builder can impose invariants on its parameters. The build method can check these invariants. builder 像构造器一样，可以对其参数强加约束条件。build 方法可以检验这些约束条件。
- It is critical that they be checked after copying the parameters from the builder to the object, and that they be checked on the object fields rather than the builder fields. 将参数从 builder 拷贝到对象中之后，并在对象域而不是 builder 域中对它们进行检验，这一点很重要。

当时看这段的时候，有两点没明白：1 是什么叫 invariant，2 是为什么一定要在 object fields 上做校验而不是 builder fields。

## invariant 释义

invariant 在词典里的字面意思是

> *n.* 不变式，不变量

按照这样的字面意思很难理解 'A builder can impose invariants on its parameters' 这句话：builder 强加的是什么不变式或者不变量呢？

我们先看下在维基百科 [invariant](https://www.wikiwand.com/en/Invariant_(computer_science)) 词条的定义：

> In computer science, an invariant is a condition that can be relied upon to be true during execution of a program, or during some portion of it. It is a logical assertion that is held to always be true during a certain phase of execution. For example, a loop invariant is a condition that is true at the beginning and end of every execution of a loop.
>
> 在计算机科学中，invariant 就是在程序的执行过程或部分执行过程中，可以认为绝对正确的条件。它是在执行的某个阶段中总是 true 的逻辑断言。比如，循环不变性约束条件就是在循环每次执行的开始和结束都是 true 的条件。

在没有完全理解的情况下看这段介绍是有点摸不着头脑，弄明白之后再看倒是能看得懂了 -_- 而且我觉得该词条使用的例子（循环不变式和 MU puzzle）都不太方便理解，反而是 class invariant 更容易理解一点。

网络的其他地方也可以找到解答[^3]，但我想用一个更简单的例子来说明：我们想要实现一个加法运算的函数，它接受两个正整数作为参数，并输出二者相加的和。

```java

```



### invariant 与 immutable

将 invariant 翻译成不变性容易和 immutable 混淆，虽然二者的意思其实差别很大。我们在「Java 编程思想」这本书中就发现了这个错误：

>  Integer类（以及基本的“封装器”类）用简单的形式实现了“不变性”：它们没有提供可以修改对象的方法。 若确实需要一个容纳了基本数据类型的对象，并想对基本据类型进行修改，就必须亲自创建它们。

注意这个翻译是不好的。immutable 翻译成不可变更好。虽然我在本文中将 invariant 翻译成不变性约束条件，但约束条件其实是一个隐含的意思，并非字面意思，正常情况下还是翻译成不变性比较好。因此，immutable 的翻译应能够很好地区分 invariant 才是。

invariant 指的是对对象的某些约束条件，而 immutable 指的是对象本身是不可变的。比如我们用经纬度来表示地球表面的一个点，这个类是 Point，它有两个参数：经度和纬度。所谓 invariant，就是说这个类的经度参数必须在 -180 到 180 之间，而纬度必须在 -90 到 90 之间。无论是构造器还是 setter 方法都必须验证这个 invariant 条件。所谓 immutable，指的是为了方便地实现线程安全类，我们将 Point 设计为 immutable 的，即经度和纬度属性都是 final 的，且不能提供 setter 方法。

可见，invariant 和 immutable 是无关的，不管一个类是不是 immutable 的，它都必须受到 invariant 条件的制约（即谓不变性），否则它产生的对象就可能是无效的。

## 参数校验域

The constructor is where the validation occurs. Even when you're not using the builder pattern, constructors are responsible for ensuring that the object is in a valid state when it is created. And the constructor should create defensive copies  and validate the new object's fields[^1].

build 可以验证， setter 也可以验证

build 验证不变性，setter 统一验证不变性，比如 setStartAndEnd(int start, int end)

IllegalArgumentsException 和 IllegalStateException 的区别



[^1]: stackoverflow 上的这个问题 [builder-pattern-validation-effective-java](https://stackoverflow.com/questions/38173274/builder-pattern-validation-effective-java) 的解答了这个问题，在 object 上验证 fields 更规范，因为实际上要验证的就是 object 的 fields 而不是 builder 的，builder 只是一个方便创建对象的工具。
[^2]: stackexchange 上的这个问题 [builder-pattern-when-to-fail](https://softwareengineering.stackexchange.com/questions/241309/builder-pattern-when-to-fail) 的解答从安全角度说明了这个问题。怎么说明的
[^3]: stackoverflow 上的 [what-is-an-invariant](https://stackoverflow.com/questions/112064/what-is-an-invariant) 对 invariant 的解释也较为浅显，可以参考。
[^4]: 事实上 [Java Concurrency in Practice](https://book.douban.com/subject/1888733/) 这本书才是让我理解 invariant 的关键，因为在并发条件下，一个类的不变性约束条件极有可能被破坏掉，我们也很容易通过这种破坏理解什么是 class invariant：对类的不变性约束条件。------这本书翻译成不变性