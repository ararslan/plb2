**TL;DR**: see the figure below. Note that nqueen and matmul are implemented in
all languages but sudoku and bedcov are only implemented in some.

<img align="left" src="https://lh3.sourceforge.io/images/plb2-m1.png?v14"/>

## Table of Contents

- [Introduction](#intro)
- [Results](#result)
  - [Overall impression](#overall)
  - [Caveats](#caveat)
    - [Startup time](#startup)
	- [Elapsed time vs CPU time](#cputime)
  - [Subtle optimizations](#opt)
	- [Optimizing inner loops](#matmul)
	- [Controlling memory layout](#memlayout)
- [Discussions](#conclusion)
- [Appendix: Timing on Apple M1 Macbook Pro](#table)

## <a name="intro"></a>Introduction

Programming Language Benchmark v2 (plb2) evaluates the performance of 24
programming languages on four CPU-intensive tasks. It is a follow-up to
[plb][plb] conducted in 2011. In plb2, all implementations use the same
algorithm for each task and their performance bottlenecks do not fall in
library functions. We do not intend to compare different algorithms or the
quality of the standard libraries in these languages. Plb2 aims to
evaluate the performance of a language when you have to implement a new
algorithm in the language - this may happen if you can't find the algorithm in
existing libraries.

The four tasks in plb2 all take a few seconds for a fast implementation to
complete. The tasks are:

* **nqueen**: solving a [15-queens problem][8queen]. The algorithm was inspired
  by the second C implementation [from Rosetta Code][8qrc]. It involves nested
  loops and integer bit operations.

* **matmul**: multiplying two square matrices of 1500x1500 in size. The inner
  loop resembles BLAS' [axpy][axpy] operation.

* **sudoku**: solving 4000 hard [Sudokus][sudoku] (20 puzzles repeated for 200
  times) using the [kudoku algorithm][kudoku]. This algorithm heavily uses
  small fixed-sized arrays with a bit complex logic.

* **bedcov**: finding the overlaps between two arrays of 1,000,000 intervals
  with [implicit interval trees][iitree]. The algorithm involves frequent
  array access in a pattern similar to binary searches.

Every language has nqueen and matmul implementations. Some languages do not
have sudoku or bedcov implementations. Most programs were initially implemented
by me and a few were contributed by others. As I am mostly a C programmer,
implementations in other languages may be suboptimal and there are no
implementations in functional languages. **Pull requests are welcomed!**

## <a name="result"></a>Results

The figure at the top of the page summarizes the elapsed time of each implementation
measured on an Apple M1 MacBook Pro. [Hyperfine][hyperfine] was used for timing
except for a few slow implementations which were timed with the "time" bash
command without repetition. A plus sign "+" indicates [ahead-of-time
compilation][aot] (AOT). Exact timing can be found in the [table below](#table). The
figure was [programmatically generated](analysis) from the table.

### <a name="overall"></a>Overall impression

Programming language implementations in plb2 can be classified into three groups
depending on how and when compilation is done:

1. Purely interpreted (QuickJS, Perl and [CPython][cpy], the official Python
   implementation). Not surprisingly, these are among the slowest language
   implementations in this benchmark.

2. JIT compiled (Dart, Bun/Node, Java, Julia, LuaJIT, PHP, PyPy and Ruby3 with
   [YJIT][yjit]). They are generally faster than pure interpretation.
   Nonetheless, there is a large variance in this group. While PHP and Ruby3
   are faster than Perl and CPython, they are still an order of magnitude
   slower than PyPy. The two JavaScript engines (Bun and Node) and Julia
   perform well. They are about twice as fast as PyPy.

3. AOT compiled (the rest). Optimizing binaries for specific hardware, these
   compilers tend to generate the fastest executables.

### <a name="caveat"></a>Caveats

#### <a name="startup"></a>Startup time

Some JIT-based language runtimes take up to ~0.3 second to compile and warm-up.
We are not separating out this startup time. Nonetheless, because most
benchmarks run for several seconds, including the startup time does not greatly
affect the results.

#### <a name="cputime"></a>Elapsed time vs CPU time

Although no implementations use multithreading, language runtimes may be doing
extra work, such as garbage collection, in a separate thread. In this case, the
CPU time (user plus system) may be longer than elapsed wall-clock time. Julia,
in particular, takes noticeably more CPU time than wall-clock time even for the
simplest nqueen benchmark. In plb2, we are measuring the elapsed wall-clock
time because that is the number users often see. The ranking of CPU time may be
slightly different.

### <a name="opt"></a>Subtle optimizations

#### <a name="memlayout"></a>Controlling memory layout

When implementing bedcov in Julia, C and many compiled languages, it is
preferred to have an array of objects in a contiguous memory block such that
adjacent objects are close in memory. This helps cache efficiency. In most
scripting languages, unfortunately, we have to put references to objects in an
array at the cost of cache locality. The issue can be alleviated by cloning
objects to a new array. This doubles the speed of PyPy and Bun.

#### <a name="matmul"></a>Optimizing inner loops

The bottleneck of matrix multiplication falls in the following nested loop:
```cpp
for (int i = 0; i < n; ++i)
    for (int k = 0; k < n; ++k)
        for (int j = 0; j < n; ++j)
            c[i][j] += a[i][k] * b[k][j];
```
It is obvious that `c[i]`, `b[k]` and `a[i][k]` can be moved out of the inner
loop to reduce the frequency of matrix access. The Clang compiler can apply
this optimization. Manual optimization may actually hurt performance.

However, **many other languages cannot optimize this nested loop.** If we
manually move `a[i][k]` to the loop above it, we can often improve their
performance. Some C/C++ programmers say compilers often optimize better than
human, but this might not be the case in other languages.

## <a name="conclusion"></a>Discussions

The most well-known and the longest running language benchmark is the [Computer
Language Benchmark Games][clbg]. Plb2 differs in that it includes different
languages (e.g. Nim and Crystal), different language runtimes (e.g. PyPy and
LuaJIT) and new tasks, and it comes with more uniform
implementations and focuses more on the performance of the language itself
without library functions. **Plb2 complements the Computer Language Benchmark
Games.**

One important area that plb2 does not evaluate is the performance of memory
allocation and/or garbage collection. This may contribute more to practical
performance than generating machine code. Nonetheless, it is challenging to
design a realistic micro-benchmark to evaluate memory allocation. If the
built-in allocator in a language implementation does not work well, we can
implement customized memory allocator just for the specific task but this, in
my view, would not represent typical use cases.

When plb was conducted in 2011, half of the languages in the figure above were
not mature or even did not exist. It is exciting to see many of them have
reached the 1.0 milestone and are gaining popularity among modern programmers.
On the other hand, Python remains one of the two most used scripting languages
despite its poor performance. In my view, this is because PyPy would not be
officially endorsed while other JIT-based languages are not general or good
enough. Will there be a language to displace Python in the next decade? I am
not optimistic.

## <a name="table"></a>Appendix: Timing on Apple M1 Macbook Pro

In the following table, star "\*" indicates AOT compilation and plus "+"
indicates JIT compilation.

|Label    |Language  |Runtime|Version| nqueen | matmul | sudoku | bedcov |
|:--------|:---------|:------|:------|-------:|-------:|-------:|-------:|
|c:clang* |C         |Clang  |15.0.0 | 2.57   | 0.54   | 1.56   | 0.84   |
|cl:sbcl* |Lisp      |SBCL   |2.4.0  | 3.19   | 3.84   |        |        |
|crystal* |Crystal   |       |1.10.0 | 3.28   | 2.45   | 3.14   | 0.87   |
|c#:.net* |C#        |.NET   |8.0.100| 2.82   | 1.38   | 3.12   |        |
|d:ldc2*  |D         |LDC2   |2.105.2| 2.68   | 0.57   | 1.60   |        |
|dart:jit+|Dart      |(JIT)  |3.2.4  | 3.62   | 4.81   | 3.24   |        |
|elixir+  |Elixir    |       |1.15.7 | 26.17  | 67.39  |        |        |
|f90:gcc* |Fortran90 |GCC    |13.2.0 | 2.67   | 0.51   | 1.84   |        |
|go*      |Go        |       |1.21.5 | 2.94   | 1.63   | 2.04   |        |
|java+    |Java      |OpenJDK|20.0.1 | 3.92   | 1.14   | 3.20   |        |
|js:bun+  |JavaScript|Bun    |1.0.20 | 3.11   | 1.75   | 3.07   | 2.83   |
|js:deno+ |JavaScript|Deno   |1.39.1 | 4.00   | 3.06   | 4.04   | 3.87   |
|js:k8+   |JavaScript|k8     |1.0    | 3.79   | 2.99   | 3.76   | 4.02   |
|js:node+ |JavaScript|Node   |21.5.0 | 3.73   | 2.88   | 3.77   | 3.83   |
|js:node  |JavaScript|Node-nojit|21.5.0|55.48 | 162.84 | 63.91  | 21.61  |
|js:qjs   |JavaScript|QuickJS|23-12-09|59.04  | 135.66 | 67.55  | 38.27  |
|julia+   |Julia     |       |1.10.0 | 3.02   | 0.76   | 2.35   | 1.96   |
|luajit+  |Lua       |LuaJIT |2.1    | 5.31   | 2.66   | 4.48   | 10.59  |
|mojo*    |Mojo      |       |0.6.1  | 3.24   | 1.12   |        |        |
|nim*     |Nim       |       |2.0.2  | 2.57   | 0.56   | 1.64   | 1.07   |
|ocaml*   |OCaml     |       |4.14.1 | 3.56   | 2.14   |        |        |
|perl     |Perl      |       |5.34.1 | 158.34 | 158.01 | 90.78  |        |
|php+     |PHP       |       |8.3    | 48.15  | 71.20  |        |        |
|py:codon\*|Python   |Codon  |0.16.3 | 2.91   |        | 3.10   |        |
|py:cpy   |Python    |CPython|3.11.7 | 159.97 | 108.21 | 52.88  | 42.84  |
|py:pypy+ |Python    |Pypy   |7.3.14 | 6.91   | 4.95   | 8.82   | 6.27   |
|ruby+    |Ruby      |(YJIT) |3.3.0  | 88.15  | 130.51 | 52.26  |        |
|rust*    |Rust      |       |1.75.0 | 2.49   | 0.56   | 1.65   | 0.94   |
|swift*   |Swift     |       |5.9.0  | 2.92   | 0.56   | 1.78   |        |
|v*       |V         |       |0.4.3  | 2.55   | 0.57   | 1.66   | 1.23   |
|zig*     |Zig       |       |0.11.0 | 2.72   | 0.56   |        |        |

[plb]: https://github.com/attractivechaos/plb
[8queen]: https://en.wikipedia.org/wiki/Eight_queens_puzzle
[8qrc]: https://rosettacode.org/wiki/N-queens_problem#C
[sudoku]: https://en.wikipedia.org/wiki/Sudoku
[kudoku]: https://attractivechaos.github.io/plb/kudoku.html
[iitree]: https://academic.oup.com/bioinformatics/article/37/9/1315/5910546
[hyperfine]: https://github.com/sharkdp/hyperfine
[cpy]: https://en.wikipedia.org/wiki/CPython
[pypy]: https://www.pypy.org
[bun]: https://bun.sh
[luablog]: https://attractivechaos.wordpress.com/2011/01/23/amazed-by-luajit/
[yjit]: https://github.com/ruby/ruby/blob/master/doc/yjit/yjit.md
[aot]: https://en.wikipedia.org/wiki/Ahead-of-time_compilation
[clbg]: https://benchmarksgame-team.pages.debian.net/benchmarksgame/index.html
[axpy]: https://en.wikipedia.org/wiki/Basic_Linear_Algebra_Subprograms#Level_1
