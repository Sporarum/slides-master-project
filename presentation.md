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

Add _ example outside of Qual types
Maybe "table of content" slide
-->

These slides can be found at [go.epfl.ch/QT-slides](https://go.epfl.ch/QT-slides).

The source code: [https://github.com/Sporarum/slides-master-project](https://github.com/Sporarum/slides-master-project).

::: notes

Please, questions at the end

except clarification questions

:::

# Introduction

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

<!-- Add List[Pos] example -->

<!-- So last week I wanted to be sure what it meant, so I went to google translate -->

## What are qualified types ?

More precise types, inspired by sets builders: $\{x \in \mathbb{Z}\;|\;x > 0\}$

::: fragment
```scala {.numberLines}
type Pos = Int with _ > 0
4: Pos
```
:::
::: fragment
```scala
type NonEmptyString = String with s => !s.isEmpty
"Nilla Wafer Top Hat Time": NonEmptyString
```
:::
::: fragment
```scala
type PoliteString = NonEmptyString with s =>
    s.head.isUpper &&
    s.takeRight(6) == "please"
"Pass me the butter, please": PoliteString
```
:::

## Why not contracts



## Un type qualifié, c'est quoi ?

::::: columns

:::: {.column width="10%"}

::::


:::: {.column width="40%"}


::: fragment

![](typeQualifie.PNG){ width=100% .center }

:::

::::

:::: {.column width="10%"}


::::

:::::

# Syntax

Elements:

* Base type (`String`)
* Identifier (`s`)
* Qualifier (`!s.isEmpty`)

. . .

Internal representation:

```scala
type Pos = Int @qualified[Int]((x: Int) => x > 0)
```

## Unanimous

Boolean expressions:
```scala
type Trivial = Int with true
type Empty   = Int with false
```

. . .

Available identifiers:
```scala
def foo(x: Int with x > 0, y: Int with y > x): Int = y - x
```


<!-- slide for this syntax -->

::: notes

But this doesn't work for the return type
:::

## Postfix Lambda

```scala
import scala.language.experimental.postfixLambda

type Pos = Int with x => x > 0
// or
type Pos = Int with _ > 0

type Digit = Int with x => 0 <= x && x < 10
```

. . .

But

```scala
(Int with x => x > 0) => (Int with y => y < 0) => Int
```

::: notes

That's the syntax we've been using so far

:::

## Set Notation

```scala
import scala.language.experimental.setNotation

type Pos = {x: Int with x > 0}

type Digit = {x: Int with 0 <= x && x < 10}
```

. . .

But

```scala
(x: Int) => Int with x > 0
(x: Int with x > 0) => Int with x > 0
{x: Int with x > 0} => Int with x > 0
```

<!-- What if we just added an identifier which always refers to the base type -->

## `it`

```scala
type Pos = Int with it > 0

type Digit = Int with 0 <= it && it < 10
```

. . .

(Optional) If nested qualifiers:

<!-- Add example where that is useful -->
```scala
it < super.it + super.super.it
```

<!-- What if we just allowed any type to have an identifier in front -->

## `id`

```scala
type Alias = (x: Int)
```

. . .

```scala
type Pos = (x: (y: Int) with y > 0)
```

. . .

```scala
type Pos = (x: Int with x > 0)

type Digit = (x: Int with 0 <= x && x < 10)
```

## `it` & `id`

Nothing stops us from allowing both !

. . .

```scala
type Pos = (x: Int with x > 0)
// as
type Pos = (x: (Int with it > 0))
```

. . .

```scala
type Pos = Int with it > 0
// as
type Pos = (it$1: Int) with it$1 > 0
```

::: notes

Choosing which one is "true" depending on IR:

* `it` if de Bruijn indices
* `id` if identifiers

:::
# Pattern Matching

## Is just syntactic sugar

::::: columns

:::: column
```scala
costlyCall() match
  case i: Int =>
    i + 1

  case s: String =>
    s.toUpperCase




```
::::

:::: column

::: fragment

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

:::::

::: notes

Transformation done at pattern matching phase

:::

## Pattern guards

::::: columns

:::: {.column width="50%"}

```scala
case y: Int if
    0 <= y && y < 10 =>
```
::: fragment
```scala
x.isInstanceOf[Int] && {
  val y = x.asInstanceOf[Int]
  0 <= y && y < 10
}
```
:::

::::


:::: {.column width="50%"}
<span style="color: white;">
Test
</span>

::::

:::::

## Pattern guards & qualified types

::::: columns

:::: {.column width="50%"}

```scala
case y: Int if
    0 <= y && y < 10 =>
```

```scala
x.isInstanceOf[Int] && {
  val y = x.asInstanceOf[Int]
  0 <= y && y < 10
}
```

::::

:::: {.column width="50%"}

```scala
case y: Int with
    0 <= y && y < 10 =>
```

::: fragment
```scala
x.isInstanceOf[Int with y => 
    0 <= y && y < 10]
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

::::

:::::

<!-- Either exaplain why this similarity is not exact, or not mention it -->

::: notes

Transformation done at erasure phase

:::

<!-- Needs more transition until this -->
## Ducks

If it quacks like a pattern guard, why not make it look like one:

```scala
type Pos = Int if it > 0

x match
  case (x: Int) if x > 0 =>
  case x: (Int if x > 0) =>
```

## Before

```scala
def answerRequest(x: Any): Either[String, String] =
  x match
    case s: PoliteString => Right("Of course !")
    case _               => Left("Please be polite ...")
```

. . .

```scala
type NonEmptyString = String with s => !s.isEmpty

type PoliteString = NonEmptyString with s => s.head.isUpper &&
                                             s.takeRight(6) == "please"
```

::: notes
Draw your attention to `s.head`
:::


## After

```scala
def answerRequest(x: Object): scala.util.Either =
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
def transformTypeTest(expr: Tree, testType: Type, ...): Tree =
  testType.dealiasKeepQualifyingAnnots match {
    ...
    case refine.EventuallyQualifiedType(baseType,
                             closureDef(qualifier: DefDef)) =>
      evalOnce(expr) { e =>
        transformTypeTest(e, baseType, flagUnrelated).and(
          BetaReduce(qualifier, List(List(e.asInstance(baseType)))))
      }
    ...
```

# Conclusion

* 2 Implemented syntaxes:
  postfix lambda and set notation
* 1 proposed syntax:
  `it` & `id`
* Pattern matching:
  frames qualified types are as a compiletime generalization of pattern guards

# Extra Slides

## Irrefutability

```scala
type Pos = Int with x > 0

object Extractor:
  def unapply(x: Pos): Some[Pos] = Some(x)

-1 match
  case Extractor(x) => x
  case _ => None // warning: unreachable case
```

## JSON schema

Works for both syntaxes:

```scala
case class LongitudeAndLatitudeValues(
  latitude: Double with -90 <= latitude && latitude <= 90,
  longitude: Double with -180 <= longitude && longitude <= 180,
  city: String with raw"^[A-Za-z . ,'-]+$$".r.matches(city)
) extends Obj
```

## Tensor: Postfix Lambda

```scala
type Shape = List[Int with _ > 0]

extension (s: Shape)
  def nbrElems = s.fold(1)(_ * _)
  def reduce(axes: List[Int with x => s.indices.contains(x)]) = s.zipWithIndex.filter((_, i) => axes.contains(i)).map(_._1)

trait Tensor[T]:
  val shape: Shape
  def sameShape(t: Tensor[T]): Boolean = shape.corresponds(t.shape)(_ == _)
  def add(t: Tensor[T] with t.sameShape(this)): Tensor[T] with _.sameShape(this)
  def mean(axes: List[Int with x => shape.indices.contains(x)]): Tensor[T] with _.shape == shape.reduce(axes)
  def reshape(newShape: Shape with newShape.nbrElems == shape.nbrElems): Tensor[T] with _.shape == newShape
```

## Tensor: Set Notation

```scala
type Shape = List[{x: Int with x > 0}]

extension (s: Shape)
  def nbrElems = s.fold(1)(_ * _)
  def reduce(axes: List[{x: Int with s.indices.contains(x)}]) = s.zipWithIndex.filter((_, i) => axes.contains(i)).map(_._1)

trait Tensor[T]:
  val shape: Shape
  def sameShape(t: Tensor[T]): Boolean = shape.corresponds(t.shape)(_ == _)
  def add(t: Tensor[T] with t.sameShape(this)): {res: Tensor[T] with res.sameShape(this)}
  def mean(axes: List[{x: Int with shape.indices.contains(x)}]): {res: Tensor[T] with res.shape == shape.reduce(axes)}
  def reshape(newShape: Shape with newShape.nbrElems == shape.nbrElems): {res: Tensor[T] with res.shape == newShape}
```
