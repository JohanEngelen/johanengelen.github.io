---
layout: post
title:  "Finding memory bugs in D code with AddressSanitizer"
categories: LDC
---

_LDC comes with improved support for [Address Sanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer) since the 1.4.0 release. Address Sanitizer (ASan) is a runtime memory write/read checker that helps discover and locate memory access bugs. ASan is part of the official LDC release binaries; to use it you must build with `-fsanitize=address`.
In this article, I'll explain how to use ASan, what kind of bugs it can find, and what bugs it will be able to find in the (hopefully near) future.
LDC does not yet support ASan on Windows._

# Address Sanitizer

[Address Sanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer) (or ASan for short) is a fast memory access error detector. When it detects that a program tries to read/write from/to an invalid memory address, it aborts the program and outputs an error report with many details about the error. To use ASan you have to compile with ASan enabled (`-fsanitize=address`) and you have to link with the ASan runtime library (again `-fsanitize=address` when linking with LDC).

ASan is developed for catching bugs in C++ codebases. Although D is a much more memory-safe language, the safety measures do require some developer effort and discipline. The same type of memory bugs that happen in C++ can happen in D too. ASan to the rescue! To pique your interest, at CppCon 2017, Louis Brandy, Engineering Director at Facebook, notes that ["AdressSanitizer is probably the most important thing that's happened, at least in our tool set, in a long time."](https://youtu.be/lkgszkPnV8g?t=409) And ["one cannot speak highly enough of AddressSanitizer, and the amount of bugs that it finds in a codebase."](https://youtu.be/lkgszkPnV8g?t=2649)

ASan catches bad memory accesses at runtime by instrumenting[^instrum] memory accesses during compilation, where the instrumentation code checks the accessed address range against a table of accessible/inaccessible memory (stored in a "shadow" memory region). The instrumentation of code includes adding checks on every read/write from memory and adding calls to mark specific memory regions as valid ('unpoisoned') or invalid ('poisoned') for access. The instrumentation and runtime library add poisoned 'red zones' around valid memory regions (stack variables and heap), such that buffer over-/underflow can be detected. The ASan runtime library provides the memory marking/checking and error reporting functions, but also overrides `malloc` and `free` such that accesses to dynamically allocated memory regions can also be checked for validity. With this mechanism, ASan detects for example use-after-free bugs and heap buffer overflows.

Bugs involving GC allocated memory are not detected (yet!). I'll return to this issue at the end of the article.


[^instrum]: Here, "instrumenting" means inserting extra code around the instrumented operations.


# A simple example

A contrived example of a bug easily caught with ASan:

```c++
// File: asan_1.d
void foo(int* arr) {
    arr[10] = 1; // BOOM !!!
}

void main() {
    int[10] tenIntegers;
    foo(&tenIntegers[0]);
}
```

Let's look at the runtime output of this small buggy program `asan_1.d`. We build with `-fsanitize=address` and then run the program:

```shell
> ldc2 -fsanitize=address -g asan_1.d
> ./asan_1
```

This outputs[^colors]:

[^colors]: The ASan output is nicely colorized in the terminal, but I have not figured out how to easily add colors in these fragments with Jekyll... Let me know if you do know.

```
=================================================================
==15099==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7fff5dfce3a8 at pc 0x000101c31b97 bp 0x7fff5dfce330 sp 0x7fff5dfce328
WRITE of size 4 at 0x7fff5dfce3a8 thread T0
    #0 0x101c31b96 in _D6asan_13fooFPiZv asan_1.d:3
    #1 0x101d6238e in _D2rt6dmain211_d_run_mainUiPPaPUAAaZiZ6runAllMFZ9__lambda1MFZv (asan_1:x86_64+0x10013138e)
    #2 0x101c31df4 in main __entrypoint.d:8

Address 0x7fff5dfce3a8 is located in stack of thread T0 at offset 72 in frame
    #0 0x101c31bbf in _Dmain asan_1.d:6

  This frame has 1 object(s):
    [32, 72) '' <== Memory access at offset 72 overflows this variable
HINT: this may be a false positive if your program uses some custom stack unwind mechanism or swapcontext
      (longjmp and C++ exceptions *are* supported)
SUMMARY: AddressSanitizer: stack-buffer-overflow asan_1.d:3 in _D6asan_13fooFPiZv
Shadow bytes around the buggy address:
  0x1fffebbf9c20: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1fffebbf9c30: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1fffebbf9c40: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1fffebbf9c50: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1fffebbf9c60: 00 00 00 00 00 00 00 00 00 00 00 00 f1 f1 f1 f1
=>0x1fffebbf9c70: 00 00 00 00 00[f3]f3 f3 f3 f3 f3 f3 00 00 00 00
  0x1fffebbf9c80: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1fffebbf9c90: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1fffebbf9ca0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1fffebbf9cb0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1fffebbf9cc0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
...
==15099==ABORTING
```

The program aborts because ASan has intercepted a bad memory access: a `stack-buffer-overflow` due to a `WRITE of size 4` (bytes) at program location `0x101c31b96 in _D6asan_13fooFPiZv asan_1.d:3`. File `asan_1.d` line 3 is the line with `// BOOM !!!`. So not only did ASan detect a bug, it immediately shows us where the bug happens. ASan also reports in which stack frame the bad memory address lies (`_Dmain asan_1.d:6`) and it tells us which variable is close to that address:

```
 This frame has 1 object(s):
    [32, 72) '' <== Memory access at offset 72 overflows this variable
```

What's not working yet is showing the name of that variable: it should say `[32, 72) 'tenIntegers'`, but that's not working for D (yet! Perhaps a problem only on OSX).

What follows is a view of the shadow memory around the accessed memory address indicated between `[` `]`. The value `f3` means that the accessed location is in the 'stack right redzone', i.e. the address is right after the stack allocated variable. Note that the shadow region is mostly unpoisoned and addressable. This means that only bad accesses to memory close to the variable are detected by ASan. If I change line 3 to `arr[30] = 1;`, then ASan does not detect the bug on my system.

Note that in idiomatic D code one would use a slice instead of a pointer, and D's boundschecking would catch the bug:
```c++
void foo(int[] arr) {
    arr[10] = 1; // Boom! Caught by D's boundschecking. ASan not needed.
}

void main() {
    int[10] tenIntegers;
    foo(tenIntegers);
}
```

# Bug in LDC's druntime

Let's look at a [bug in LDC's druntime](https://issues.dlang.org/show_bug.cgi?id=17821), prior to version 1.4.0, that would have been easily found with ASan.
The bug was [reported by Eyal from Weka.io](https://issues.dlang.org/show_bug.cgi?id=17821), and I fear it cost him a lot of time to figure out what was broken: `core.atomic.atomicStore` would do a stack overread when passed a `ulong` (8 bytes) and an `int` (4 bytes), resulting in `atomicStore` writing garbage to one half of the `ulong`. Calling `atomicStore` with `ulong` and `int` may happen quite easily, as you can see in this code fragment:

```c++
// File: asan_2.d
import core.atomic : atomicStore;
void main() {
    shared ulong x = 0x1234_5678_8765_4321;
    atomicStore(x, 0); // 0 is of type `int`
    // assert(x == 0); // failed prior to LDC 1.4.0
}
```

Without the assert, the code executed without crashing or aborting; needless to say, when something as basic as storing data is broken it leads to severe bugs. With ASan enabled, the bug is immediately found and ASan reports a stack buffer overflow error!

```
=================================================================
==22331==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7fff57a13360 at pc 0x0001081ece5c bp 0x7fff57a13330 sp 0x7fff57a13328
READ of size 8 at 0x7fff57a13360 thread T0
    #0 0x1081ece5b in _Dmain atomic.d:389
```

In this case the ASan output is harder to correlate with the source code because the call to `core.atomic.atomicStore` is inlined even at `-O0` because of `pragma(inline, true)`. I'm on OSX, so I need to run `dsymutil` on the `asan_2` binary first. Then after adding `llvm-symbolizer` to the path, I get a better stack trace that does show `_Dmain` calling `core.atomic.atomicStore`:

```
=================================================================
==22051==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7fff562c0360 at pc 0x00010993fe5c bp 0x7fff562c0330 sp 0x7fff562c0328
READ of size 8 at 0x7fff562c0360 thread T0
    #0 0x10993fe5b in _D4core6atomic50__T11atomicStoreVE4core6atomic11MemoryOrderi3TmTiZ11atomicStoreFNaNbNiNeKOmiZv <..>/druntime/src/core/atomic.d:389:9
    #1 0x10993fe5b in _Dmain asan_2.d:5
    #2 0x109a7a3be in _D2rt6dmain211_d_run_mainUiPPaPUAAaZiZ6runAllMFZ9__lambda1MFZv (./asan_2:x86_64+0x10013b3be)
    #3 0x109940214 in main __entrypoint.d:8:15

Address 0x7fff562c0360 is located in stack of thread T0 at offset 32 in frame
    #0 0x10993fcef in _Dmain /Users/johan/github.io/johanengelen.github.io/code/asan_2.d:3

  This frame has 2 object(s):
    [32, 36) '' <== Memory access at offset 32 partially overflows this variable
    [48, 56) ''
HINT: this may be a false positive if your program uses some custom stack unwind mechanism or swapcontext
      (longjmp and C++ exceptions *are* supported)
SUMMARY: AddressSanitizer: stack-buffer-overflow /Users/johan/ldc/johan/runtime/druntime/src/core/atomic.d:389:9 in _D4core6atomic50__T11atomicStoreVE4core6atomic11MemoryOrderi3TmTiZ11atomicStoreFNaNbNiNeKOmiZv
Shadow bytes around the buggy address:
  0x1fffeac58010: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1fffeac58020: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1fffeac58030: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1fffeac58040: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1fffeac58050: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
=>0x1fffeac58060: 00 00 00 00 00 00 00 00 f1 f1 f1 f1[04]f2 00 f3
  0x1fffeac58070: f3 f3 f3 f3 00 00 00 00 00 00 00 00 00 00 00 00
  0x1fffeac58080: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1fffeac58090: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1fffeac580a0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1fffeac580b0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

The output nicely shows that at the address accessed (look for the `[ ]` brackets), only 4 bytes are addressible instead of the 8-byte read attempt. The [bug fix](https://github.com/ldc-developers/druntime/pull/102) is in LDC 1.4.0, so you won't be able to reproduce it anymore with the `asan_2.d` code ;-)

The lesson is clear: we should start running Phobos and druntime testsuites with ASan enabled as soon as possible!

# Blacklisting

Some code will need to access memory regions that are protected by ASan. For example, stack tracing code necessarily reads beyond the limits of user variables. Without special precautions, such stack tracing code would trigger an ASan stack-buffer-overflow error. Notably, the standard libraries contain such code. We do need to ASan instrument the standard library however, to find bugs with incorrect usage of (or bugs inside) the standard library. For such purpose, we can specify a blacklist such that functions that match the blacklist will not be instrumented.

To specify which functions should not be instrumented by ASan, use the `-fsanitize-blacklist=<filename>` compiler flag. An example of such a blacklist file:

```
# File: asan_blacklist.txt

# Blacklist for the sanitizers. Turns off instrumentation of particular
# functions or sources. Use with care. You may set location of blacklist
# at compile-time using -fsanitize-blacklist=<path> flag.

# Example usage:
# fun:*bad_function_name*
# src:file_with_tricky_code.cc

# Blacklist druntime functions whose inline assembly does not sit well with
# ASan yet, see https://github.com/ldc-developers/ldc/issues/2257
fun:_D*callWithStackShell*
fun:_D*getcacheinfoCPUID2*

# Functions from the conservative gc druntime
fun:*12conservative2gc*
fun:_D4core5cpuid*
```

In this `asan_blacklist.txt` file, I've disabled instrumentation of a few functions that are in druntime and Phobos, such that ASan doesn't trigger on the "invalid" memory operations performed by that code. Note: some functions that match this blacklist may really be buggy!
We can use the [`ldc-build-runtime` tool](https://wiki.dlang.org/Building_LDC_runtime_libraries) to build ASan-enabled druntime and Phobos standard libraries with this blacklist:

```bash
> ./ldc-build-runtime --dFlags='-fsanitize=address;-fsanitize-blacklist=asan_blacklist.txt' BUILD_SHARED_LIBS=OFF
```

Now for running the Phobos and druntime testsuites, we need to explicitly specify the ASan library for the linker too (I've intentionally retained the absolute paths of my system):

```bash
> ./bin/ldc-build-runtime --dFlags='-fsanitize=address;-fsanitize-blacklist=/Users/johan/ldc/ldc2-1.7.0-beta1-osx-x86_64/asan_blacklist.txt' BUILD_SHARED_LIBS=OFF --testrunners --linkerFlags='/Users/johan/ldc/ldc2-1.7.0-beta1-osx-x86_64/lib/libldc_rt.asan_osx_dynamic.dylib'
```

We will need to study this much more, and in the future LDC will likely ship with an ASan blacklist for Phobos and druntime.
The blacklist given above is a good starting point, but building ASan-enabled standard libraries is at an early stage and some real effort is still needed to make it work.
But it is not at all necessary to use an ASan-enabled standard library right now. Without ASan-enabled standard libs, ASan testing will still cover your code and (a lot of) standard library _templated_ code. If you want to try ASan out on your own code, you can start by ignoring the D standard libraries.


# Future work: detecting stack use after return

D's [`@safe`](https://dlang.org/spec/function.html#safe-functions) attribute (together with [`-dip1000`](https://github.com/dlang/DIPs/blob/master/DIPs/DIP1000.md)) prevents code from doing any memory unsafe operations. Returning a reference to a local stack variable from an `@safe` function is rejected by the compiler, but there are corner cases where it is still possible to do so, as discussed [in a forum post last August](https://forum.dlang.org/post/kfdekipdbxnrawzfczva@forum.dlang.org). Here is a contrived example demonstrating the bug:

```c++
class A {
    int i;
}

void inc(A a) @safe {
    a.i += 1; // Line 6
}

auto makeA() @safe {  // Line 9
    import std.algorithm : move;
    scope a = new A(); // allocates object on the stack instead of the heap
    return move(a);
}

void main() @safe {
    auto a = makeA();
    a.inc(); // Line 17
}
```

Note that the compiler should have caught this bug (at compile time!), but it currently [does not](https://issues.dlang.org/show_bug.cgi?id=18128). Regardless, ASan will find these kind of bugs for us too.

We can pass [extra run-time flags to ASan](https://github.com/google/sanitizers/wiki/AddressSanitizerFlags) by setting the `ASAN_OPTIONS` environment variable. There are a number of interesting options, but here I'll only show what setting `detect_stack_use_after_return` does.
Let's also pipe the output through `ddemangle` so we get human-readable D function names.

```bash
> ldc2 -fsanitize=address -disable-fp-elim scopeclass.d -g -O1 -dip1000
> ASAN_OPTIONS=detect_stack_use_after_return=1 ./scopeclass 2>&1 | ddemangle
```

Read the full output of ASan and you'll understand the bug (use of a stack allocated variable after exiting that stack scope):

```
=================================================================
==11446==ERROR: AddressSanitizer: stack-use-after-return on address 0x000104929050 at pc 0x0001007a9837 bp 0x7fff5f457510 sp 0x7fff5f457508
READ of size 4 at 0x000104929050 thread T0
    #0 0x1007a9836 in @safe void scopeclass.inc(scopeclass.A) scopeclass.d:6
    #1 0x1007a9a20 in _Dmain scopeclass.d:17
    #2 0x1008e40ce in _D2rt6dmain211_d_run_mainUiPPaPUAAaZiZ6runAllMFZ9__lambda1MFZv (scopeclass:x86_64+0x10013c0ce)
    #3 0x7fff9729b5ac in start (libdyld.dylib:x86_64+0x35ac)

Address 0x000104929050 is located in stack of thread T0 at offset 80 in frame
    #0 0x1007a984f in pure nothrow @nogc @safe scopeclass.A scopeclass.makeA() scopeclass.d:9
```

**Big caveat**: [ASan's stack-use-after-return detection](https://github.com/google/sanitizers/wiki/AddressSanitizerUseAfterReturn) works by allocating stack variables on the heap using malloc, but that memory is not (yet) registered with the garbage collector (GC). This means that the GC no longer sees stack allocated pointers, and will incorrectly collect memory if it is only pointed to by stack pointers... Very bad! So although this example here worked, for larger programs things will definitely _not_ work well. Further work is needed to have the instrumentation code add the dynamically allocated stack to the list of GC-scanned memory ranges [^llvmpass].

[^llvmpass]: It's a work in progress. A glimpse: I am adding an LDC LLVM pass right after the ASan pass, that adds calls to `GC.addRange` and `GC.removeRange` right after `__asan_stack_malloc` and `__asan_stack_free`, respectively. But instead of a call to `__asan_stack_free`, sometimes LLVM directly emits the equivalent of the inlined call. In that case, my pass doesn't know where to add `GC.removeRange` and things become very complicated. I'll have to extend LLVM to add calls to optional user-defined functions after `__asan_stack_malloc` and `__asan_stack_free`. Work in progress...

# Future work: memory allocated by the garbage collector (GC)

With the current LDC implementation, ASan does not detect bugs related to memory that has been allocated by the GC. This is because all memory managed by the GC is in large GC pools that have been pre-allocated by `malloc` and are thus marked valid/unpoisoned (by ASan's override of `malloc`). When user code allocates (`new`), a memory portion of the GC memory pool is given to the user. Only after that allocation by the user should the memory be valid to access. However, because `malloc` already marked it as valid, ASan will not trigger on overreads, etc.

Clearly, this is a major missing part of ASan for D.
What must be implemented in druntime is the poisoning of the GC pool memory, and only unpoisoning the parts that have been given to user code (and poisoning it again after it is no longer used and has been collected).

# Closing remarks

The memory-bug situation in D isn't nearly as bad as in C++ partly because of garbage collection, array slices, and `@safe`. The fact that the examples I use are either non-idiomatic D code or compiler bugs should strengthen your belief that D really is a much safer language than C++. However in practice, not all D code uses `@safe` and [DIP1000](https://github.com/dlang/DIPs/blob/master/DIPs/DIP1000.md), and developers keep using pointers more often than they should, and so the risk of memory bugs remains. Using ASan should be part of your testing _in addition to_ using the memory safety features already present in D.

I hope that this article helps you in using ASan in your projects. Compiling with ASan enabled is the same as with Clang and GCC: `-fsanitize=address`, simple!

ASan's bug-finding capabilities mesh very well with fuzzing, and I've been able to easily find [a bug](https://github.com/dlang/dmd/pull/7050) in the DMD frontend (the topic of a future article). Needless to say for open-source code: your adversaries will use ASan, so you'd better do so too.

If you have any issues in using ASan with LDC, report them in [LDC's bugtracker](https://github.com/ldc-developers/ldc/issues). Or consult [the D Learn forum](http://forum.dlang.org/group/learn) when ASan reports a bug but you can't find it.


# Acknowledgments

I'd like to thank the folks at [Weka.io](https://weka.io), who own the world's most awesome D codebase, and who partly sponsor my work on LDC.

# Feedback

Constructive feedback, positive and negative, is always appreciated. The best way to get in touch with me and the other LDC developers is via the [digitalmars.D.ldc forum](https://forum.dlang.org/group/digitalmars.D.ldc) and our [Gitter chat](https://gitter.im/ldc-developers/main).

# Footnotes
