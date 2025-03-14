---
layout: post
title:  "Link Time Optimization (LTO), C++/D cross-language optimization"
tags: ["LDC"]
date: 2016-11-10
---

_A short article about Link Time Optimization (LTO) with LDC, with a simple example demonstrating how it can increase the performance of programs. Because LTO works at the LLVM IR level, it enables optimization across the C++/D language barrier!  
Important: LDC/LLVM's LTO is not available on Windows._

## Link Time Optimization

Link Time Optimization (LTO) refers to program optimization during linking. The linker pulls all object files together and combines them into one program. The linker can see the whole of the program, and can therefore do whole-program analysis and optimization. However, normally the linker only gets to see the program when it has already been translated into machine code. At that level, it should still be possible to do optimization but it is terribly hard. GCC's or LLVM's optimizers cannot be used.

The LTO mechanism of LLVM (same as GCC) is based on passing code (LLVM IR) that _can_ be understood by LLVM's optimizers to the linker, such that whole-program analysis and optimization can be performed during linking. So-called "full" LTO combines all LLVM IR code of the separate object files into one big LLVM module, and then optimizes that and generates machinecode as usual. "ThinLTO" keeps the modules separate, but imports functions as needed from other modules and does optimization and machinecodegen in parallel. Please read more about it in the LLVM Project Blog article ["ThinLTO: Scalable and Incremental LTO"](http://blog.llvm.org/2016/06/thinlto-scalable-and-incremental-lto.html) and in [Clang's ThinLTO documentation](http://clang.llvm.org/docs/ThinLTO.html).

All optimization improvements of full LTO can be had by compiling all your source at once, in one compilation invocation. All-at-once compilation is what `dub` does, and is also how (the D-part of) LDC itself is built at the moment.  
The advantage of doing LTO instead of all-at-once compilation is that the (part of) compilation is done in parallel with LTO. For full LTO (`-flto=full`), only the semantic analysis is done in parallel, but the optimization and machine codegen is done in a single thread. For ThinLTO (`-flto=thin`), all steps are done in parallel except for a global analysis step. ThinLTO is therefore much faster than full LTO or all-at-once compilation, especially if you have a machine with many cores available.

Yesterday, I merged my [pull request](https://github.com/ldc-developers/ldc/pull/1840) that adds LTO capability to LDC into the LDC master branch. So you can start playing with LTO after you've built LDC master (LLVM 3.9 or newer is needed).

To use LTO, probably all you need to do is specify `-flto=thin` or `-flto=full` on the commandline!

## Linker support

The way LTO works is that the object files output by the compiler are not regular object files: they are LLVM IR bitcode files disguised as object files simply by the objectfile file extension. This means that the linker has to support this LLVM LTO mechanism somehow.  
On Mac OS X, LLVM/Clang is used as the system compiler and the linker knows how to deal with LTO using the `libLTO.dylib` library. The LDC package for Mac OS X will ship with this library included, such that it is up-to-date with LDC's LLVM version.  
On Linux, the [gold linker](https://sourceware.org/binutils/) provides support for plugins, and the [LLVM gold plugin](http://llvm.org/docs/GoldPlugin.html) is used to deal with LTO. I am not sure if it is possible to ship this plugin with the LDC binary packages; perhaps different installations have incompatible plugin ABIs. Thus you may need to [build the plugin yourself](http://llvm.org/docs/GoldPlugin.html#lto-how-to-build) as part of building LLVM. You can then copy the plugin binary to LDC's `lib` folder, or pass `-flto-binary=<plugin file>` to LDC so that the linker can find it.

[LTO options](http://clang.llvm.org/docs/ThinLTO.html#usage) (such as ThinLTO caching for incremental builds) can be passed as usual linker options:  
OS X: `ldc2 -L-cache_path_lto -L/path/to/cache ...`  
gold: `ldc2 -L-plugin-opt=cache-dir=/path/to/cache ...`

## A simple example

Consider the following example, where code is spread across two files, `lto_a.d` and `lto_b.d`:

```d
// File lto_b.d
// `extern(C++)` is used to be able to define it in C++ later.
extern(C++) void doesNothing() {}
```

```d
// File lto_a.d
extern(C++) void doesNothing(); // Note: declaration only

void main() {
    for (ulong i = 0; i < 1_000_000_000; ++i) {
        doesNothing();
    }
}
```

Let's compile `lto_b.d` first into `lto_b.o`, and then later on compile `lto_a.d` and link it with `lto_b.o`. The program doesn't do anything, and the optimizer should be able to figure that out, but... The optimizer can't. While compiling `lto_a.d`, it has no knowledge of what `doesNothing()` does and can therefore not do much optimization: the program will loop 1 billion times calling a function that immediately returns. On my machine, executing the program takes about 2 seconds:

```
>  ldc2 -c -O3 lto_b.d -of=lto_b.o
>  ldc2 -O3 lto_a.d lto_b.o -of=program
>  time ./program
./program  1.81s user 0.01s system 98% cpu 1.845 total
```

With LTO, `doesNothing()` is imported into the `lto_a` module and the optimizer can work its magic:

```
>  ldc2 -c -O3 -flto=thin lto_b.d -of=lto_b.o
>  ldc2 -O3 -flto=thin lto_a.d lto_b.o -of=program_lto
>  time ./program_lto
./program_lto  0.00s user 0.00s system 28% cpu 0.012 total
```

The same run time is obtained by compiling all source in one compiler invocation:

```
>  ldc2 -O3 lto_a.d lto_b.d -of=program_allatonce
>  time ./program_allatonce
./program_allatonce  0.00s user 0.00s system 44% cpu 0.008 total
```

## Crushing the C++/D language barrier

D can interoperate (relatively) easily with C++ code. LDC itself is a nice example of that: the front-end of LDC is written in D, whereas its backend (LLVM) is written in C++. However, optimization across the C++/D language barrier can not be done by compiling all source at once, because neither the C++ compiler nor the D compiler understand both languages. Thus, for example, a C++ function will not be inlined into a D function:

```cpp
// File lto_b.cpp
void doesNothing() {}
```

```
>  clang -c -O3 lto_b.cpp -o lto_b.o
>  ldc2 -O3 lto_a.d lto_b.o -of=program_cpp
>  time ./program_cpp
./program_cpp  2.09s user 0.01s system 99% cpu 2.125 total
```

The good news: _LTO does not have the language barrier_. Because LTO works at the LLVM IR level and both LDC and Clang compile to the same LLVM IR language, equal optimization potential is achieved for C++-only, D-only, and C++/D-mixed programs!  
For the given example, the execution time is reduced to "zero" with the following build steps:  

```
>  clang -c -O3 -flto=thin lto_b.cpp -o lto_b.o
>  ldc2 -O3 -flto=thin lto_a.d lto_b.o -of=program_cpp_lto -mtriple=x86_64-apple-macosx10.11.0
>  time ./program_cpp_lto
./program_cpp_lto  0.00s user 0.00s system 61% cpu 0.005 total
```

Note that I had to explicitly specify the target triple when invoking `ldc2` (I think this is only needed on OS X). LDC and Clang default to a slightly different triple on Mac OS X, and the LTO codegenerator will complain when the triples are not the same. Interestingly, explicit mention of the triple is not needed when invoking the compilers the other way around, but then you do have to explicitly pass the D runtime libraries to Clang...

## Thanks Teresa!

I'd like to thank Teresa Johnson very much for helping out with troubleshooting the issues I encountered, and for rapidly fixing the LLVM bugs discovered.

## Feedback

Constructive feedback, positive and negative, is always appreciated. The best way to get in touch with me and the other LDC developers is via the [digitalmars.D.ldc forum](https://forum.dlang.org/group/digitalmars.D.ldc) and our [Gitter chat](https://gitter.im/ldc-developers/main).

Links to discussions:  
[Reddit](https://www.reddit.com/r/programming/comments/5c8p58/link_time_optimization_cd_crosslanguage/)  
[D forum thread](https://forum.dlang.org/post/edzoqcprdqtewqqaabge@forum.dlang.org)
