---
title: Syntax and Runtime Checks for Qualified Types in Scala 3
author: "[Quentin Bernet](mailto:quentin.bernet@epfl.ch) @[LARA](https://lara.epfl.ch/w/), [EPFL](https://www.epfl.ch/fr/)"
---

# Slides

<!-- 
Max ~25 slides
1 min by slide
Slides to the point
commit html
can make github page
 -->

These slides can be found at [go.epfl.ch/QT-slides](https://go.epfl.ch/QT-slides).

The source code: [https://github.com/Sporarum/slides-master-project](https://github.com/Sporarum/slides-master-project)

# Introduction

::: notes

Test

:::

<!-- 
## What are terms ?

These things:

```scala {.numberLines}
"Hello"

1 + 1

(x: Int) => -x

{
  def foo(x: Int) = -x
  foo(42)
}
```

## What are types ?

These things:
```scala {.numberLines}
val exponent: Double = 2

def exp(x: Double): Double = Math.pow(x, exponent)

type Number = Int | Double

type TypeConstructor[T] = List[T]
```
## Quiz

Are the following terms or types ?

``` {.numberLines}
f
g

Letter
Number
```

## Answers


```scala {.numberLines}
type f = Float
val g = 'c'

type Letter = Char
val Number = 1000
```

What this teaches us:
We can only know from context ! -->

## What are qualified types ?

More precise types, inspired by sets builders: $\{x \in \mathbb{Z}\;|\;x > 0\}$

```scala {.numberLines}
type Pos = Int with _ > 0
4: Pos

type NonEmptyString = String with s => !s.isEmpty
"Nilla Wafer Tophat Time": NonEmptyString

type PoliteString = NonEmptyString with s =>
    s.head.isUpper &&
    s.takeRight(6) == "please"
"Pass me the butter, please": PoliteString
``` 

# Syntax

## Internal Representation

Currently:
```scala
type Pos = Int @qualified[Int]((x: Int) => x > 0)
```

## Unanimous

```scala
type Trivial = Int with true
type Empty   = Int with false

def foo(x: Int with x > 0, y: Int with y > x): Int with x > 0 = y - x
```

<!-- ## This syntax ? -->

## Postfix Lambda

The syntax we'll be using as a default

```scala
type Pos = Int with _ > 0
// desugars to
type Pos = Int with x => x > 0

type Digit = Int with x => 0 <= x && x < 10
```

## `it`

```scala
type Pos = Int with it > 0

type Digit = Int with 0 <= it && it < 10
```

In case nested qualifiers:

```scala
it < super.it + super.super.it
```

## Set Notation

```scala
type Pos = {x: Int with x > 0}

type Digit = {x: Int with 0 <= x && x < 10}
```

## `id`

```scala
type Alias = (x: Int)

type Pos = (x: Int with x > 0)
// desugars to
type Pos = (x$1: (x$2: Int) with x$2 > 0)

type Digit = (x: Int with 0 <= x && x < 10)
// desugars to
type Digit = (x$1: (x$2: Int) with 0 <= x$2 && x$2 < 10)
```

## `it` & `id`

Nothing stops us from allowing both !

Choosing which one is "true" depending on IR:

* `it` if de Bruijn indices
* `id` if identifiers

## Keyword

Existing: `if`, `with`

Or new: `where`

# Pattern Matching

## Desugars to type test and cast

:::: columns

::: column
```scala
costlyCall() match
  case i: Int =>
    i + 1

  case s: String =>
    s.toUpperCase




```
:::

::: column

```scala
val x = costlyCall()
if x.isInstanceOf[Int] then
  val i = x.asInstanceOf[Int]
  i + 1
else if x.isInstanceOf[String] then
  val s = x.asInstanceOf[String]
  s.toUpperCase
else
  throw new MatchError(x)
```

:::

::::

::: notes

Transformation done at pattern matching phase

:::


## For pattern guards

```scala
case y: Int if 0 <= y && y < 10 =>
```
::: fragment
```scala
x.isInstanceOf[Int] && {
  val y = x.asInstanceOf[Int]
  0 <= y && y < 10
}
```
:::
::: fragment
```scala
x.isInstanceOf[Int] && {
  val y = x.asInstanceOf[Int]
  0 <= y && y < 10
}
```
:::

::: notes

Transformation done at erasure phase

:::

## For qualified types

```scala
case y: Int with 0 <= y && y < 10 =>
```
::: fragment
```scala
x.isInstanceOf[Int with y => 0 <= y && y < 10]
```
:::
::: fragment
```scala
x.isInstanceOf[Int] && {
  val y = x.asInstanceOf[Int]
  0 <= y && y < 10
}
```
:::

::: notes

Transformation done at erasure phase

:::

## Example

```scala
def answer(x: Any): Either[String, String] =
  x match
    case s: PoliteString => Right("Of course !")
    case _               => Left("Please be polite ...")

type NonEmptyString = String with s => !s.isEmpty

type PoliteString = NonEmptyString with s => s.head.isUpper &&
                                             s.takeRight(6) == "please"
```

## After erasure

```scala
def answer(x: Object): scala.util.Either =
  if
    x1.isInstanceOf[String] &&
    {
      val _$1: String = x1.asInstanceOf[String]
      !(_$1.isEmpty)
    } &&
    {
      val s: String = x1.asInstanceOf[String]
      s.head.isUpper &&
      s.takeRight(6) == "please"
    }
  then
    val s: String = x1.asInstanceOf[String]
    return Right("Of course !")
  
  return Left("Please be polite ...")
```

::: notes

This is simplified from the actual output

:::

## Implementation

```scala
def transformTypeTest(expr: Tree, testType: Type, flagUnrelated: Boolean): Tree =
  testType.dealiasKeepQualifyingAnnots match {
    ...
    case refine.EventuallyQualifiedType(baseType, closureDef(qualifier: DefDef)) =>
      evalOnce(expr) { e =>
        // e.isInstanceOf[baseType] && qualifier(e.asInstanceOf[baseType])
        transformTypeTest(e, baseType, flagUnrelated)
          .and(BetaReduce(qualifier, List(List(e.asInstance(baseType)))))
      }
    ...
```
