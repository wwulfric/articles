---
title: Builder 模式如何验证对象的 fields
date: 2017-11-21 22:26
categories: [技术]
tags: [java, effective java, builder]
---

Effective Java 的 [item 2](/2017/11/effetive-java-2#Item2) 的 Builder 模式可以在 Builder 的 build 中检查参数是否符合约束条件，原文和中文版译文是这样的：

> It is critical that they be checked after copying the parameters from the builder to the object, and that they be checked on the object fields rather than the builder fields. If any invariants are violated, the `build` method should throw an `IllegalStateException`.
>
> 将参数从 builder 拷贝到对象中之后，并在对象域而不是 builder 域中对它们进行检验，这一点很重要。如果违反了任何约束条件，`build`方法就应该抛出`IllegalStateException`。

那么为什么要在对象域而不是 builder 域中验证这些 class invariant 呢？原文中有两个要点，一个是参数要从 builder 拷贝到对象中，另一个是要在对象的 fields 上做验证。

### 验证对象的 fields

我们看下这个解释[^1]：

> The constructor is where the validation occurs. Even when you're not using the builder pattern, constructors are responsible for ensuring that the object is in a valid state when it is created. And the constructor should create defensive copies  and validate the new object's fields, **not** the builder's fields, because the builder could be mutated while the fields are being copied.
>
> 所谓构造器就是字段验证所在的地方。即使你没有使用 builder 模式，也是构造器来负责检查一个对象在创建的时候是否处于正常的状态。构造器应该防御性拷贝参数，并验证新创建的对象上的字段，而不是 builder 的字段。因为在拷贝字段的时候，builder 是可变的（mutable）。

很容易理解为什么要在对象的 fields 上验证。对象上的 fields 构成了对象内部的状态，而 Builder 仅仅是一个辅助创建对象的工具，所以应该在对象的 fields 上验证这些 class invariant。

另外，Builder 可能并不是创建对象的唯一方法，如果我们在 Builder 上做验证，我们仍然无法避免在对象上做验证，这将导致验证代码的冗余[^2]，相同功能的代码冗余是各种 bug 的根源。

更不能在 setter 方法里验证。因为如果对象的状态是由多个参数构成的，在一个 setter 中是无法验证由多个参数构成的状态是否合理的。

### 参数拷贝

原文也提到了复制参数，即我们从 Builder 获取到参数之后，要防御性拷贝之后，再传到对象的构造器中。

之所以要做防御性拷贝是考虑到对象的安全，对象在创建成功之后，便不应该受到外界的影响，Effective Java 的 item 39 就是这样的一个例子；

```java
// Broken "immutable" time period class
public final class Period {
   private final Date start;
   private final Date end;

   /**
    * @param start the beginning of the period
    * @param end the end of the period; must not precede start * @throws IllegalArgumentException if start is after end
    * @throws NullPointerException if start or end is null
    */
   public Period(Date start, Date end) {
      if (start.compareTo(end) > 0)
         throw new IllegalArgumentException(start + " after " + end);
      this.start = start;
      this.end   = end;
   }

   public Date start() { return start; }
   public Date end() { return end; }
   ...  // Remainder omitted
}
```

这段代码试图将 Period 设计为一个不可变对象（immutable），但却是一个失败的例子。

```java
// Attack the internals of a Period instance
Date start = new Date();
Date end = new Date();
Period p = new Period(start, end); 
end.setYear(78); // Modifies internals of p!
```

因为 Date 是一个可变（mutable）对象，Period 的设计是失败的。解决方法就是防御性拷贝：

```java
// Repaired constructor - makes defensive copies of parameters
public Period(Date start, Date end) {
   this.start = new Date(start.getTime());
   this.end   = new Date(end.getTime());
   if (this.start.compareTo(this.end) > 0)
      throw new IllegalArgumentException(start +" after "+ end);
}
```

这样就不存在这种问题了，参数传入之后很好的和外界隔离开来（`start()`和`end()`方法也有相同的问题，这里就不展开了）。

### 在 Builder 中验证 fields 的场景

除此之外原文中还提到：

> 对多个参数强加约束条件的另一种方法是，用多个 setter 方法对某个约束条件必须持有的所有参数进行检查。如果该约束条件没有得到满足，setter 方法就会抛出`IllegalArgumentsException`。

这似乎表明，我们除了可以在`build()`方法上验证， 也可以在 setter 上验证。但这和我们上面看到的解释不是有冲突吗？

其实不是这样的。如果你有 web 开发的经历就知道，验证用户提交的表单的最好方法是，不仅要提供后端验证，也要提供前端验证。后端验证是为了保证数据一定是合理的，而前端验证是为了让错误尽早反馈给用户。因为前端的验证是可以人为跳过的（只要懂一点 js 就可以自己伪造请求），所以前端验证的目的并非保证数据合理。前端验证除了可以让错误尽快反馈给用户，也可以节省 HTTP 的请求数。

所以，如果某个状态由多个参数构成，我们可以在一个 setter 中赋值并验证这些参数的状态。比如我们有这样一个 build 和 setter 方法：

```java
// builder 的 setter 方法
public Builder setStartAndEnd(Date start, Date end) { 
  // 验证 start <= end，throw IllegalArgumentsException
  // 赋值     
  return this;
}
// builder 的 build 方法
public SomeObject build() {
  SomeObject obj = new SomeObject(this);
  // 验证状态，throw IllegalStateException 
}
```

这里我们可以尽早的验证传入的 start 和 end 是否满足要求。同时要注意，对象上的验证仍不可省略。

其实也有在 Builder 的 setter 中验证而不必在对象上验证参数的场景，stackexchange 的这个[回答](https://softwareengineering.stackexchange.com/a/241319)也提到了这种可能[^2]。在这个场景中，setter 方法的参数类型和实际创建对象的参数类型不一致，那么，setter 方法必须独自验证自己的参数的状态，而对象则在创建时验证对象入参的状态。

```java
// builder 的 setter 方法
public Builder setStartAndEnd(Date start, Date end) { 
  // 验证 start <= end，throw IllegalArgumentsException
  // 赋值
  this.period = new Period(start, end);
  return this;
}
// builder 的 build 方法
public SomeObject build() {
  SomeObject obj = new SomeObject(this);
  // 验证状态，throw IllegalStateException 
}
```

如代码所示，builder 的 setter 方法入参是两个 Date，而构造函数的入参是 Period(实际是 Builder) 。

### 错误处理

还有一点不知道大家有没有注意到，如果是 setter 中参数错误，应该抛出 IllegalArgumentsException 异常，而如果是在对象 fields 上验证错误，应该抛出 IllegalStateException 异常。这是符合异常的规范要求的。

| Exception                       | Occasion for Use(使用场景)                   |
| ------------------------------- | ---------------------------------------- |
| IllegalArgumentsException       | Non-null parameter value is inappropriate |
| IllegalStateException           | Object state is inappropriate for method invocation |
| NullPointerException            | Parameter value is null where prohibited |
| IndexOutOfBoundsException       | Index parameter value is out of range    |
| ConcurrentModificationException | Concurrent modification of an object has been detected where it is prohibited |
| UnsupportedOperationException   | Object does not support method           |

如上表所示。通常 setter 的参数验证可以类比于 web 的前端验证，验证的是参数是否合法，而对象上的验证则是验证构成对象的状态是否合法。

[^1]: stackoverflow 上的这个问题 [builder-pattern-validation-effective-java](https://stackoverflow.com/questions/38173274/builder-pattern-validation-effective-java) 的解答了这个问题，在 object 上验证 fields 更规范，因为实际上要验证的就是 object 的 fields 而不是 builder 的，builder 只是一个方便创建对象的工具
[^2]: stackexchange 上的 [builder-pattern-when-to-fail](https://softwareengineering.stackexchange.com/questions/241309/builder-pattern-when-to-fail) 的解答深入讨论了这个问题，而且存在不少争论，不过大体上还是认同的居多