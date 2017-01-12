========================================
    Nim - Goals, history, macros
========================================



Goals
=====

"The reasonable man adapts himself to the world: the unreasonable one persists
in trying to adapt the world to himself. Therefore all progress depends on
the unreasonable man." -- George Bernard Shaw

..
  I wanted a programming language that is

* as fast as C
* as expressive as Python
* as extensible as Lisp



The history of Nim
==================

* Development started in 2006.
* First bootstrapping succeeded in 2008.
  - Compiler written in Pascal.
  - Translated by a tool (pas2nim) to Nim
  - used pas2nim to produce the first wrappers

The history of Nim
==================

* Development started in 2006.
* First bootstrapping succeeded in 2008.
  - Compiler written in Pascal.
  - Translated by a tool (pas2nim) to Nim
  - used pas2nim to produce the first wrappers

* Goals:
  - **leverage meta programming to keep the language small**
  - compile to C
  - implementation size: 20_000 lines of code
  - learn from C's, C++'s, Ada's mistakes


The history of Nim (2)
======================

  "Good Software Takes Ten Years. Get Used To it." -- Joel Spolsky

- current implementation size: about 90_000 lines of code
- language is actually pretty big: generics, concepts, exceptions, procs,
  templates, macros, methods, inheritance, pointers, effect system, ...

..
  Language influences
  ===================

  The language borrows heavily from:

  - Modula 3:
    * traced vs untraced pointers

  - Delphi
    * type safe bit sets (``set of char``)
    * the parts of the syntax people don't like

  - Ada
    * subrange types
    * distinct type
    * safe variants / case objects

  - C++
    * Excessive overloading
    * Generic programming


  Language influences
  ===================

  - Python
    * indentation based syntax
    * programming should be fun
    * the parts of the syntax people do like

  - Lisp
    * we really want a macro system
    * embrace the AST
    * homoiconicity; everything is a function application
      (well, in Nim's case ... not really)

  - Oberon
    * the export marker

  - C#
    * async / await
    * lambda macros


Implementation aspects
======================

- Nim compiles to C, C++, Objective-C and JavaScript
- thread local soft realtime GC
- whole program dead code elimination: stdlib carefully crafted to make use of it;
  for instance parsers do not use (runtime) regular expression -> re engine not part of the executable
- the GC is also optimized away if you do not use it


Hello world
===========

.. code-block:: nim

  echo "hello ", "world", 99


Hello world
===========

.. code-block:: nim

  echo "hello ", "world", 99

is rewritten to:

.. code-block:: nim

  echo([$"hello ", $"world", $99])

- echo is declared as: ``proc echo(a: varargs[string, `$`]);``
- ``$`` (Nim's ``toString`` operator) is applied to every argument
- local type converter: only in this context everything is converted to a string via ``$``
- in contrast to C#/Java's solution this requires no dynamic binding


Hello world
===========

.. code-block:: nim

  echo "hello ", "world", 99

is rewritten to:

.. code-block:: nim

  echo([$"hello ", $"world", $99])

- echo is declared as: ``proc echo(a: varargs[string, `$`]);``
- ``$`` (Nim's ``toString`` operator) is applied to every argument
- local type converter: only in this context everything is converted to a string via ``$``
- in contrast to C#/Java's solution this requires no dynamic binding
- it is extensible:

.. code-block:: nim

  proc `$`(x: MyObject): string = x.s
  var obj = MyObject(s: "xyz")
  echo obj  # works


Meta programming features
=========================

Nim's focus is meta programming; macros are used

1. to avoid code duplication / boilerplate:

.. code-block:: nim

  template htmlTag(tag: untyped) =
    proc tag(): string = "<" & astToStr(tag) & ">"

  htmlTag(br)
  htmlTag(html)

  echo br()

Produces::

  <br>



Meta programming features
=========================

2. for control flow abstraction:

.. code-block:: nim
    :number-lines:

  template once(body) =
    var guard {.global.} = false
    if not guard:
      guard = true
      body

  proc p() =
    once:
      echo "first call of p"
    echo "some call of p"

  p()
  once:
    echo "new instantiation"
  p()


Meta programming features
=========================

2. for control flow abstraction:

.. code-block:: nim
    :number-lines:

  template once(body) =
    var guard {.global.} = false
    if not guard:
      guard = true
      body

  proc p() =
    once:
      echo "first call of p"
    echo "some call of p"

  p()
  once:
    echo "new instantiation"
  p()


Produces::

  first call of p
  some call of p
  new instantiation
  some call of p



Meta programming features
=========================


3. for lazy evaluation:

.. code-block:: nim
    :number-lines:

  template log(msg: string) =
    if debug:
      echo msg

  log("x: " & $x & ", y: " & $y)



Meta programming features
=========================

4. to implement DSLs:


.. code-block:: nim
    :number-lines:

  html mainPage:
    head:
      title "now look at this"
    body:
      ul:
        li "Nim is quite capable"

  echo mainPage()

Produces::

  <html>
    <head><title>now look at this</title></head>
    <body>
      <ul>
        <li>Nim is quite capable</li>
      </ul>
    </body>
  </html>



Meta programming features
=========================

Implementation:

.. code-block:: nim
    :number-lines:

  template html(name, body) =
    proc name(): string =
      result = "<html>"
      body
      result.add("</html>")

  template head(body) =
    result.add("<body>")
    body
    result.add("</body>")

  template title(x: string) =
    result.add("<title>$1</title>" % x)

  ...



Meta programming features
=========================


.. code-block:: nim
    :number-lines:

  html mainPage:
    head:
      title "now look at this"
    body:
      ul:
        li "Nim is quite capable"

  echo mainPage()

Is translated into:

.. code-block:: nim
    :number-lines:

  proc mainPage(): string =
    result = "<html>"
    result.add("<head>")
    result.add("<title>$1</title>" % "now look at this")
    result.add("</head>")
    result.add("<body>")
    result.add("<ul>")
    result.add("<li>$1</li>" % "Nim is quite capable")
    result.add("</ul>")
    result.add("</body>")
    result.add("</html>")


Operators
=========

.. code-block::nim
   :number-lines:

  template `??`(a, b: untyped): untyped =
    let x = a
    (if x.isNil: b else: x)

  var x: string
  echo x ?? "woohoo"


Lifting
=======

.. code-block:: nim
   :number-lines:

  import math

  template liftScalarProc(fname) =
    ## Lift a proc taking one scalar parameter and returning a
    ## scalar value (eg ``proc sssss[T](x: T): float``),
    ## to provide templated procs that can handle a single
    ## parameter of seq[T] or nested seq[seq[]] or the same type
    proc fname[T](x: openarray[T]): auto =
      result = newSeq[type(x[0])](x.len)
      for i in 0..<x.len:
        result[i] = fname(x[i])

  liftScalarProc(sqrt)   # make sqrt() work for sequences
  echo sqrt(@[4.0, 16.0, 25.0, 36.0])   # => @[2.0, 4.0, 5.0, 6.0]


Improving APIs
==============

.. code-block:: nim
   :number-lines:

  proc drawLine(context: Context; color: Color; p1, p2: Point) = ...

  proc drawRect(c: Context; col: Color; p1, p2: Point) =
    drawLine c, col, p1, Point(...)
    drawLine c, col, p1, Point(...)
    drawLine c, col, p2, Point(...)
    drawLine c, col, p2, Point(...)



Improving APIs
==============

.. code-block:: nim
   :number-lines:

  var drawColor {.threadvar.}: Color

  proc setDrawColor*(context: Context; col: Color) =
    drawColor = col

  proc drawLine(context: Context; p1, p2: Point) = ...

  proc drawRect(c: Context; p1, p2: Point) =
    drawLine c, p1, Point(...)
    drawLine c, p1, Point(...)
    drawLine c, p2, Point(...)
    drawLine c, p2, Point(...)




Improving APIs
==============

.. code-block:: nim
   :number-lines:

  proc drawLine(context: Context; color: Color; p1, p2: Point) = ...

  template drawLine(p1, p2: Point) =
    drawLine(context, color, p1, p2)

  template drawRect(p1, p2: Point) =
    drawLine p1, Point(...)
    drawLine p1, Point(...)
    drawLine p2, Point(...)
    drawLine p2, Point(...)



Improving APIs
==============

.. code-block:: nim
   :number-lines:

  proc main =
    const color = White
    var context = newContext()
    drawRect Rect(...)
    drawRect Rect(...)


Declarative programming
=======================

.. code-block:: nim
   :number-lines:

  proc threadTests(r: var Results, cat: Category, options: string) =
    template test(filename: untyped) =
      testSpec r, makeTest("tests/threads" / filename, options,
        cat, actionRun)
      testSpec r, makeTest("tests/threads" / filename, options &
        " -d:release", cat, actionRun)
      testSpec r, makeTest("tests/threads" / filename, options &
        " --tlsEmulation:on", cat, actionRun)

    test "tactors"
    test "tactors2"
    test "threadex"


Varargs
=======

.. code-block:: nim

  test "tactors"
  test "tactors2"
  test "threadex"

-->

.. code-block:: nim

   test "tactors", "tactors2", "threadex"


Varargs
=======

.. code-block:: nim
   :number-lines:
  import macros

  macro va(caller: untyped; args: varargs[untyped]): untyped =
    result = newStmtList()
    for a in args:
      result.add(newCall(caller, a))

  va test, "tactors", "tactors2", "threadex"


Assert
======

.. code-block::nim
   :number-lines:

  proc main =
    var mystr = "Would you"
    mystr.add " kindly?"
    assert mystr == "Would you kindly", "bug!"
    echo mystr

  main()



Assert
======

.. code-block::nim
   :number-lines:

  template assert(cond, msg: untyped) =
    if not cond:
      quit msg


Assert
======


Desired output::

  Expected: Would you kindly
  But got: Would you kindly?


Assert
======

.. code-block::nim
   :number-lines:

  import macros

  macro assert(cond, msg: untyped): untyped =
    let body = newCall(bindSym"quit", msg)
    result = newNimNode(nnkIfStmt)
    result.add(newNimNode(nnkElifBranch).add(
      newCall(bindSym"not", cond), body))
    echo treeRepr result


Trees
=====

AST::

  IfStmt
    ElifBranch
      Call
        Sym "not"
        Infix
          Ident !"=="
          Ident !"mystr"
          StrLit Hello World
      Call
        ClosedSymChoice
          Sym "quit"
          Sym "quit"
        StrLit bug!

Assert
======


.. code-block::nim
   :number-lines:

  import macros

  macro assert(cond, msg: untyped): untyped =
    template helper(cond, msg) =
      if not cond:
        quit msg
    result = getAst(helper(cond, msg))


Assert
======

.. code-block:: nim

  import macros

  macro assert(cond, msg: untyped): untyped =
    template fallback(cond, msg) =
      if not cond:
        quit msg

    template cmp(cond, a, b, msg) =
      if not cond:
        echo "Expected: ", b
        echo "But got: ", a
        quit msg

    if cond.kind in nnkCallKinds and cond.len == 3 and
        $cond[0] in ["==", "<=", "<", ">=", ">", "!="]:
      result = getAst(cmp(cond, cond[1], cond[2], msg))
    else:
      result = getAst(fallback(cond, msg))


Typesafe Writeln/Printf
=======================

.. code-block::nim
   :number-lines:

  proc write(f: File; a: int) =
    echo a

  proc write(f: File; a: bool) =
    echo a

  proc write(f: File; a: float) =
    echo a

  proc writeNewline(f: File) =
    echo "\n"

  macro writeln*(f: File; args: varargs[typed]) =
    result = newStmtList()
    for a in args:
      result.add newCall(bindSym"write", f, a)
    result.add newCall(bindSym"writeNewline", f)


Quoting
=======

.. code-block::nim
   :number-lines:

  import macros

  macro quoteWords(names: varargs[untyped]): untyped =
    result = newNimNode(nnkBracket)
    for i in 0..names.len-1:
      expectKind(names[i], nnkIdent)
      result.add(toStrLit(names[i]))

  const
    myWordList = quoteWords(here, an, example)

  for w in items(myWordList):
    echo w



Currying
========

.. code-block::nim
   :number-lines:

  proc f(a, b, c: int): int = a+b+c

  echo curry(f, 10)(3, 4)


Currying
========

.. code-block::nim
   :number-lines:

  proc f(a, b, c: int): int = a+b+c

  echo((proc (b, c: int): int = f(10, b, c))(3, 4))


Currying
========

.. code-block::nim
   :number-lines:

  macro curry(f: typed; args: varargs[untyped]): untyped =
    let ty = getType(f)
    assert($ty[0] == "proc", "first param is not a function")
    let n_remaining = ty.len - 2 - args.len
    assert n_remaining > 0, "cannot curry all the parameters"
    var callExpr = newCall(f)
    args.copyChildrenTo callExpr

    var params: seq[NimNode] = @[]
    # return type
    params.add ty[1]

    for i in 0..<n_remaining:
      let param = ident("arg" & $i)
      params.add newIdentDefs(param, ty[i+2+args.len])
      callExpr.add param
    result = newProc(procType = nnkLambda, params = params, body = callExpr)




Parallelism
===========

.. code-block::nim
   :number-lines:

  import tables, strutils

  proc countWords(filename: string): CountTableRef[string] =
    ## Counts all the words in the file.
    result = newCountTable[string]()
    for word in readFile(filename).split:
      result.inc word


Parallelism
===========

.. code-block::nim
   :number-lines:

  #
  #
  const
    files = ["data1.txt", "data2.txt", "data3.txt", "data4.txt"]

  proc main() =
    var tab = newCountTable[string]()
    for f in files:
      let tab2 = countWords(f)
      tab.merge(tab2)
    tab.sort()
    echo tab.largest

  main()


Parallelism
===========

.. code-block::nim
   :number-lines:

  import threadpool

  const
    files = ["data1.txt", "data2.txt", "data3.txt", "data4.txt"]

  proc main() =
    var tab = newCountTable[string]()
    var results: array[files.len, ***FlowVar[CountTableRef[string]]***]
    for i, f in files:
      results[i] = ***spawn*** countWords(f)
    for i in 0..high(results):
      tab.merge(*** ^results[i] ***)
    tab.sort()
    echo tab.largest

  main()


Parallelism
===========

.. code-block::nim
   :number-lines:

  import strutils, math, threadpool

  proc term(k: float): float = 4 * math.pow(-1, k) / (2*k + 1)

  proc computePi(n: int): float =
    var ch = newSeq[FlowVar[float]](n+1)
    for k in 0..n:
      ch[k] = spawn term(float(k))
    for k in 0..n:
      result += ^ch[k]


Happy hacking!
==============

============       ================================================
Website            http://nim-lang.org
Mailing list       http://www.freelists.org/list/nim-dev
Forum              http://forum.nim-lang.org
Github             https://github.com/nim-lang/Nim
IRC                irc.freenode.net/nim
============       ================================================

