---
title: Syntax and Runtime Checks for Qualified Types in Scala 3
author: "[Quentin Bernet](mailto:quentin.bernet@bluewin.ch) @[LARA](https://lara.epfl.ch/w/), [EPFL](https://www.epfl.ch/fr/)"
---

# Slides

These slides can be found at [go.epfl.ch/QT-slides](https://go.epfl.ch/QT-slides).

Their source code: [Sporarum/slides-master-project](https://github.com/Sporarum/slides-master-project) on github.

. . .

<br>
<br>

Implementation PRs: [go.epfl.ch/QT-code](https://go.epfl.ch/QT-code)

::: notes

Please, questions at the end

except clarification questions

:::

# Introduction

::::: columns

:::: {.column width="10%"}

Our mascot:

_Le type qualifié_,

The qualified guy

::::


:::: {.column width="40%"}



![](typeQualifie_noOutline.PNG){ width=100% .center }


::::

:::: {.column width="10%"}

::::

:::::

## What are qualified types ?

More precise types, inspired by sets builders: $\{x \in \mathbb{Z}\;|\;x > 0\}$

. . .

```scala {.numberLines}
type Pos = Int with _ > 0
4: Pos
```

. . .

```scala
type NonEmptyString = String with s => !s.isEmpty
"Nilla Wafer Top Hat Time": NonEmptyString
```

. . .

```scala
type PoliteString = NonEmptyString with s =>
    s.head.isUpper &&
    s.takeRight(6) == "please"
"Pass me the butter, please": PoliteString
```
:::

## Why not contracts ?

```scala
def divideQualified(x: Int, y: Int with y != 0):
  Int with _ * y == x
```

. . .

```scala
def divideContract(x: Int, y: Int): Int = {
  require(y != 0)
  ???
}.ensuring(_ * y == x)
```

## Signature and code are mixed

```scala
def divideQualified(x: Int, y: Int with y != 0):
  Int with _ * y == x
```

. . .

```scala
def divideContract(x: Int, y: Int): Int = {
  require(y != 0)
  ???
}.ensuring(_ * y == x)
```

## Re-use predicates

```scala
type NonZero = Int with _ != 0
def divideQualified(x: Int, y: NonZero):
  Int with _ * y == x
```

. . .

```scala
def nonZero(y: Int) = y != 0
def divideContract(x: Int, y: Int): Int = {
  require(nonZero(y))
  ???
}.ensuring(_ * y == x)
```

::: notes

Separation of signature and code

Thus allows implementation to be abstract

:::

## Predicate polymorphism

```scala
def smallest[T](l: List[T]): T

def smallestPos(l: List[Pos]) = smallest(l)
```

. . .

```scala
def smallestPosContract(l: List[Int]): Int = {
  require(l.forall(_ > 0))
  ???
}.ensuring(_ > 0)
```

## In Brief

Looks like:

```scala
type Pos = Int with _ > 0
```

Behaves like contracts

Powerful thanks to polymorphism

## The compiler

::: fragment
Main phases:
:::

::: incremental

* Parser
* Typer
* Pattern Matcher
* Qualifier Checker
* Erasure
* Bytecode generator

:::

# Our work

Trying to make qualified types approachable:

::: incremental
* Finding a friendly syntax
  * 2 implemented syntaxes
  * 1 proposed syntax
* Runtime checks & Pattern matching
  * `isInstanceOf`, `asInstanceOf`
  * comparison with pattern guards
:::
. . .

Even without a solver, that's already enough for some applications !

# Syntax

```scala
type NonEmptyString = String with s => !s.isEmpty
```

Elements:

::: incremental

* Base type (`String`)
* Identifier (`s`)
* Qualifier (`!s.isEmpty`)

:::

. . .

Internal representation:

```scala
String @qualified[String]((s: String) => !s.isEmpty)
```

::: notes

TODO: Is it important to show the internal representation ?

:::

## Consensus

::: fragment
_Naming is hard_ => Don't do it if you don't have to
:::

. . .

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

## Postfix Lambda

```scala
import scala.language.experimental.postfixLambda

type Pos = Int with x => x > 0
// or
type Pos = Int with _ > 0

type Digit = Int with x => 0 <= x && x < 10
```

. . .

```scala
type IncreasingPair = (Int, Int) with _ < _
// desugared
type IncreasingPair = (Int, Int) with (x, y) => x < y
// untupled to
type IncreasingPair = (Int, Int) with p => p._1 < p._2
```

## Postfix Lambda Problems

```scala
(Int with x => x > 0) => (Int with y => y < 0) => Int
```

. . .

```scala
val f = x => x > 0

type Valid = Int with x => x > 0
```
. . .
```scala
type Invalid = Int with f
// desugars to
type Invalid = Int with x => f
```

::: notes

That's the syntax we've been using so far

:::

## Set Notation

Inspired by math $\{x \in \mathbb{Z}\;|\;x > 0\}$ <br>
And other programming languages `{v:Int | v > 0}`

```scala
import scala.language.experimental.setNotation

type Pos = {x: Int with x > 0}

type Digit = {x: Int with 0 <= x && x < 10}
```

## Set Notation Situations

```scala
(x: Int) => x.type
```
. . .
```scala
(x: Int with x > 0) => x.type
```
. . .
```scala
{x: Int with x > 0} => x.type
```

# Syntax Proposal

## `it`

Simple idea: a more legible `_` that you can repeat

. . .

```scala
type Pos = Int with it > 0

type Digit = Int with 0 <= it && it < 10
```

. . .

```scala
(Int with it > 0) => (Int with it < 0) => Int
```

. . .

```scala
(x: Int with x > 0) => Int with x > 0
    Int with it > 0 => Int with x > 0

```


## `it` may be nested

```scala
type Outer = Int with
  type Smaller = Int with it < it?

  ???
```

## `it` is actually fine

```scala
type Outer = Int with
  type Smaller = Int with it < super.it

  ???
```

::: notes

`Outer.it` would not work, as we could be in a type application:

```scala
List[
  Int with
    type Half = Int with it == super.it / 2

    ???
]
```

:::

## `id`

Different simple idea: identifiers for any type

```scala
type Function = (x: Int) => x.type
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

They are compatible, and can even be translated one into the other:

. . .

`id` to `it`:

```scala
type Pos = (x: Int with x > 0)
// transformed into
type Pos = (x: (Int with it > 0))
```

. . .

`it` to `id`:

```scala
type Pos = Int with it > 0
// transformed into
type Pos = (it$1: Int) with it$1 > 0
```

# Runtime Checks

::: fragment
Type test:

```scala
x.isInstanceOf[T] // Boolean
```
:::

. . .

<br>
Cast:

```scala
x.asInstanceOf[T] // T
```

## Type test on qualified type

```scala
x.isInstanceOf[Int with _ > 0]
// after erasure
x.isInstanceOf[Int] && x > 0
```
. . .

... won't work:

```scala
{println("❤️");}.isInstanceOf[Int with _ > 0]
// after erasure
{println("❤️");}.isInstanceOf[Int] && ({println("❤️");} > 0)
```

## Type test on qualified type for real

```scala
{println("❤️");}.isInstanceOf[Int with _ > 0]
// after erasure
val fresh1 = {println("❤️");}
```
. . .
```scala
fresh1.isInstanceOf[Int] && {
  val fresh2 = fresh1.asInstanceOf[Int]
  fresh2 > 0
}
```

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

# Pattern Matching

## Is just syntactic sugar

::::: columns

:::: column
```scala
costlyCall() match
  case i: Int if i > 0 =>
    i + 1
```
::::

:::: column

::: fragment

```scala
val x = costlyCall()
if x.isInstanceOf[Int] && {
  val i = x.asInstanceOf[Int]
  i > 0
} then
  val i = x.asInstanceOf[Int]
  i + 1
else
  throw new MatchError(x)
```

:::

::::

:::::

::: notes

Transformation done at pattern matching phase

:::

## Implementation

::: fragment
<br>
<br>
<br>
<br>

_This slide intentionally left blank_

:::

## Pattern guards & qualified types

::::: columns

:::: {.column width="50%"}

```scala
case y: Int if
    0 <= y && y < 10 =>
```
::: fragment
```scala
if x.isInstanceOf[Int] && {
  val y = x.asInstanceOf[Int]
  0 <= y && y < 10
} then ...
```
:::

::::

:::: {.column width="50%"}

::: fragment
```scala
case y: Int with
    0 <= y && y < 10 =>
```
:::

::: fragment
```scala
if x.isInstanceOf[Int with y => 
    0 <= y && y < 10]
then ...
```
:::
::: fragment
```scala
if x.isInstanceOf[Int] && {
  val y = x.asInstanceOf[Int]
  0 <= y && y < 10
} then ...
```
:::

::::

:::::

::: notes

Transformation done at erasure phase

:::

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
Draw attention to `s.head`
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

## Ducks

If it quacks like a pattern guard, why not make it look like one:

. . .

```scala
type Pos = Int if it > 0
```
. . .
```scala
x match
  case (x: Int) if x > 0 =>
  case x: (Int if x > 0) =>
```
. . .

Flow typing for free !


# Syntax + Checks =

::: fragment
```scala
type Nat = Int with _ >= 0

def log(x: Nat): Int = ???
```
:::

. . .
```scala
def logUnsafe(x: Int) =
  log(x) // error
```
. . .
```scala
def logSafe(x: Int) = x match
  case x: Nat => Some(log(x))
  case _      => None
```

# Conclusion

::::: columns

:::: column

::: incremental

* Implemented syntaxes
  * postfix lambda: `Int with _ > 0`
  * set notation: `{x: Int with x > 0}`
  Proposed syntax
  * `it`: `Int with it > 0`
  * `id`: `(x: Int) with x > 0`
  * And potentially `if` as keyword
* Pattern matching:
  * qualified types as "compiletime pattern guards"

:::

::: fragment
Future work: Solver
:::

::::


:::: {.column width="30%"}

![](typeQualifie_noOutline.PNG){ width=100% .center }

::::

:::::

# Extra Slides

## JSON schema

Works for both syntaxes:

```scala
case class LongitudeAndLatitudeValues(
  latitude: Double with -90 <= latitude && latitude <= 90,
  longitude: Double with -180 <= longitude && longitude <= 180,
  city: String with raw"^[A-Za-z . ,'-]+$$".r.matches(city)
) extends Obj
```

## Shape: Postfix Lambda

```scala
type Shape = List[Int with _ > 0]

extension (s: Shape)
  def nbrElems = s.fold(1)(_ * _)
  def reduce(axes: List[Int with x => s.indices.contains(x)]) = s.zipWithIndex.filter((_, i) => axes.contains(i)).map(_._1)
```

## Tensor: Postfix Lambda

```scala
trait Tensor[T]:
  val shape: Shape
  def sameShape(t: Tensor[T]): Boolean = ???
  def add(t: Tensor[T] with t.sameShape(this)):
    Tensor[T] with _.sameShape(this)

  def mean(axes: List[Int with x => shape.indices.contains(x)]):
    Tensor[T] with _.shape == shape.reduce(axes)

  def reshape(newShape: Shape with newShape.nbrElems == shape.nbrElems):
    Tensor[T] with _.shape == newShape
```

## Shape: Set Notation

```scala
type Shape = List[{x: Int with x > 0}]

extension (s: Shape)
  def nbrElems = s.fold(1)(_ * _)
  def reduce(axes: List[{x: Int with s.indices.contains(x)}]) = s.zipWithIndex.filter((_, i) => axes.contains(i)).map(_._1)
```

## Tensor: Set Notation

```scala
trait Tensor[T]:
  val shape: Shape
  def sameShape(t: Tensor[T]): Boolean = ???
  def add(t: Tensor[T] with t.sameShape(this)):
    {res: Tensor[T] with res.sameShape(this)}

  def mean(axes: List[{x: Int with shape.indices.contains(x)}]):
    {res: Tensor[T] with res.shape == shape.reduce(axes)}

  def reshape(newShape: Shape with newShape.nbrElems == shape.nbrElems):
    {res: Tensor[T] with res.shape == newShape}
```

## `this` doesn't work

```scala
trait Tensor[T]:
  def add(t: Tensor[T] with t.sameShape(Tensor.this)):
    Tensor[T] with this.sameShape(Tensor.this)
```

## Overloading Resolution

```scala
def f(x: Any) = "f1"
def f(x: Pos) = "f2"

f(-1)
```

. . .

During typing: `Pos =:= Int`, `f1` gets chosen

. . .

During QualChecking: `Pos < Int`, and type mismatch

. . .

But `f2` would have been fine !


## Irrefutability

```scala
type Pos = Int with x > 0

object Extractor:
  def unapply(x: Pos): Some[Pos] = Some(x)

-1 match
  case Extractor(x) => x
  case _ => None // warning: unreachable case
```

## `it` and de Bruijn indices

$\lambda f.\:\lambda g.\:\lambda x.\:f (g x)$

$\lambda\:\lambda\:\lambda\:3\:(2\:1)$

```scala
... with ...
  ... with ...
    ... with ...
      super.super.it( super.it( it ) )
```

. . .

But please don't do that
