---
title: invariant 释义
date: 2017-11-19 15:26
categories: [技术]
tags: [java, effective java, invariant, builder]
---

Effective Java 的 [item 2](/2017/11/effetive-java-2#Item2) 的 Builder 模式可以在 Builder 的 build 中检查参数是否符合约束条件，原文和中文版译文是这样的：

> Like a constructor, a builder can impose invariants on its parameters. The build method can check these invariants. 
>
> builder 像构造器一样，可以对其参数强加约束条件。build 方法可以检验这些约束条件。


在看这一段的时候并不明白 invariant 的具体含义。

invariant 在词典里的字面意思是

> *n.* 不变式，不变量

按照这样的字面意思很难理解 'A builder can impose invariants on its parameters' 这句话：builder 强加的是什么不变式或者不变量呢？

### 参考释义

我们先看下在维基百科 [invariant](https://www.wikiwand.com/en/Invariant_(computer_science)) 词条的定义：

> In computer science, an invariant is a condition that can be relied upon to be true during execution of a program, or during some portion of it. It is a logical assertion that is held to always be true during a certain phase of execution. For example, a loop invariant is a condition that is true at the beginning and end of every execution of a loop.
>
> 在计算机科学中，invariant 就是在程序的执行过程或部分执行过程中，可以认为绝对正确的条件。它是在执行的某个阶段中总是 true 的逻辑断言。比如，循环不变性约束条件就是在循环每次执行的开始和结束都是 true 的条件。

在没有完全理解的情况下看这段介绍是有点摸不着头脑，弄明白之后再看倒是能看得懂了 -.- 而且我觉得该词条中使用的例子（循环不变式和 MU puzzle）都不太方便理解，反而是 [class invariant](https://www.wikiwand.com/en/Class_invariant) 更容易理解一点。

### 示例

网络的其他地方也可以找到解答[^1]，但我想用一个更简单的例子来说明：我们想要实现一个加法运算的函数，它接受两个整数作为参数，并输出二者相加的和。

```java
class Main {
  public static void main(String args[]){
    int a = 2147483647;
    int b = 2147483646;
    System.out.println(add(a, b));
  }
  private static int add(int a, int b) {
    return a + b;
  }
}
```

执行这段代码，我们发现 add 的结果是 -3。为什么是 -3 相信大家都比较好理解：a + b 溢出了 int 的范围。这里我们就发现了 add 方法的一个 invariant：两个正整数相加，结果必须为正整数。而目前的这个 add 方法显然没有做好这个保证。

再来看看 class invariant。在维基的词条中给出了一个 D 语言的例子：

```D
class Date {
  int day;
  int hour;

  invariant() {
    assert(1 <= day && day <= 31);
    assert(0 <= hour && hour < 24);
  }
}
```

D 语言原生支持对类的 invariant 验证。Date 的 day 必然是 1 到 31，而 hour 必然是 0 到 23，这两个条件就是 Date 类的 invariant 条件。Java 语言的 Bean Validation 其实就是一种 class invariant。

### 根据释义理解 Builder 模式中的 invariant

所谓 invariant，就是针对某个主体（对象，方法等）的一些「绝对真理」式的约束，不论该主体处于何种状态或阶段，都必须满足这些约束条件。它们不受状态和阶段的转移影响，即所谓不变性。所以，在本文中，我们将 invariant 翻译成「不变性约束条件」。

事实上 [Java Concurrency in Practice](https://book.douban.com/subject/1888733/) 这本书才是让我理解 invariant 的关键，因为在并发条件下，一个类的不变性约束条件极有可能被破坏掉，我们也很容易通过这种破坏理解什么是 class invariant：对类的不变性约束条件。

代码示例如下[^3]：

```java
// "mutable" time period class
public class Period {
    private Date start;
    private Date end;

    public Date getStart() { return start; }
    public Date setStart(Date start) { this.start = start; }
    public Date getEnd() {  return end;  }
    public Date setEnd(Date end) {  this.end = end;  }
}
```

如果有两个线程都可以操作 Period 的对象，每个线程都可以设置 start 和 end，可想而知对于 Period 的实例来说，因为缺乏必要的验证，`start <= end`这个 invariant 条件很容易被破坏。

回到 Effective Java 中的 Builder 模式。Builder 只是一个方便创建对象的工具，这里面所说的 invariant 其实就是对应类的 invariant，即要满足类所映射的现实世界的约束。

### invariant 与 immutable

将 invariant 翻译成不变性[^2]容易和 immutable 混淆，即使二者的意思其实差别很大。我们在「Java 编程思想」这本书的中文版中就发现了这个错误：

>  Integer类（以及基本的“封装器”类）用简单的形式实现了“不变性”：它们没有提供可以修改对象的方法。 若确实需要一个容纳了基本数据类型的对象，并想对基本据类型进行修改，就必须亲自创建它们。

注意这个翻译是不好的。immutable 翻译成不可变更好。虽然我在本文中将 invariant 翻译成不变性约束条件，但约束条件其实是一个隐含的意思，并非字面意思，正常情况下还是翻译成不变性比较好。因此，immutable 的翻译应能够很好地区分 invariant 才是。

invariant 指的是对对象的某些约束条件，而 immutable 指的是对象本身是不可变的。比如我们用经纬度来表示地球表面的一个点，这个类是 Point，它有两个参数：经度和纬度。所谓 invariant，就是说这个类的经度参数必须在 -180 到 180 之间，而纬度必须在 -90 到 90 之间。无论是构造器还是 setter 方法都必须验证这个 invariant 条件。所谓 immutable，指的是为了方便地实现线程安全类，我们将 Point 设计为 immutable 的，即经度和纬度属性都是 final 的，且不能提供 setter 方法。一旦实例化一个 point，它就不能再被修改，而只能通过重新 new 一个新的实例来代替旧的 point。

可见，invariant 和 immutable 是无关的，不管一个类是不是 immutable 的，它都必须受到 invariant 条件的制约（即谓不变性），否则它产生的对象就可能是无效的。



[^1]: stackoverflow 上的 [what-is-an-invariant](https://stackoverflow.com/questions/112064/what-is-an-invariant) 对 invariant 的解释也较为浅显，可以参考
[^2]: [Java Concurrency in Practice](https://book.douban.com/subject/1888733/) 这本书将 invariant 翻译成不变性
[^3]: 代码改编自 [Effective Java](https://book.douban.com/subject/2696119/) Item 39