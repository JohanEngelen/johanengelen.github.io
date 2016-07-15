---
layout: post
title:  "Profile-Guided Optimization with LDC"
categories: LDC
---

[Two months ago]({% post_url 2016-04-13-PGO-in-LDC-virtual-calls %}), I measured a 7% performance gain of the compiler front-end D code using Profile-Guided Optimization (PGO) with LDC. Now, part of my PGO work was [merged into LDC master](https://github.com/ldc-developers/ldc/commit/94fe1a6dc8bd63ec5d0fb4b4e2c75eabb74f35a4) on Jun 20, 2016! This means it will be available in the next LDC release (version 1.1.0, needs LLVM 3.7 or newer). You can play with it with the [LDC 1.1.0-alpha1 release](https://github.com/ldc-developers/ldc/releases/tag/v1.1.0-alpha1).
Here I'll discuss how to use it, saving the implementation details for another article.

Using PGO with LDC is similar to using [PGO with Clang](http://clang.llvm.org/docs/UsersManual.html#profiling-with-instrumentation), and much of this article also applies to Clang.

# Profile-Guided Optimization (PGO)
"[Profile-guided optimizations](https://en.wikipedia.org/wiki/Profile-guided_optimization)" are optimizations that the compiler can do when it is given information about the typical execution flow of your program, such that the typical execution flow of your program runs faster (and rare execution flows run slower). The typical execution flow information is the "profile" of your program: how many times was this function called, how often was a control-flow branch taken, how often was function `A` called by function `B`, what are likely values of variable `X`, how much memory is usually allocated for something, etc.

# Building with PGO
A profile can be obtained through different means, but (for now?) LDC only supports _instrumented_ profiling. The compiler adds 'instrumentation' code in a first compilation pass. You then run your program with input that is representative of the use case for which you want to optimize the code, and upon exit the program outputs the profile data in a separate file. This profile data file is  used by the compiler in a second compilation pass to compile and optimize your program.

* Compile with instrumentation on:
`ldc2 -fprofile-instr-generate=profile.raw yourprogram.d -of=instrumented_program`
* Run `instrumented_program` with common workload (the kind of workload you want to optimize the program for). This should create the `profile.raw` file.
* Convert the raw profile to a format that can be used by LDC:
`ldc-profdata merge -output=profile.data profile.raw`
* Recompile your program using the profile data:
`ldc2 -O3 -fprofile-instr-use=profile.data yourprogram.d -of=optimized_program`

Instrumentation makes your program significantly slower, but you do not have to generate a new profile very often. Profile data can be reused when recompiling a program with changes; LDC will detect changes in control flow and will simply ignore outdated profile data. More on this below.

# Profile output filename

The profile output filename can be specified during compilation as shown above (`-fprofile-instr-generate=<filename>`). If no filename is specified, the environment variable `LLVM_PROFILE_FILE` is used instead. If `LLVM_PROFILE_FILE` is not defined, the default filename `default.profraw` is used.

Often, a single run of your program is not enough to create a statistically representative profile. You probably want to profile several runs of the instrumented program with different inputs. You could set `LLVM_PROFILE_FILE` to a different filename for each run of your program, but the profile runtime also supports substitution specifiers in the profile file name to help you out. The specifiers `%p` and `%h` in the filename are substituted with the process pid and the hostname, respectively (`%h` needs LLVM >= 3.9). If you compile with `-fprofile-instr-generate=test.%p.raw` and run your program a few times, you'll end up with a set of profile data files `test.*.raw`. You can then use `ldc-profdata` to merge them as usual: `profdata merge -o test.profdata test.*.raw`.

# Examining profile data

The `ldc-profdata` tool (a renamed version of [`llvm-profdata`](http://llvm.org/docs/CommandGuide/llvm-profdata.html) that matches the LLVM version that `ldc2` was built with) can be used to inspect the contents of profile data files. Let's look at the profile of this program:

```cpp
extern(C)
void foo(int x) {
    if (x > 0) {
        if (x > 500) {
            // ...
        }
    }
}

void main() {
    foreach (i; 0 .. 1000) {
        foo(i);
    }
}
```

We compile and run it with `ldc2 -fprofile-instr-generate=default.profraw -run test_foo.d`. Then, `ldc-profdata show -all-functions default.profraw` outputs:

```
Counters:
  foo:
    Hash: 0x00000000000002cb
    Counters: 3
    Function count: 1000
  _Dmain:
    Hash: 0x0000000000000004
    Counters: 2
    Function count: 1
Functions shown: 2
Total functions: 2
Maximum function count: 1000
Maximum internal block count: 1000
```

The profile data file contains a data structure with counters and other information for each function. Here we see that the program has only two instrumented functions: `foo` and `_Dmain`. Function `foo` contains 3 execution counters (function entry plus a counter for each if-statement), and the function was called 1000 times ("Function count"). The "hash" field is used by LDC to check whether the profile data can be used in the `-fprofile-instr-use` compile step.

# Re-use of profile data

LDC can reuse profile data obtained with an older version of your program.
To enable profile re-use even if the source code has changed a little, a hash of the function control flow is stored in the profile. In the `-fprofile-instr-use` compile step, LDC ensures that the function hash in the profile is equal to the hash of the code being compiled. If there is no match, LDC simply ignores profile data for that function. The hash is not perfect: different pieces of code result in the same hash, thus a profile obtained with an older version of the code may be applied to the new code. An example:

```cpp
void foo(int x) {
    if (x > 0) { // 1000x true, 4x false
        // ...
    }
}
```
The function `foo` is almost always called with positive `x`, and the profile data for the `if` statement is "branch taken 1000 times, not taken 4 times". If the code is now changed from `x > 0` to `x < 0`, the function hash stays the same (current LDC behavior) and old profile data will wrongly be used.

```cpp
void foo(int x) {
    if (x < 0) { // With old data: 1000x true, 4x false. Oops!
        // ...
    }
}
```

The hash could be made to depend on more features of the code, such as distinguishing `>` from `<` expressions. At the moment the hashing is mainly done to prevent compile errors when there is not enough or too much profile information, so e.g. only the existence of the if-statement is put into the hash. The imperfection of the hash is by design, because some changes are not likely to change the validity of old profile data. Consider a code change from `x > 1000` to `x > 1001`. In this case, the old profile data is probably still good enough and it'd be worse for performance to discard it.

No need for distress: PGO does not change the semantics of D, it is merely an optimization technique. So if the profile data no longer matches with what would happen with the modified code, the worst that can happen is that the performance of your program decreases. Because you are already measuring (right?) the performance of your program with and without PGO, you'll easily see when the profile needs updating.

# Reset profile after startup code
The `ldc.profile` module contains functions to interface with the profile instrumentation. I think the only really useful function is `resetAll()`. The functionality exposed by the other `ldc.profile` functions is very low-level and may become unavailable with future compiler/LLVM versions.
`resetAll()` resets all profiling information and can be used to remove your program's start-up behavior from the profile:

```cpp
import ldc.profile : resetAll;
void main()
{
    initializeProgram();

    resetAll();

    operate();
}
```

This way, PGO will optimize the program for best performance of `operate()`, possibly reducing the performance of `initializeProgram()`.

# Function-level enabling/disabling of profiling

The profiling implementation works on a per-function basis, and it is therefore straightforward to turn instrumentation on/off for specific functions with the new `pragma(LDC_profile_instr, {true|false})`. The pragma has the same semantics as `pragma(inline,...)`.

```cpp
void instrumented() {}
void not_instrumented() {
    pragma(LDC_profile_instr, false);
}
pragma(LDC_profile_instr, false) {
    void not_instrumented_2() {}

    void instrumented2_override() {
        pragma(LDC_profile_instr, true);
    }

    pragma(LDC_profile_instr, true):
    void instrumented3() {  }
    void instrumented4() {  }
}

pragma(LDC_profile_instr, false)
struct Strukt {
    void not_instrumented() {}
    void instrumented_method() {
        pragma(LDC_profile_instr, true);
    }
}
```

Instrumentation is _on_ per default. So for very selective instrumentation, you'd need to put `pragma(LDC_profile_instr, false):` at the start of every file. But even then, imported functions will still be instrumented. If there is enough demand, it is easy to implement a commandline switch that turns instrumentation _off_ per default so it is less hassle to only instrument just a certain set of functions.


# LLVM 3.9 outlook
PGO is under very active development in LLVM, and the next LLVM release (3.9) is going to rock new PGO features, for example more optimization passes use profile data and there is a new PGO-only optimization pass ([indirect-call promotion](http://reviews.llvm.org/D17864)). So you can expect an extra PGO performance boost going from LLVM 3.8 to 3.9.

The LLVM 3.9 profile runtime can merge profiles on the fly, using the `%m` specifier in the profile filename. Compilation with `-fprofile-instr-generate=test.%m.raw` and running the instrumented program will result in a single profile file with data from all runs merged. Running multiple instances of your program concurrently should work correctly! You can tell the runtime to use a pool of files to reduce contention: with `%3m` a pool of 3 files is used.

