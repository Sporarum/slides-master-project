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
"nilla wafer top hat time": NonEmptyString

type PoliteString = NonEmptyString with s =>
    s.head.isUpper &&
    s.takeRight(6) == "please"
"Pass me the butter, please": PoliteString
``` 

# Syntax

Elements:

* Base type
* Qualifier
* Identifier

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
::: fragment
But

```scala
(Int with x => x > 0) => (Int with y => y < 0) => Int
```
:::

::: notes

That's the syntax we've been using so far

:::

## `it`

```scala
type Pos = Int with it > 0

type Digit = Int with 0 <= it && it < 10
```

::: fragment

(Optional) If nested qualifiers:

```scala
it < super.it + super.super.it
```

:::

## Set Notation

```scala
import scala.language.experimental.setNotation
```

```scala
type Pos = {x: Int with x > 0}

type Digit = {x: Int with 0 <= x && x < 10}
```

::: fragment

But

```scala
(x: Int) => Int with x > 0
(x: Int with x > 0) => Int with x > 0
{x: Int with x > 0} => Int with x > 0
```

:::

## `id`

```scala
type Alias = (x: Int)

type Pos = (x: (y: Int) with y > 0)
// or
type Pos = (x: Int with x > 0)

type Digit = (x: Int with 0 <= x && x < 10)
```

## `it` & `id`

Nothing stops us from allowing both !

For example:
```scala
type Pos = (x: Int with x > 0)
// as
type Pos = (x: (Int with it > 0) )

type Pos = Int with it > 0
// as
type Pos = ( (it$1: Int) with it$1 > 0 )
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


::: notes

Transformation done at erasure phase

:::

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

type NonEmptyString = String with s => !s.isEmpty

type PoliteString = NonEmptyString with s => s.head.isUpper &&
                                             s.takeRight(6) == "please"
```

::: notes
Draw your attention to `s.head`
:::


## After

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

## Conclusion

* 2 Implemented syntaxes:
  postfix lambda and set notation
* 1 proposed syntax:
  `it` & `id`
* Pattern matching:
  frames qualified types are as a compiletime generalization of pattern guards
