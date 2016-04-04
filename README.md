# Enumeratum [![Build Status](https://travis-ci.org/lloydmeta/enumeratum.svg?branch=master)](https://travis-ci.org/lloydmeta/enumeratum) [![Coverage Status](https://coveralls.io/repos/lloydmeta/enumeratum/badge.svg?branch=master)](https://coveralls.io/r/lloydmeta/enumeratum?branch=master) [![Codacy Badge](https://www.codacy.com/project/badge/a71a20d8678f4ed3a5b74b0659c1bc4c)](https://www.codacy.com/public/lloydmeta/enumeratum)

A type-safe and powerful enumeration implementation for Scala with exhaustive pattern match warnings, Enumeratum is
an implementation based on a single Scala macro that searches for implementations of a sealed trait or class.

Enumeratum aims to be similar enough to Scala's built in `Enumeration` to be easy-to-use and understand while offering
more flexibility, safety, and power. It also has **zero** dependencies, which means it's light-weight, but more importantly,
won't clutter your (or your dependants') namespace.

Using Enumeratum allows you to use your own `sealed` traits/classes without having to maintain your own collection of
values, which not only means you get exhaustive pattern match warnings, but also richer enum values, and methods that
can take your enum values as arguments without having to worry about erasure (for more info, see [this blog post on Scala's
`Enumeration`](http://underscore.io/blog/posts/2014/09/03/enumerations.html))


Enumeratum has the following niceties:

- Zero dependencies
- Simplicity; most of the complexity in this lib is in the macro, and the macro is fairly simple conceptually
- No usage of `synchronized`, which may help with performance and deadlocks prevention
- No usage of reflection at run time. This may also help with performance but it means Enumeratum is compatible with ScalaJS and other
  environments where reflection is a best effort.
- All magic happens at compile-time so you know right away when things go awry


Compatible with Scala 2.10.x and 2.11.x

[Scaladocs](https://beachape.com/enumeratum/latest/api)

## SBT / Installation basics

Set the Enumeratum version in a variable (for the latest version, use `val enumeratumVersion = "1.3.7"`).

For basic enumeratum (with no Play support):
```scala
libraryDependencies ++= Seq(
    "com.beachape" %% "enumeratum" % enumeratumVersion
)
```

For enumeratum with full Play support:
```scala
libraryDependencies ++= Seq(
    "com.beachape" %% "enumeratum" % enumeratumVersion,
    "com.beachape" %% "enumeratum-play" % enumeratumVersion
)
```

#### ScalaJs

In a ScalaJS project, add the following:

```scala
libraryDependencies ++= Seq(
    "com.beachape" %%% "enumeratum" % enumeratumVersion
)
```


There are other ways to use Enumeratum (e.g. a la carte), [see here](#sbt-a-la-carte).

## How-to + example

Using Enumeratum is simple. Simply declare your own sealed trait or class `A`, and implement it as case objects inside
an object that extends from `Enum[A]` as follows.

Note that by default, `findValues` will return a `Seq` with the enum members listed in written-order (relevant if you want to
use the `indexOf` method).

```scala

import enumeratum._

sealed trait Greeting extends EnumEntry

object Greeting extends Enum[Greeting] {

  val values = findValues

  case object Hello extends Greeting
  case object GoodBye extends Greeting
  case object Hi extends Greeting
  case object Bye extends Greeting

}

// Object Greeting has a `withName(name: String)` method
Greeting.withName("Hello")

// => res0: Greeting = Hello

Greeting.withName("Haro")
// => java.lang.IllegalArgumentException: Haro is not a member of Enum (Hello, GoodBye, Hi, Bye)

import Greeting._

def tryMatching(v: Greeting): Unit = v match {
  case Hello => println("Hello")
  case GoodBye => println("GoodBye")
  case Hi => println("Hi")
}

/**
Pattern match warning ...

<console>:24: warning: match may not be exhaustive.
It would fail on the following input: Bye
       def tryMatching(v: Greeting): Unit = v match {

*/

Greeting.indexOf(Bye)
// => res2: Int = 3

```

The name is taken from the `toString` method of the particular
`EnumEntry`. This behavior can be changed in two ways. The first is
to manually override the `def entryName: String` method.

```scala

import enumeratum._

sealed abstract class State(override def entryName: String) extends EnumEntry

object State extends Enum[State] {

   val values = findValues

   case object Alabama extends State("AL")
   case object Alaska extends State("AK")
   // and so on and so forth.
}

import State._

State.withName("AL")

```

The second is to mixin the stackable traits provided for common string
conversions, `Snakecase`, `Uppercase`, and `Lowercase`.

```scala

import enumeratum._
import enumeratum.EnumEntry._

sealed trait Greeting extends EnumEntry with Snakecase

object Greeting extends Enum[Greeting] {

  val values = findValues

  case object Hello extends Greeting
  case object GoodBye extends Greeting
  case object ShoutGoodBye extends Greeting with Uppercase

}

Greeting.withName("hello")
Greeting.withName("good_bye")
Greeting.withName("SHOUT_GOOD_BYE")

```


### Play 2

The `enumeratum-play` project is published separately and gives you access to various tools
to help you avoid boilerplate in your Play project. (for Play 2.4.x support, use versions <= 1.3.7)

The included `PlayEnum` trait is probably going to be the most interesting as it includes a bunch
of built-in implicits like Json formats, Path bindables, Query string bindables,
and form field support.

For example:

```scala
package enums._

import enumeratum._

sealed trait Greeting extends EnumEntry

object Greeting extends PlayEnum[Greeting] {

  val values = findValues

  case object Hello extends Greeting
  case object GoodBye extends Greeting
  case object Hi extends Greeting
  case object Bye extends Greeting

}

/*
  Then make sure to import your PlayEnums into your routes in your Build.scala
  or build.sbt so that you can use them in your routes file.

  `routesImport += "enums._"`
*/


// You can also use the String Interpolating Routing DSL:

import play.api.routing.sird._
import play.api.routing._
import play.api.mvc._
Router.from {
    case GET(p"/hello/${Greeting.fromPath(greeting)}") => Action {
      Results.Ok(s"$greeting")
    }
}

```
### Play-JSON

The `enumeratum-play-json` project is published separately and gives you access to Play's auto-generated boilerplate
for JSON serialization in your Enum's.

For example:

```scala
package enums._

import enumeratum.{ PlayJsonEnum, Enum, EnumEntry }

sealed trait Greeting extends EnumEntry

object Greeting extends Enum[Greeting] with PlayJsonEnum[Greeting] {

  val values = findValues

  case object Hello extends Greeting
  case object GoodBye extends Greeting
  case object Hi extends Greeting
  case object Bye extends Greeting

}
```

## SBT a la carte

If you just want to use the macro that finds implementations of a sealed trait within an enclosed
tree:

```scala
libraryDependencies ++= Seq("com.beachape" %% "enumeratum-macros" % enumeratumVersion)
```

For enumeratum with [uPickle](http://lihaoyi.github.io/upickle/):

```scala
libraryDependencies ++= Seq(
    "com.beachape" %% "enumeratum" % enumeratumVersion,
    "com.beachape" %% "enumeratum-upickle" % enumeratumVersion
)
```

For enumeratum with Play JSON:
```scala
libraryDependencies ++= Seq(
    "com.beachape" %% "enumeratum" % enumeratumVersion,
    "com.beachape" %% "enumeratum-play-json" % enumeratumVersion
)
```

### ScalaJs

There is support for ScalaJs, though only for the core lib and the UPickle helper lib.

```scala
libraryDependencies ++= Seq(
    "com.beachape" %%% "enumeratum" % enumeratumVersion
)
```

To use with uPickle:

```scala
libraryDependencies ++= Seq(
    "com.beachape" %%% "enumeratum" % enumeratumVersion,
    "com.beachape" %%% "enumeratum-upickle" % enumeratumVersion
)
```

## Licence

The MIT License (MIT)

Copyright (c) 2015 by Lloyd Chan

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
