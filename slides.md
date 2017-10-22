layout: true
name: title-slide
class: middle, title-slide

---
# Genomic Analysis in Scala

<br/>

SPLASH 2017

<br/>

October 22, 2017

Ryan Williams

---
layout: true
name: main-slides
class: main-slide

---
# Hammer Lab

- Mt. Sinai School of Medicine, NYC
- Personal Genome Vaccine pipeline / clinical trial
- Checkpoint blockade biomarkers, mutational signatures
- Genome biofx using Spark + Scala
- Biofx workflows and tools in OCaml
- http://www.hammerlab.org/


.img-container.top-pad-2em[
![](img/lab-photo.jpg)
]

---
# Overview
- Applications
- Libraries
- Design-Pattern Deep-Dive

---
## coverage-depth


---
## spark-bam


---
layout: false
class: divider-slide, middle
# Libraries

---
layout: true
class: main-slide
---
## [magic-rdds](https://github.com/hammerlab/magic-rdds)
Collection-operations implemented for Spark RDDs

--
- scans

--
  - {left,right}

--
  - {elements, values of tuples}

--
- `.runLengthEncode`

--
- `.reverse`

--
- `.maxByKey`, `.minByKey`

--
- sliding/windowed traversals

--
- `.size` - smart `count`

--
- zips

--
  - lazy partition-count

--
  - eager partition-number check)

--
- `sameElements`, `equals`

--
- group/sample by key: first elems or reservoir-sampled

---
## [iterators](https://github.com/hammerlab/iterators)

--
- scans

--
- sliding/windowed traversals

--
- eager drops/takes

--
  - by number

--
  - while

--
  - until

--
- sorted/range zips

--
- `SimpleBufferedIterator`

--
  - iterator in terms of `_advance(): Option[T]`

--
  - `hasNext` lazily buffers/caches `head`

--
- etc.

---
layout: false
class: divider-slide, middle
# Design Patterns

---
layout: true
template: main-slides

---
class: line-height-code-11, pad-h2-bottom
## [shapeless-utils](https://github.com/hammerlab/shapeless-utils)

--
### "recursive structural types"

--

.left-code-col.code-col[
Deep case-class hierarchy:

```scala
case class A(n: Int)
case class B(s: String)
case class C(a: A, b: B)
case class D(b: Boolean)
case class E(c: C, d: D, a: A, a2: A)
case class F(e: E)
```
]

--
.right-code-col.code-col[
Instances:

```scala
val a = A(123)
val b = B("abc")
val c = C(a, b)
val d = D(true)
val e = E(c, d, A(456), A(789))
val f = F(e)
```
]

--
.left-code-col.code-col[
Pull out fields by type and/or name:

```scala
f.find('c)      // f.e.c
f.findT[C]      // f.e.c
f.field[C]('c)  // f.e.c

f.field[A]('a2) // f.e.a2
f.field[B]('b)  // f.e.c.b
```
]

--
.right-code-col.code-col[
As evidence parameters:

```scala
def findAandB[T](t: T)(
  implicit
  findA: Find[T, A],
  findB: Find[T, B]
): (A, B) =
  (findA(t), findB(t))
```
]

---
class: line-height-code-11, pad-h2-bottom, slide-padding-3em
## Command-line interfaces

--
.left-code-col.code-col[
### [args4j](http://args4j.kohsuke.org/)

```
class Opts {
  @args4j.Option(
    name = "--in-path",
    aliases = Array("-i"),
    handler = classOf[PathOptionHandler],
    usage = "Input path to read from"
  )
  var inPath: Option[Path] = None

  @args4j.Option(
    name = "--out-path",
    aliases = Array("-o"),
    handler = classOf[PathOptionHandler],
    usage = "Output path to write to"
  )
  var outPath: Option[Path] = None

  @args4j.Option(
    name = "--overwrite",
    aliases = Array("-f"),
    usage = "Whether to overwrite an existing output file"
  )
  var overwrite: Boolean = false
}
```
]

--
.right-code-col.code-col[
### [case-app](https://github.com/alexarchambault/case-app)

```
case class Opts(
  @Opt("-i")
  @Msg("Input path to read from")
  inPath: Option[Path] = None,

  @Opt("-o")
  @Msg("Output path to write to")
  outPath: Option[Path] = None,

  @Opt("-f")
  @Msg("Whether to overwrite an existing output file")
  overwrite: Boolean = false
)
```

]

--
.right-code-col.code-col[
- statically-checked/typed handlers
]

--
.right-code-col.col[
- implicit resolution
]

--
.right-code-col.col[
- nesting: inheritance vs. composition

]

--
.right-code-col.col[
- mutable vs. immutable

]

--
.right-code-col.col[
- case-app positional-arg support: [#58](https://github.com/alexarchambault/case-app/issues/58)

]

--
.right-code-col.col[
- [hammerlab/spark-commands](https://github.com/hammerlab/spark-commands/)

]

---
## Argument-Passing

--
[Seen in the wild](https://github.com/hammerlab/guacamole/blob/9d330aeb3a7a040c174b851511f19b42d7717508/src/main/scala/org/hammerlab/guacamole/commands/GermlineAssemblyCaller.scala#L63-L75):

--
minAreaVaf: minAreaVaf = args.minAreaVaf / 100.0f
discoverGermlineVariants: discoverGermlineVariants
```
val calledAlleles =
  {{discoverGermlineVariants}}(
    args.sampleName,
    kmerSize = args.kmerSize,
    assemblyWindowRange = args.assemblyWindowRange,
    minOccurrence = args.minOccurrence,
    {{minAreaVaf}},
    reference = reference,
    minMeanKmerQuality = args.minMeanKmerQuality,
    minPhredScaledLikelihood = args.minLikelihood,
    shortcutAssembly = args.shortcutAssembly
  )
```
--
Just two (trivial) things happening here:

--
minAreaVaf: `minAreaVaf = args.minAreaVaf / 100.0f`
1. convert a percentage to a `Double`

--
minAreaVaf: minAreaVaf = args.minAreaVaf / 100.0f
discoverGermlineVariants: `discoverGermlineVariants`
2. call .highlight-inline-code[`discoverGermlineVariants`] with (10!) obvious arguments


--
Shorthand for passing variables as identically-named parameters would be nice

--
- can kind of do this with `kwargs` in Python, structuring/destructuring in JavaScript


--
Even better: use implicits to do "obvious" arg-passing like this

--
- variable names ⟶ types


--
  ⟹ only one instance of a given type in scope

--
- what to do with var names? `_1`, `_2`, `_3`, …? 😭

---
name: mixing-implicits
class: line-height-code-11, pad-h2-bottom
## Nesting/Mixing implicit contexts

--
**Minimal boilerplate Spark CLI apps:**

--
- input `Path`

--
psMsg: 
- output `Path` {{psMsg}}

--
psMsg:  or just a `PrintStream`

--
- `SparkContext`

--
- select `Broadcast` variables

--
- other argument-input objects


--
.half-top-pad[
**How to make all of these things `implicit`ly available with minimal boilerplate?**
]

--
.l.code-col-40[
```
def run(
  implicit
  inPath: Path,
  printStream: PrintStream,
  sc: SparkContext,
  ranges: Broadcast[Ranges],
  …
): Unit = {
  // do thing
}
```
]

--
.r.code-col-56[
```
case class Context(
  inPath: Path,
  printStream: PrintStream,
  sc: SparkContext,
  ranges: Broadcast[Ranges],
  …
)
```
]

--
.r.code-col-56[
```
def run(`implicit ctx: Context`): Unit = {
  implicit val Context(
    `inPath, printStream, sc, ranges, …`
  ) = ctx
  // do thing
}
```
]

---
template: mixing-implicits
answer: 
How to make many `implicit`s available with minimal boilerplate? {{answer}}

--
answer: Inheritance…

--
```
trait HasSparkContext {
  implicit val sc: SparkContext = new SparkContext(…)
}
```
--
```
abstract class HasArgs(args: Array[String])
```
--
.l.col[
```
trait HasInputPath { self: Args ⇒
  implicit val inPath = args(0)
}
```
]
--
.r.col[
```
trait HasOutputPath { self: Args ⇒
  implicit val outPath = args(1)
}
```
]
--
.clear-both.tenth-top-pad[
```
trait HasPrintStream extends HasOutputPath { self: Args ⇒
  implicit val printStream = new PrintStream(newOutputStream(outPath))
}
```
]

--
.l.col[
```
class MinimalApp(args: Array[String])
  extends HasArgs(args) 
  with HasInputPath 
  with HasPrintStream 
  with HasSparkContext
```
]

--
.r.col[
```
object Main {
  def main(args: Array[String]): Unit = 
    new MinimalApp(args) {
      // all the implicits!
    }
  }
}
```
]

---
class: line-height-code-11, pad-h2-bottom
## `{to,from}String`: invertible syntax

--
Miscellaneous tools output "reports":

```text
    466202931615 uncompressed positions
    156G compressed
    Compression ratio: 2.78
    1236499892 reads
    22489 false positives, 0 false negatives
```

--
.left-code-col.col.half-top-pad[
That comes from a data structure like:

```
case class Result(
  numPositions     : Long,
  compressedSize   : Bytes,
  compressionRatio : Double,
  numReads         : Long,
  numFalsePositives: Long,
  numFalseNegatives: Long
)
```
]

--
.right-code-col.col.half-top-pad[
or better yet:

```
case class Result(
  numPositions    : `NumPositions`,
  compressedSize  : `CompressedSize`,
  compressionRatio: `CompressionRatio`,
  numReads        : `NumReads`,
  `falseCounts     : FalseCounts`
)
```
]

--
toStringMsg: This is basically `toString`
.clear-both[
- {{toStringMsg}}
]

--
toStringMsg: This is ~~basically `toString`~~ the `Show` type-class

--
- twist: downstream tools want to parse these reports

--
- want to re-hydrate `Result` instances


--
```
implicit val _iso: Iso[FalseCounts] =
    iso"${'numFPs} false positives, ${'numFNs} false negatives" }
```

---
## Dreams
- biofx filesystem layouts
- AST editing / style
- shading / linking

---
layout: false
class: divider-slide, middle
# Thanks!



