# nim和面向对象(二) 

## 原文链接：[http://blog.csdn.net/dajiadexiaocao/article/details/46848175](http://blog.csdn.net/dajiadexiaocao/article/details/46848175)

**以下是文章内容：**

# Nim and OO, Part II

**原文链接：http://goran.krampe.se/2014/10/31/nim-and-oo-part-ii/**

**作者：Goran Krampe**

**Roads Less Taken
A blend of programming, boats and life.**

**时间：2014.10.31**

在前一篇文章中，当我探索在nim中的面向对象时，在我的Fruit例子中我留了一个球位。这是一个后续。


在那篇文章中，我们第一次实现了一些Fruit"类"混合方法和过程。然后我呈现了一个仅仅使用方法的整洁版本，它有一个轻巧的模版为了重用一个基本的方法。这个模版是需要的，对于多元化由于nim目前没有一个如Dylan或CLOS有的"call-next-method-matching"机制。这正在讨论中，我认为大家都会同意这里需要一些机制以至于在所有的多元匹配中你可以调用一个"下一个较小的匹配"。


但是我也写了那个例子仅仅使用泛型和过程可以工作的更好，因此确保静态绑定和最大速度。但是对于过程依然存在"super call"问题，以及该模版黑客仅仅是一个黑客。在多次实验后，现在我认为我找到了一个合适的nim方式来解决这个问题，所以让我们一起看看。。。

## 泛型
在nim中我们可以使用泛型来生成多实例的过程，对于所有不同静态类型的组合被用来调用这个过程。确切的说，编译器没有必要必须生成多复制，它可以使用一个case开关类型处理它，或者无论如何-但是它的一个合理的构思模型发生了什么。另一个构思模型是一个泛型过程，只要你的编译，将会运行在给定调用点的类型参数。

所以，如果我们有一个过程声明为proc calcPrice(self): Dollar对于self没有特定的类型，然后nim将认为与calcPrice[T](self: T): Dollar是相同的，并且会搜集所有的调用点，对于self的每一个独特的静态类型都将被传递，它会生成一个匹配的calcPrice过程。据推测手动的写所有的这些过程会导致相同的代码。

这是我最后的相当不错的泛型变量的Fruit库代码：

```nim
import math

# Dollars and Kgs
type
  Dollar* = distinct float
  Kg* = distinct float

proc `$`*(x: Dollar) :string =
  $x.float

proc `==`*(x, y: Dollar) :bool {.borrow.}


# Fruit class, all procs for Fruit are generic
type
  Fruit* = ref object of RootObj
    origin*: string
    price*: Dollar

# Default reduction is 0
proc reduction*(self) :Dollar =
  Dollar(0)

# This one factored out from calcPrice below
# so that subclasses can "super" call it.
proc basePrice*(self): Dollar =
  Dollar(round(self.price.float * 100)/100 - self.reduction().float)

# Default implementation, if you don't override.
proc calcPrice*(self) :Dollar =
  self.basePrice()


# Banana class, relies on inherited calcPrice()
type
  Banana* = ref object of Fruit
    size*: int

# Overrides reduction though.
proc reduction*(self: Banana): Dollar =
  Dollar(9)


# Pumpkin, overrides calcPrice() and makes a "super call" by calling basePrice()
type
  Pumpkin* = ref object of Fruit
    weight*: Kg

# Also overrides reduction.
proc reduction*(self: Pumpkin): Dollar =
  Dollar(1)

# Override to multiply super implementation by weight.
# Calling basePrice works because it will not lose type of self so
# when basePrice() calls self.reduction() it will resolve properly.
proc calcPrice*(self: Pumpkin) :Dollar =
  Dollar(self.basePrice().float * self.weight.float)


# BigPumpkin, overrides calcPrice() without super call
# No override of reduction.
type
  BigPumpkin* = ref object of Pumpkin

method calcPrice*(self: BigPumpkin) :Dollar =
  Dollar(1000)


# Construction procs, this encapsulates our structure of the objects in this module.
proc newPumpkin*(weight, origin, price): Pumpkin = Pumpkin(weight: weight, origin: origin, price: price)
proc newBanana*(size, origin, price): Banana = Banana(size: size, origin: origin, price: price)
proc newBigPumpkin*(weight, origin, price): BigPumpkin = BigPumpkin(weight: weight, origin: origin, price: price)
```

在我提出上面看似非常简单的代码之前-我实验用一个构图方式使用代表团而不是继承，以及也使用了泛型。它工作了，但是它变得相当混乱。上面的解决方法，另一个方面感觉像一个简单的模式，你可以使用它解决这样"super calll"的问题。现在...又是什么问题？:)

问题：当你仅仅使用过程以及重载过程，为Banana, Pumpkin and BigPumpkin实现calcPrice()时-并且在基类Fruit中也有一个基本默认的实现-然后你不能从其他任何中调用Fruit中的基础实现。换句话说，你不能使用一个"super call",只能试试看以及查看无限递归。

在基础的实现中，为了self理论上我们可以使用Fruit类型(不是一个泛型self)，并且使用一个自身的类型转换，如Fruit(self).calcPrice()为了能够调用它-但是然后你将运行在Fruit中的sel成为一个Fruit的calcPrice()！那是不好的，因为当它随后调用self.reduction()时，它将不能决定合适的过程，它将仅能够决定Fruit中的基础实现。所以，在过程的基础实现中，对于self参数忘记使用抽象类型作为类型。

解决方法：我们可以用另一个名字提取calcPrice()的基础实现，比如basePrice()。然后我们可以很容易的从子类中调用它。以及在Fruit中calcPrice()的默认实现将调用basePrice()。这是很重要的：在Fruit类中对于过程参数我们使用泛型self，以至于当我们调用这些过程时，我们不会丢失类型信息。

这是它的测试代码看看正如期望的那样工作：

```nim
import fruit

# Create some objects using proper encapsulated construction procs 
var p = newPumpkin(weight = 5.2.Kg, origin = "Africa", price = 10.00111.Dollar)
var b = newBanana(size = 0, origin = "Liberia", price = 20.00222.Dollar)
var bp = newBigPumpkin(weight = 15.Kg, origin = "Africa", price = 10.Dollar)

# Check correct pricing
assert(b.calcPrice() == 11.0.Dollar)
assert(p.calcPrice() == (9*5.2).Dollar)
assert(bp.calcPrice() == 1000.Dollar)

proc testing(x) :Dollar =
  result = x.calcPrice()
  echo($result)

# Check correct pricing when called via a proc with generic type
assert(testing(b) == 11.Dollar)
assert(testing(p) == (9*5.2).Dollar)
assert(testing(bp) == 1000.Dollar)

echo "All good."
```

## 结论
怎样在nim中使用泛型，对于已经很长一段时间没有使用泛型的一些人比如我，是一场头脑风暴。但是经过几个消失的折腾之后，它开始变得顺手。‘

我猜想上面的唯一问题-事实上是默认的calcPrice()对basePrice()产生了一个额外的调用（但是仅仅是默认的，所以仅对于Bananas）所以一些周期消失除非nim优化了它。当然一个模版可能会解决一些事情。

Posted by Göran Krampe Oct 31st, 2014  [Languages](http://goran.krampe.se/category/languages/), [Nim](http://goran.krampe.se/category/nim/), [Nimrod](http://goran.krampe.se/category/nimrod/), [OOP](http://goran.krampe.se/category/oop/), [Object orientation](http://goran.krampe.se/category/object-orientation/), [Programming](http://goran.krampe.se/category/programming/)