---
title: Effective-Java-创建和销毁对象
date: 2017-11-13 20:08
categories: [技术]
tags: [java, effective java]
---

Effective Java[EJ] 的第二章讲的是创建和销毁对象：在什么情况下需要创建对象，怎样创建；在什么情况下需要避免创建对象，怎样避免；如何按照既定顺序销毁对象，如何在销毁对象前正确清理现场。

## 使用静态工厂方法而不是构造器 {#Item1}

创建对象的最基础最常用的方法是通过构造函数 new 出来。但在实际的复杂业务环境中，往往显得太简单，不能很好的满足各种业务需求，而且很有可能使代码变得混乱。

你可以考虑为你的类提供一个静态工厂方法。静态工厂方法是一个返回类的示例的静态方法，如：

```java
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```

通过静态工厂方法获取实例拥有以下优势：

- 静态工厂方法可以根据方法名表明自己的作用，比如 valueOf 表示获取值，newInstance 表示创建新实例，getInstance 一般意味着拿到一个重用的实例。如果一个类需要提供不同的实例，静态工厂方法可以很清晰的表明自己，而构造器由于方法名必须和类名一致，容易让人感到混乱；
- 静态工厂方法在调用时不一定会创建对象，比如 Boolean.valueOf 就不会创建对象；
- 构造器会返回一个确定的类型（当前类），而静态工厂方法可以返回 subtype（子类，接口类），这在 java.util.Collections 中用的非常多；

```java
public static Map getInstance();
// 这个接口可以返回所有实现了 Map 的接口的类的实例

public static AbstractMap getInstance();
// 这个接口可以返回所有继承了 AbstractMap 的类的实例
```

- 让开发者更关注返回对象的接口而不是具体的实现类；
- 方便维护。比如一个接口返回 EnumSet，在实现上可以返回各种 EnumSet 的子类型，升级之后也可以更改返回的类型。

一个典型的例子就是服务提供框架（TODO）。

使用静态工厂方法的劣势：

- 只有静态工厂方法的类，无法创建子类来继承；
- 不能让人一眼看出来是重要的创建对象的方法（和构造函数相比），只能在注释中写明 。

常用的静态工厂方法的名字有：valueOf, of,newInstance, getInstance, getType, newType 等。

当你编写类的时候，请先考虑使用静态工厂方法，如果确定不需要，再使用构造器。

## 构造函数参数过多的时候可以考虑 Builder {#Item2}

静态工厂方法和构造器有一个共同的局限：不能很好的扩展到大量的可选参数。书中给出了这样一个例子：

```java
// Telescoping constructor pattern - does not scale well!
public class NutritionFacts {
  // ... field 定义
  public NutritionFacts(int servingSize, int servings);
  public NutritionFacts(int servingSize, int servings, int calories);
  public NutritionFacts(int servingSize, int servings, int calories, int fat);
  public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium);
  public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium, int carbohydrate);
}
```

这样的构造函数想必在没有充分优化的代码中很常见。客户端想要的实例可能有各种情况的默认值，类就要为各种情况都提供构造器，代码显得非常臃肿；当获取实例的时候，冗长的参数列表也很容易导致参数填写出错。静态工厂方法也是如此。

JavaBean 可以避免这种问题。JavaBean 在现代 Java 代码中应该说使用的非常频繁了（序列化，对象持久化等）。客户端先 new 一个空对象，然后调用 setter 方法设置属性。JavaBean 代码没那么臃肿了，同时也不容易填错参数，但是一样有缺点：

- 代码长，每个参数都需要提供 setter 方法；setter 方法通常返回值是 void，这意味着客户端代码只能一行一行 set，不能链式调用；
- JavaBean 便无法实现不可变对象，因为随时可以执行 setter 方法，对象的状态不可控。JavaBean 极有可能出现不一致的情况，对客户端来说，它并不知道这个 JavaBean 是否已经完成构建（你并不知道某个 setter 是尚未执行还是不需要执行），这导致该对象是线程不安全的。

Builder 模式可以很好的兼顾安全和代码可读性。Builder 是目标类的静态成员类，在 Builder 类中处理实际参数的设置。客户端通过 Builder 类设置参数，然后由无参的 build 方法获取所需的不可变对象。

```java
// Builder 模式
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;

    public static class Builder {
        // 必需参数
        private final int servingSize;
        private final int servings;

        // 可选参数 - 初始化为默认值
        private int calories     = 0;
        private int fat          = 0;
        private int sodium       = 0;
        private int carbohydrate = 0;

        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings    = servings;
        }

        public Builder calories(int val) 
            { calories = val;        return this; }
        public Builder fat(int val)
            { fat = val;             return this; }
        public Builder sodium(int val)
            { sodium = val;          return this; }
        public Builder carbohydrate(int val)
            { carbohydrate = val;    return this; }

        public NutritionFacts build() {
            return new NutritionFacts(this);
        }
    }

    private NutritionFacts(Builder builder) {
        servingSize  = builder.servingSize;
        servings     = builder.servings;
        calories     = builder.calories;
        fat          = builder.fat;
        sodium       = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }
}
```

可见，NutritionFacts 是不可变的；Builder 的 setter 方法返回 this，这样可以链式调用 setter，如下所示：

```java
NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8).calories(100).sodium(35).carbohydrate(27).build()
```

动态语言如 Python 可以通过参数默认值实现类似效果，比如：

```python
def f(a=1, b=2, c=3):
  print(a,b,c)

f(c=1,a=3)
```

Builder 模式代码易编写，易阅读，同时可以实现不可变对象，因此是线程安全的。除此之外：

- 参数可检查，可以在 Builder 的 build 中检查参数是否符合 invariant 约束条件（见 [invariant 释义](/2017/11/what-is-invariant)）。约束条件的检查应该在 object 的 fields 上而不是 Builder 的 fields 上， 否则约束条件还是有可能被破坏（[为什么要在对象的 fields 上验证](/2017/11/builder-validate-fields)）；
- 可以有多个 varargs（每个 setter 都可以），非常灵活；
- 创建对象时可以自动填入某些字段，例如每次创建对象时自动增加序列号；
- 如果 builder 是外部类，那么设置了参数的 builder 是一个很好的静态抽象工厂

当然 Builder 模式也有自身的缺点：

- 创建 Builder 导致的潜在的性能问题，一般来说几乎可以忽略不计，但在某些性能敏感的场合就必须考虑到这种损耗；
- 写起来稍微有点麻烦，因此只适用于参数很多，定制化很强的场景（要考虑到未来的情况，一个类很有可能有简单变得复杂）。

随着代码的积累可能既有构造器也有 Builder，这使得代码显得混乱不易控制，因此推荐优先使用 Builder。

## 使用私有构造器或枚举类型强化单例属性 {#Item3}

单例也是一种常用的设计模式，Singleton 通常被用来代表那些本质上唯一的系统组件，比如窗口管理器或者文件系统。但 Singleton 使得测试更困难：无法使用 mock 的替代实现，除非实现了一个可以看做该类型的接口。

常用的两个实现单例方法：

- 实例成员是一个 final 的变量

```java
// Singleton with public final field
public class Elvis {
    public static final
Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
public void leaveTheBuilding() { ... }
}
```

- 通过一个静态工厂方法返回

```java
// Singleton with static factory
public class Elvis {
    private static final
Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public static Elvis getInstance() { return INSTANCE }
public void leaveTheBuilding() { ... }
}
```

需要注意，享有特权的客户端可以借助 AccessibleObject.setAccessible 方法，通过反射机制（Item 53）调用私有构造器。如果需要抵御这种攻击，可以修改构造器，让它在被要求创建第二个实例的时候抛出异常。

得益于现代 JVM 的优化，二者性能上没啥区别；前者简单，但后者更加灵活，可以支持全局单例，线程内单例等；而且在泛型方面也更有优势。

前两种实现有这么一个问题：容易在序列化的时候踩坑。因为每次反序列化都会创建一个新的对象，所以需要添加一个 readResolve 方法：

```java
// readResolve method to preserve sigleton property
private Object readResolve() {
    // Return the one true Elvis and let the garbage
collector
    // take care of the Elvis impersonator.
    return
INSTANCE;
}
```

现在更推荐使用枚举来实现单例，它和 public field 方法是一样的，但是自带序列化支持，可以防范复杂的序列化和反射攻击。单元素的枚举类型已经成为实现 Singleton 的最佳方法。

```java
// Enum singleton - the prefered approach
public enum Elvis {
    INSTANCE;

    public void leaveTheBuilding() { ... }
}
```

## 使用私有构造器强制禁止实例化 {#Item4}

有些类就是需要设计成不能实例化的，比如 util 工具类，Math 类，实现同一接口的对象的工厂方法等等。对于这些类，可以提供一个私有的构造器，这样就不能实例化了。

```java
// Noninstantiable utility class
public class UtilityClass {
    // Suppress default constructor for noninstantiability
    private UtilityClass() {
        throw new AssertionError();
    }
    ... // Remainder omitted
}
```

缺点是这种类也无法继承了。

## 避免创建不必要的对象 {#Item5}

如果对象可重用，尽量重用而不是每次都新建，这样能够节省资源。如果对象是不可变的，它就始终可以被重用。


对于同时提供了静态工厂方法（[Item 1](#Item1)）和构造器的不可变类，通常可以使用静态工厂方法而不是构造器，以避免创建不必要的对象。

如果一个属性初始化之后就不再变化，并且会被多次使用，它应该被当做常量（static final field），尽量不要当做变量。

有一种容易忽略的不必要创建对象的场合：基本类型的自动装箱。如下所示：

```java
// Hideously slow program! Can you spot the object creation?
public static void main(String[] args) {
    Long sum = 0;
    for (long i = 0; i <
            Integer.MAX_VALUE; i++) {
        sum += i;
    }
    System.out.println(sum);
}
```

上面这段代码可以正常运行，但是很慢，因为 sum 是 Long 类型，而 i 是 long 类型，执行`sum += i`的时候会自动给 i 装箱 Long，然后执行计算。可想而知自动装箱执行了 Integer.MAX_VALUE 次，也创建了 Integer.MAX_VALUE 这么多中间对象。

解决方法：将`Long sum = 0L`改成`long sum = 0`即可，这样就不存在自动装箱了。但不能将`long i = 0`改成`Long i = 0L`，因为 Long 是不可变对象，所以在进行加法的时候，事实上是每次都要创建 Long 对象，速度依然会很慢，但 long 没有对象创建，只是基本类型，速度会好很多。

这个例子表明，要优先使用基本类型而不是装箱基本类型，要当心无意识的自动装箱。

但是实际使用中恐怕还是装箱类型多一些，毕竟泛型只支持装箱类型不支持基本类型。书中也表示，小对象的创建和销毁是很轻量的操作，通过创建对象使得程序简洁、清晰、功能强大，这是一件好事。

在提倡使用保护性拷贝的时候，因为重用对象要付出的代价要远远大于因创建重复对象而付出的代价。必要时如果没能实施保护性拷贝，将会导致潜在的错误和安全漏洞；而不必要的创建对象则只会影响程序的风格和性能。

## 消除过期的对象引用 {#Item6}

Java 不能使用指针，过期的对象由系统自动回收（GC）。但是即使有 GC，也不能完全不关心内存管理，本节讲的就是这种情况。

书中的 Stack 示例表明，如果没有及时清除过期引用，就可能导致出现内存泄露。一个过期引用没有清除的话，该对象所引用的其他对象也不会被清除了，随着这种过期引用的积累，可能会使程序发生 OOM 崩溃。

解决方法就是 null 掉过期的引用。

但是不应该对此过分小心，jvm的 GC 效率是很高的。时刻提醒自己要清除对象引用不仅麻烦，而且会让代码混乱。控制好变量的作用域，依赖正常的 GC 来做这件事情。

在例子中，Stack 自己管理内存，所以就可能出现内存管理不善的问题。所以，当需要自己管理内存时，一定要注意潜在的内存泄露问题。

## 避免使用终结方法 {#Item7}

终结方法（finalizer）通常是不可预测的，也是很危险的，一般情况下是不必要的：

- finalizer 是 GC 调用的，GC 调用时间不可预测
- 性能变差

资源回收应该显式使用 try finally。

finalizer 也有可以使用的场合，但都比较偏门，还是不要用了，真正需要的时候自然会想起来有这个东西的。
