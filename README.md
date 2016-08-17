# airframe  [![Gitter Chat][gitter-badge]][gitter-link] [![CircleCI][circleci-badge]][circleci-link] [![Coverage Status][coverall-badge]][coverall-link]
Dependency injection library tailored to Scala.

[circleci-badge]: https://circleci.com/gh/wvlet/airframe.svg?style=svg
[circleci-link]: https://circleci.com/gh/wvlet/airframe
[gitter-badge]: https://badges.gitter.im/Join%20Chat.svg
[gitter-link]: https://gitter.im/wvlet/wvlet?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge
[coverall-badge]: https://coveralls.io/repos/github/wvlet/airframe/badge.svg?branch=master
[coverall-link]: https://coveralls.io/github/wvlet/airframe?branch=master


# Introduction

Airframe injects object dependencies as in [Google Guice](https://github.com/google/guice). 

With Airframe you can build objects in three steps:
- *Bind*: Describe instance types necessary in your class with `bind[X]`: 
```scala
import wvlet.airframe._

trait App {
  val x = bind[X]
  val y = bind[Y]
  // Do something with X and Y
}
```
- *Design*: Describe how to provide object instances:
```scala
val design : Design = 
   Airframe.newDesign
     .bind[X].toInstance(new X)  // Bind type X to a concrete instance
     .bind[Y].toSingleton        // Bind type Y to a singleton object
```
- *Build*: Create a concrete instance:
```scala
val app : App = design.build[App]
```

`Design` class is *immutable*, so you can easily reuse and extend the design for creating new types of objects.

The major advantages of Airframe include:
- You can describe the knowledge on how to create objects within `Design`.
  - It enables you to reuse the same design to prepare objects both in production and test code. This avoids code duplications that create instances with constructors (e.g., `new App(new X, new Y, ...)`).
  - When writing application codes, you only need to care about how to **use** objects`, rather than how to **provide** them. 
- You can mix-in Scala traits that have multiple dependencies, instead of writing constructors that have many arguments.
  - No longer need to remember the constructor argument orders.
  - You can enjoy the flexibility of Scala traits and dependency injection (DI) at the same time.


# Usage

(The whole code used in this section can be found here [AirframeTest](https://github.com/wvlet/airframe/blob/master/src/test/scala/wvlet/airframe/AirframeTest.scala))

You can inject an object with `bind` method in Airframe. Assume we want to create a service that prints a greeting at random:

```scala
import wvlet.airframe._ 
import wvlet.log.LogSupport

trait Printer {
  def print(s: String): Unit
}

// Concrete classes which will be bound to Printer
class ConsolePrinter(config: ConsoleConfig) extends Printer { 
  def print(s: String) { println(s) }
}
class LogPrinter extends Printer with LogSupport { 
  def print(s: String) { info(s) }
}

class Fortune { 
  def generate: String = { /** */ }
}
```

## Mix-in instances

A simple way to is to create a service trait which uses binding objects. Since trait can be shared multiple components and a class 
can mix-in any traits, this is a simple way to use binding objects.

```scala
trait PrinterService {
  protected def printer = bind[Printer] // It's binded any Printer mix in instances.
}

trait FortuneService {
  protected def fortune = bind[Fortune]
}

trait FortunePrinterMixin extends PrinterService with FortuneService {
  printer.print(fortune.generate)
}
```

## Local variable binding

We can bind a object to local variable explicitly.

```scala
trait FortunePrinterEmbedded {
  protected def printer = bind[Printer]
  protected def fortune = bind[Fortune]
  
  printer.print(fortune.generate)
}
```

## Tagged binding

Airframe enables us to bind multiple implementation to a trait by using object tagging.
 
 
 ```scala
 import wvlet.obj.tag.@@
 case class Fruit(name: String)
 
 trait Apple
 trait Banana
 trait Lemon

 trait TaggedBinding {
   val apple  = bind[Fruit @@ Apple]
   val banana = bind[Fruit @@ Banana]
   val lemon  = bind(lemonProvider _)
 
   def lemonProvider(f: Fruit @@ Lemon) = f
 }
 ```


## Injecting

It is necessary to define `Design` of dependency components before using binding objects. It's similar to `module` in Guice.

```scala
val design = Airframe.newDesign
  .bind[Printer].to[ConsolePrinter]  // Generated in resolved dependency components in Airframe design
  .bind[ConsoleConfig].toInstance(ConsoleConfig(System.err)) // Binding actual instance
```

Binding tagged object can be done with `@@`.

```scala
val design = Airframe.newDesign
  .bind[Fruit @@ Apple].toInstance[Fruit("apple")]
  .bind[Fruit @@ Banana].toInstance[Fruit("banana")]
  .bind[Fruit @@ Lemon].toInstance[Fruit("lemon")]
````

If we want to bind a class as singleton, `toSingleton` cab be used.

```scala
class HeavyObject extends LogSupport { /** */ }

val design = Airframe.newDesign
  .bind[HeavyOBject].toSingleton
````

We can create a object needed with `build` keyword.

```
design.build[FortunePrinterMixin]
```

See more detail in [AirframeTest](https://github.com/wvlet/airframe/blob/master/src/test/scala/wvlet/airframe/AirframeTest.scala).

# LICENSE

[Apache v2](https://github.com/wvlet/airframe/blob/master/LICENSE)