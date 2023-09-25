---
title: Syntax and Runtime Checks for Qualified Types in Scala 3
author: "[Quentin Bernet](mailto:quentin.bernet@epfl.ch) @[LARA](https://lara.epfl.ch/w/), [EPFL](https://www.epfl.ch/fr/)"
---

# Introduction
<!-- 
Max ~25 slides
1 min by slide
Slides to the point
commit html
can make github page
 -->

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

## What are _qualified_ types ?

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

# Markdown

- Item 1
- Item 2

1. Item 1
2. Item 2

_Emphasis_, __Strong__

# Syntax highlighting

```scala
val a = if true then 0 else 42
transparent inline def hello() = ???
case class Foo(val bar: Int)
extension (o: Object) def baz() = ???
```

(Supports Scala 3 syntax!)

# LaTeX

$$\phi = \frac{a}{b}$$

# Slide break

Can break to the next slide without a new heading using a Markdown section beak:

```
---
```

---

Second part.

# Columns

::: columns

:::: column
Column 1 Content
::::

:::: column
Column 2 Content
::::

:::

# Fragments

::: fragment

First thing

:::

::: fragment

Second thing

:::
