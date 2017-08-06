# nim和面向对象(四)

## 原文链接：[http://blog.csdn.net/dajiadexiaocao/article/details/46945599](http://blog.csdn.net/dajiadexiaocao/article/details/46945599)

**以下为文章内容：**

#  Nim and OO, Part IV

**原文链接：[http://goran.krampe.se/2014/11/30/nim-and-oo-part-iv/](http://goran.krampe.se/2014/11/30/nim-and-oo-part-iv/)**

**作者：Goran Krampe**

**Roads less Taken**

**A blend of programming, boats and life.**

**时间：2014,11,30**

 正如在早期的帖子中我所描述的，当使用方法而不是静态绑定过程和泛型时Nim不支持"super call"(超类构造方法调用)。在IRC上我的文章引起了一些关于这个观点的讨论并且Andreas决定实现他早已经计划的机制-但是还没有完全确定一个好名字。


前几日这个机制进入发展分支，这意味着它将在Nim的下一个版本中成为官方的，在2014年底前我所猜想的将会实现。应当指出的是devel主要经历bug修复，所以除非你是偏执狂否则它非常有用。现在...当然我必须在我的示例代码中尝试超级构造方法调用。。。


与其称之为static_call-意味着调用解析是在编译时静态确定，名字最终成为procCall-意味着调用解析简单完成就好像对过程所做的那样-不同的词。换句话说，即使我们调用一个方法，让静态类型的参数决定调用哪一个方法，不是实际的运行时类型。


早期的文章有点重复-今天你可以选择在重载的过程中通过使用类型转换调用，所以如果当你有一个B类型的对象而你想调用有一个A参数类型的myProc时(B是A的一个子类型)，你可以使用myProc(A(b))。


这叫做一个类型转换并且可以被看做一种安全类型塑造，如果它是安全的它才会工作。Nim也有铸型但通常如果你知道你正在做什么你才能使用。:)


现在...方法不依赖于静态类型-它们的解决基于对象的实际运行时类型-这是它们存在的全部原因并且为了支持更复杂的面向对象代码这是必不可少的。所以在本身的类型转换技术只适用于在重载的过程中选择，而不是方法。


但是现在，随着添加procCall我们可以调用重载的方法使用完全相同的技术。我们的Fruit代码从早期的文章现在可以简化-我们不再需要用一个不同的名字basePrice提取calcPrice的基础实现作为一个方法。并且它依然能预期的工作，这是最新的代码：

```nim
import math

# Dollars and Kgs
type
  Dollar* = distinct float
  Kg* = distinct float

proc `$`*(x: Dollar) :string =
  $x.float

proc `==`*(x, y: Dollar) :bool {.borrow.}


# Fruit class
type
  Fruit* = ref object of RootObj
    origin*: string
    price*: Dollar

method `$`*(self: Fruit): string =
  self.origin & " " & $self.price

method reduction(self: Fruit) :Dollar =
  Dollar(0)


method calcPrice*(self: Fruit): Dollar =
  Dollar(round(self.price.float * 100)/100 - self.reduction().float)


# Banana class
type
  Banana* = ref object of Fruit
    size*: int

method reduction(self: Banana): Dollar =
  Dollar(9)

method calcPrice*(self: Banana): Dollar =
  procCall Fruit(self).calcPrice()

# Pumpkin
type
  Pumpkin* = ref object of Fruit
    weight*: Kg

method reduction(self: Pumpkin): Dollar =
  Dollar(1)

method calcPrice*(self: Pumpkin) :Dollar =
  Dollar(procCall(Fruit(self).calcPrice()).float * self.weight.float)


# BigPumpkin
type
  BigPumpkin* = ref object of Pumpkin

method calcPrice*(self: BigPumpkin) :Dollar =
  Dollar(1000)


# Construction procs
proc newPumpkin*(weight, origin, price): Pumpkin = Pumpkin(weight: weight, origin: origin, price: price)
proc newBanana*(size, origin, price): Banana = Banana(size: size, origin: origin, price: price)
proc newBigPumpkin*(weight, origin, price): BigPumpkin = BigPumpkin(weight: weight, origin: origin, price: price)
```

正如我们所见"句法风格"可以与往常大不一样，在第40行我们可以不带括号使用procCall，但是在第51行我们需要在一个"调用风格"中使用它为了工作优先级。


所以...你可能疑问，为什么没有正如在Smalltalk和Jave等中的一些东西如super?这个原因是相当简单的，Nim支持多调度-换句话说在Nim中我们可以基于几种参数的类型调度，只要它们是对象。方法调度中没有特定的参数，它们享有"self"特权，以公约作为接收器我们仅倾向于使用第一个参数。这也意味着"super"在Nim中没有合理的意义，super是谁?


类型转换应该与procCall一起使用并明确的阐明"超级类型"(在我们上面的例子中-"Fruit")，因此是它非常清楚你想调用哪个方法。一个清教徒可能会宣称它的双亲类有太多它的父类(相比更加抽象的非耦合的"super")。但是我认为这显示的类型更好的适合Nim的观念模式-并且即使这使改变父类有一点乏味(你需要搜索直到找到procCalls并且固定类型转换)-一些未来重构的工具不能轻易的解决并无大碍，并且无论如何改变父类不是你每分钟都要做的事。


你也可以说程序员的意图有点失去了这个意图。它没有明显的误读为"调用超级实现",并且我猜想我们会发现编码器使用这也抛出一个小评论说明，hey，这是一个超级调用。


该机制也能够"超级超级"因此你可以显示的跳过中间类，当然它一般不是好风格，但是同时，至少显示的遵循以下原则：)

## Calling overloads on self


我们可以很容易有另一种方案，一个foo类有多个重载了一个方法，在foo类中它们中的一个或多个想要调用"主要"执行的行为。实际上技术上这是与"超级调用"相同的方案，并且将相同的方式解决-它仅是多调度的一个例子，在例子中我们调度超过第一个参数。比方说我们有一个Foo类，具有一系列水果名称，我们想能够给它添加水果名称通过发送各种不同的对象，Fruits是明显的一个，但是也收集Fruits等。

```nim
import fruitmethod

# Foo class
type
  Foo* = ref object of RootObj
    fruits*: seq[string]

# The core method we want to call
method add(self: Foo, fruit: Fruit) =
  echo "   ...and here we actually add it"
  self.fruits.add($fruit)

method add(self: Foo, banana: Banana) =
  echo "Adding a Banana"
  procCall self.add(Fruit(banana))

method add(self: Foo, pumpkin: Pumpkin) =
  echo "Adding a Pumpkin"
  procCall self.add(Fruit(pumpkin))

method add(self: Foo, fruits: seq[Fruit]) =
  echo "Adding a bunch of Fruit"
  for f in fruits:
    self.add(f)

# This is a constructor proc that is normally used to initialize objects
proc newFoo(): Foo =
  result = Foo(fruits: newSeq[string]())

when isMainModule:
  # Get us a Foo
  var foo = newFoo()

  # Add a single Banana
  foo.add(newBanana(size = 0, origin = "Liberia", price = 20.Dollar))

  # Create a seq of Fruit
  var s = newSeq[Fruit]()
  var b = newBanana(size = 0, origin = "Liberia", price = 20.00222.Dollar)
  var p = newPumpkin(weight = 5.2.Kg, origin = "Africa", price = 10.00111.Dollar)
  s.add(b)
  s.add(p)

  # Add them all, this will first call the method for a seq of Fruit,
  # and that method will in turn call the one for Banana and the one for Pumpkin
  # and those in turn will use procCall to call the primary method for Fruit.
  foo.add(s)
```

正如我们上面所看到的我们需要在第15行和第19行使用procCall为了能够调用add方法，它从其他的add方法取得一个Fruit。所以，这不仅对使用经典的超级调用是有用的。


## Conclusion


procCall机制用方法解决了超级调用问题，并且它也解决了这种情况下的类似的使用情况。据我所能说的，这为能够在Nim中做严谨的面向对象清除了最后的障碍。


Go Nim!

Posted by Göran Krampe Nov 30th, 2014 [Languages](http://goran.krampe.se/category/languages/), [Nim](http://goran.krampe.se/category/nim/), [Nimrod](http://goran.krampe.se/category/nimrod/), [OOP](http://goran.krampe.se/category/oop/), [Object orientation](http://goran.krampe.se/category/object-orientation/), [Programming](http://goran.krampe.se/category/programming/)

至此，Goran Krampe的关于nim和面向对象的四部分内容都已经翻译完毕，在这里的相关博客只做了翻译工作，由于我对相关知识的理解较浅，所以并未发表自己的相关见解，且译文中肯定会存在一些问题，有兴趣的可深入研究，也欢迎提出不当之处。