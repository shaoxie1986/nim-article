# nim和面向对象(三)

## 原文链接：[http://blog.csdn.net/dajiadexiaocao/article/details/46906721](http://blog.csdn.net/dajiadexiaocao/article/details/46906721)

**以下是文章内容：**

**原文链接:[http://goran.krampe.se/2014/10/31/nim-and-oo-part-iii/](http://goran.krampe.se/2014/10/31/nim-and-oo-part-iii/)**

**作者：Goran Krampe**

**Roads less Taken**

**A blend of programming, boats and life.**

**时间：2014,10,31**

因此在前面一篇文章Nim and OO Part II中，我们看到在Nim中我们怎样仅仅使用过程和泛型解决"super call"问题的。这意味着所有的代码都是静态绑定。

但是如果你已经读过所有的文章，你了解我已经尝试探索面向对象的更合适的机制-它被称作方法。在nim中一个过程是一个正则静态绑定函数，快速并且简单。在另一个方面一个方法使用对所有对象的运行时类型多参数动态调度。在nim中使用对象最简单的方式(带有继承的行为)就是使用方法-但是当然，这意味着动态查找有一个运行时花费，但是正如我们所看到的它相当小。

带有方法的fruit代码是这样的:

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

method reduction(self: Fruit) :Dollar =
  Dollar(0)

# Code broken out enabling super call of it
method basePrice(self: Fruit): Dollar =
  Dollar(round(self.price.float * 100)/100 - self.reduction().float)

method calcPrice*(self: Fruit): Dollar =
  self.basePrice()


# Banana class
type
  Banana* = ref object of Fruit
    size*: int

method reduction(self: Banana): Dollar =
  Dollar(9)

method calcPrice*(self: Banana): Dollar =
  self.basePrice()

# Pumpkin
type
  Pumpkin* = ref object of Fruit
    weight*: Kg

method reduction(self: Pumpkin): Dollar =
  Dollar(1)

method calcPrice*(self: Pumpkin) :Dollar =
  Dollar(self.basePrice().float * self.weight.float)


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

下面相同的代码仅使用过程和泛型。这次我依然使用正式的fn[T](self: T)而不是fn(self)-Andreas认为它看起来是"草率的"，没有合适的签名:).所以再次：

```nim
import math

# Dollars and Kgs
type
  Dollar* = distinct float
  Kg* = distinct float

proc `$`*(x: Dollar) :string =
  $x.float

proc `==`*(x, y: Dollar) :bool {.borrow.}


# Fruit class, procs are generic
type
  Fruit* = ref object of RootObj
    origin*: string
    price*: Dollar

# Default reduction is 0
proc reduction*[T](self: T) :Dollar =
  Dollar(0)

# This one factored out from calcPrice
# so that subclasses can "super" call it.
proc basePrice[T](self: T): Dollar =
  Dollar(round(self.price.float * 100)/100 - self.reduction().float)

# Default implementation, if you don't override.
proc calcPrice*[T](self: T) :Dollar =
  self.basePrice()


# Banana class, relies on inherited calcPrice()
type
  Banana* = ref object of Fruit
    size*: int

proc reduction*(self: Banana): Dollar =
  Dollar(9)


# Pumpkin, overrides calcPrice() and calls basePrice()
type
  Pumpkin* = ref object of Fruit
    weight*: Kg

proc reduction*(self: Pumpkin): Dollar =
  Dollar(1)

# Override to multiply super implementation by weight.
# Calling basePrice works because it will not lose type of self so
# when it calls self.reduction() it will resolve properly.
proc calcPrice*(self: Pumpkin) :Dollar =
  Dollar(self.basePrice().float * self.weight.float)


# BigPumpkin, overrides calcPrice() without super call
# No reduction.
type
  BigPumpkin* = ref object of Pumpkin

proc calcPrice*(self: BigPumpkin) :Dollar =
  Dollar(1000)


# Construction procs
proc newPumpkin*(weight, origin, price): Pumpkin = Pumpkin(weight: weight, origin: origin, price: price)
proc newBanana*(size, origin, price): Banana = Banana(size: size, origin: origin, price: price)
proc newBigPumpkin*(weight, origin, price): BigPumpkin = BigPumpkin(weight: weight, origin: origin, price: price)
```

现在...相同的技术被用来解决"super call"(超类基函数的调用)问题：仅仅用一个不同名字的方法或者过程提取公因子代码，以至于你可以从子类调用它。如果你执意只使用过程-在那个分解方法中泛型是重要的，否则子序列的self调用将不会适当的解决。


今天与Andreas讨论他提出了一个很好的建议-你可以使用私有的过程，公共的方法。


对于私有的行为它很容易使过程静态绑定正确。因为你知道所有的调用点，它们在这个模块。所以在这个例子中，由于它的私有性你可以很容易的创造一个basePrice()过程。公共协议是更好的保持的方法，既然我们没有控制所有的调用点以及被传递的变量类型。当你稍后在这篇文章中看到，它将变得一目了然。

## Andreas and static_call to the rescue

能够"super calls"的分解技术在未来将会不需要。Andress旨在添加static_call机制以至于你可以使用"select"你想调用的那个方法。

UPDATE: See (part IV)[2014/11/30/nim-and-oo-part-iv] about this.

现在通过使用类型转换你可以选择任何的过程，所以如果你想调用带有一个A的myProc而你有一个b: B在手中时，你仅用做myProc(A(b))。但是方法不依赖静态类型-它们决心依赖对象的运行时类型，所以这个技术目前仅作用于在重载过程中选择，而不是方法。

这个主意是在方法调用前使用static_call myMethod(A(b))将会造成Nim使用调用点的静态类型去决定适合的方法而不是运行时类型。所以静态名称应用静态解析而不是动态解析方法通常是这样做的。

这个static_call使你调用什么方法非常清楚并且恕我直言这听起来比"调用下一个方法"更明确-许多其他的语言(CLOS, Dylan, Julia)看起来也在使用这样的方法。我猜想那些其他的语言力求不是"硬链接"选择合适的类型，因此保存"超级类型"适应继承的重构，但是。。。它似乎依然有些太automagic-gotcha-there-buddy。

## performance

我设计了一个愚蠢的参照基准为Bananas,PumpkinsBig以及Pumpkins在一个单独的seq[Fruit]中各创造了5百万个，并且每5百万在每种单独的序列类型。然后遍历这些项并计算总和，并且做10次以上只是为了获得更精确的数字。

如果基于对象对于过程你有一个异构集-然后你需要做的是“手动式测试”代码的使用为了知道调用什么：

```nim
# Loop over fruits, darnit! Need to do type check and conversion
for f in fruits:
  if f of Banana:
    total += Banana(f).calcPrice.float
  # Note that you need to check for BigPumpkin first! `of` is true for all subtypes.
  elif f of BigPumpkin:
    total += BigPumpkin(f).calcPrice.float
  elif f of Pumpkin:
    total += Pumpkin(f).calcPrice.float
```

当然如果你有基于对象的方法你不需要这样做。速度?事实证明，当你使用方法时上面的手动检查比通过Nim的自动解决慢10%。

但是对于方法的惩罚可以看到同质集合-对于带有静态绑定的过程这些循环运行约快2倍。两个听起来很大的一个因素，但事实上-我认为那是一个非常令人难忘的小数目!对于纯静态代码，编译器可以做很多优化。

下面是用于装配方法基础代码的代码，对于过程或者泛型的代码仅仅是相同的但与上述类型测试和转换代码中添加fruits循环：

```nim
import fruitmethod, math, future

# Create LOTS of fruit in a heterogenous seq, sum up their costs.
# Also create three separate seqs, and sum up, to be fair comparing with procs.

# Util to measure time
import times, os
template time(s: stmt): expr =
  let t0 = cpuTime()
  s
  cpuTime() - t0


# A heterogenous seq collection typed as Fruit
var fruits = newSeq[Fruit]()

# And some homogenous
var bananas = newSeq[Banana]()
var pumpkins = newSeq[Pumpkin]()
var bigPumpkins = newSeq[BigPumpkin]()

# Fill em up
for i in 1..5000000:
  fruits.add(newBanana(size = 0, origin = "Liberia", price = 20.00222.Dollar))
  fruits.add(newPumpkin(weight = 5.2.Kg, origin = "Africa", price = 10.00111.Dollar))
  fruits.add(newBigPumpkin(weight = 15.Kg, origin = "Africa", price = 10.Dollar))
  bananas.add(newBanana(size = 0, origin = "Liberia", price = 20.00222.Dollar))
  pumpkins.add(newPumpkin(weight = 5.2.Kg, origin = "Africa", price = 10.00111.Dollar))
  bigPumpkins.add(newBigPumpkin(weight = 15.Kg, origin = "Africa", price = 10.Dollar))


var total: float = 0

proc calcTenTimes() =
  for i in 1..10:
    # Loop over fruits, our classes use methods so we do not need to cast
    # f from Fruit to specific types.
    for f in fruits:
      total += f.calcPrice.float

proc calcTenTimesSame() =
  for i in 1..10:
    total = 0
    for f in bananas:
      total += f.calcPrice.float
    for f in pumpkins:
      total += f.calcPrice.float
    for f in bigPumpkins:
      total += f.calcPrice.float


echo "Time for calc: " & $time(calcTenTimes())
echo "Time for calc same: " & $time(calcTenTimesSame())
```

运行它：

```sh
time ./shopmethod
Time for calc: 1.591322
Time for calc same: 3.068159000000001

real  0m8.342s
user  0m7.163s
sys   0m1.106s
```

注：我的运行结果与之不同(与循环次数有关)

for i in 1..5000000

结果是:

out of memory

Error: execution of an external program failed

for i in 1..500000

结果是：

Time for calc: 2.501571

Time for calc same: 2.818261

for i in 1..50000

结果是:

Time for calc: 0.256148

Time for calc same: 0.279489

## 结论

首先，Nim是快速的!创建30000000个对象，在一些集合中动态增长。然后在它们中循环10次大约调用calcPrice()300000000次?而这一切仅花费8秒?我印象深刻。

第二，方法的惩罚事实上非常小-但是如果你碰巧有一个在一个紧凑的循环中确实做数以百万计调用的代码-当然，过程将会更快。但是我的猜测是对于大多数真实的代码-这种差异甚至不会被注意到。

Andress也告诉过我，方法查找确实是非常优化的，并且甚至比在c++中的更快。所以对于我的对象我将使用方法，特别对于公共行为-只是快乐。

最后，为什么在stdlibs中没有使用方法?引用Andreas的话:"为什么我们不在stdlib库中使用方法的主要原因是它们不能与效果系统兼容的很好并且我们的死代码消除优化"。

当然。记住，Nim是用于系统级编程。所以有stdlibs是"完全静止"本质上是很重要的。

Go Nim!

Posted by Göran Krampe Oct 31st, 2014  [Languages](http://goran.krampe.se/category/languages/), [Nim](http://goran.krampe.se/category/nim/), [Nimrod](http://goran.krampe.se/category/nimrod/), [OOP](http://goran.krampe.se/category/oop/), [Object orientation](http://goran.krampe.se/category/object-orientation/), [Programming](http://goran.krampe.se/category/programming/)

