---
layout: post
title:  "Fuzzing D code with LDC"
categories: LDC
---

_A not-so-well-written article about the fuzzing capability recently added to LDC, using LLVM's libFuzzer. Compiling code with `-fsanitize=fuzzer` adds control-flow instrumentation used to guide the fuzzing and links-in the `libFuzzer` library that drives the fuzz testing (same as Clang).
`-fsanitize=fuzzer` is available from LDC 1.4.0, not on Windows. LDC 1.6.0 was used for the examples in this article._

# Fuzzing and libFuzzer

Fuzz testing ("fuzzing") is a technique to find bugs by testing a (part of a) program many times with randomly generated input. After fuzz testing, typically you end up with a large collection of input files that test the majority of execution paths of your program, and a collection of input files for which the program crashed (hopefully none). Key ingredients to fuzz testing are "many" and "random". With "random", I mean that the inputs are generated without necessarily knowing much about the kind of input the program usually receives. That results in many garbage tests, so we have to run _many_ of such tests (e.g. more than 500 tests per second). To guide the pseudo-random input generation, we can instrument the code such that the fuzzer can detect whether a new input generated a previously unseen execution trace (different state transitions are assumed to be more interesting (i.e. buggy) than the same execution trace with different values). This is called coverage-guided fuzz testing. The fuzz testing should be fast, and therefore the instrumentation and detection of control flow is approximate and a good balance between accuracy and speed is important. A famous fuzzing tool is [American Fuzzy Lop (AFL)](http://lcamtuf.coredump.cx/afl/). A concise explanation of AFL's implementation of coverage-guided fuzz testing is [the `afl-fuzz` whitepaper](http://lcamtuf.coredump.cx/afl/technical_details.txt); highly recommended reading.
But I have yet to try AFL... (It'd be great to see results with AFL using `ldc2 -fsanitize-coverage=trace-pc-guard`).  
In this article, I will discuss fuzz testing with LLVM's `libFuzzer`, which ships with LDC since version 1.4.0.

To use libFuzzer, we have to compile with [LLVM's code coverage instrumentation](https://clang.llvm.org/docs/SanitizerCoverage.html) enabled, and we need to link with the libFuzzer runtime library. The `libFuzzer` library implements `main` and drives the fuzz testing, sending test input to the `LLVMFuzzerTestOneInput` function that the user must define. LDC, like Clang, has a commandline switch specifically for fuzzing: `-fsanitize=fuzzer`. That switch sets `-fsanitize-coverage=trace-pc-guard,indirect-calls,trace-cmp` and links with libFuzzer runtime library.  
In this article, I will show examples of how to use the fuzzer, so you can get started on fuzzing your own projects. For more information about libFuzzer, read [its documentation](https://llvm.org/docs/LibFuzzer.html) and [the libFuzzer tutorial](https://github.com/google/fuzzer-test-suite/blob/master/tutorial/libFuzzerTutorial.md).

# Runtime checks

The effectiveness of fuzzing can be greatly enhanced by having the compiler insert extra runtime checks for bad behavior. For example, without bounds checking, an array overread may (by chance) not be fatal for your program and fuzz testing will not find the bug. At the same time however, you want the fuzz testroutine to be as fast as possible. I can't solve that trade-off between fuzz speed and extra runtime checks for you, but I'd err on the side of extra checks. You could create a test corpus (discussed later) fast without extra runtime checks at first, and then use that corpus when fuzz testing _with_ extra runtime checks.

One of the safety features built into D is bounds checking on array accesses. Many people disable those bounds checks with `-release` or `-boundscheck=off` to get extra performance. However I'm sure that you'll discover more bugs with bounds checking _enabled_ during fuzz testing. The built-in bounds checking can also be circumvented by accessing an array through its `.ptr` property: i.e. `array.ptr[i]` instead of the usual `array[i]`. One reason for using `.ptr` for access is to disable bounds checking locally for that particular array access, but leave bounds checking enabled for the non-performance-critical parts of your program. However, using `.ptr` permanently disables the built-in bounds checking, and the check cannot be re-enabled for fuzz testing (without modifying the code).

A broader set of runtime checks can be added by using [Address Sanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer) (or ASan for short). ASan is a fast memory error detector that operates in conjunction with compiler instrumentation on memory reads/writes (i.e. extra runtime checks). ASan will detect out-of-bounds array accesses regardless of whether `array[i]` or `array.ptr[i]` is used. Currently, ASan is not yet (!) able to detect bugs with GC-allocated memory and therefore ASan is not yet able to catch some bugs that D's built-in bounds checking would catch; thus: enable them both.  
I wrote a [separate article about the use of ASan with LDC]({% post_url 2017-12-25-LDC-and-AddressSanitizer %}), so here I will only add that ASan is enabled by compiling with `-fsanitize=address`.

# A very simple example

Let's look at a small fuzz target with a bug:

```d
// File: example.d

bool FuzzMe(const(ubyte[]) data)
{
    return (data.length >= 3) &&
       data[0] == 'F' && data[1] == 'U' && data[2] == 'Z' &&
       data[3] == 'Z'; // <-- access possibly out of bounds
}

extern (C) int LLVMFuzzerTestOneInput(const(ubyte*) data, size_t size)
{
    FuzzMe(data[0 .. size]);
    return 0;
}
```

We compile with `-fsanitize=fuzzer` and run the program (a big part of the output has been cut).
```
> ldc2 -fsanitize=fuzzer -g ../example.d -of=example
> ./example
...
==18381== ERROR: libFuzzer: deadly signal
SUMMARY: libFuzzer: deadly signal
MS: 3 EraseBytes-ChangeBit-EraseBytes-; base unit: 3e67ddbe5fee7aa7241d0db7cb27b2199c34091b
0x46,0x55,0x5a,
FUZ
artifact_prefix='./'; Test unit written to ./crash-0eb8e4ed029b774d80f2b66408203801cb982a60
...
```

The fuzzer has found an input for which a crash has happened. The input bytes causing the crash (`0x46,0x55,0x5a`, the ASCII encoding of `FUZ`) have been written to the file `crash-0eb8e4ed029b774d80f2b66408203801cb982a60`. We can use this file to reproduce the crash. Note that we now have all we need for a bug report: we have the reproducer code (the fuzz target) and its input (the `crash-*` file).

 Let's load the example in the debugger and see what causes the crash.

```
> lldb example
(lldb) target create "example"
Current executable set to 'example' (x86_64).
(lldb) run crash-0eb8e4ed029b774d80f2b66408203801cb982a60
...
/Users/johan/github.io/johanengelen.github.io/code/fuzz/run_example/example: Running 1 inputs 1 time(s) each.
Running: crash-0eb8e4ed029b774d80f2b66408203801cb982a60
Process 21839 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=1, address=0x0)
    frame #0: 0x000000010017073d example`gc_malloc + 13
...
(lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=1, address=0x0)
  * frame #0: 0x000000010017073d example`gc_malloc + 13
    frame #1: 0x000000010015daaa example`onRangeError + 26
    frame #2: 0x000000010015e5f8 example`_d_arraybounds + 8
    frame #3: 0x00000001000010af example`_D7example6FuzzMeFxAhZb(data=(length = 3, ptr = "FUZ")) at example.d:5
    frame #4: 0x0000000100001100 example`LLVMFuzzerTestOneInput(data="FUZ", size=3) at example.d:12
...
```

The crash backtrace shows us how the crash happened: inside `_D7example6FuzzMeFxAhZb` [^wrongline] (`example.FuzzMe`) boundschecking code called `_d_arraybounds` which found the index to be out-of-bounds and called `onRangeError` which called `gc_malloc`, and then the program crashes...? Why did `gc_malloc` crash? We didn't initialize druntime!

[^wrongline]: Note that the line number is wrong for the out-of-bounds array access. It's a [bug in the debuginfo generated](https://github.com/ldc-developers/ldc/issues/2090) by LDC.

# The complete very simple example

We have to initialize druntime manually, because `libFuzzer` implements `main` and there is no "D main" in our program. Note that our fuzz target will be called many times, but we should initialize druntime only _once_. Also, we have to set up a try-catch clause to catch and report all uncaught exceptions in our test. The complete simple example looks like this:

```d
// File: example2.d

bool FuzzMe(const(ubyte[]) data)
{
    return (data.length >= 3) &&
           data[0] == 'F' && data[1] == 'U' && data[2] == 'Z' &&
           data[3] == 'Z'; // <-- access possibly out of bounds
}

int LLVMFuzzerTestOneInput(const(ubyte[]) data)
{
    FuzzMe(data);
    return 0;
}

// Boilerplate that you should not have to write in the future:
extern (C) void _d_print_throwable(Throwable t);
extern (C) int LLVMFuzzerTestOneInput(const(ubyte*) data, size_t size) {
    // D runtime must be initialized, but only once.
    static bool init = false;
    if (!init) {
        import core.runtime : rt_init;
        rt_init();
        init = true;
    }
    try {
        return LLVMFuzzerTestOneInput(data[0 .. size]);
    }
    catch (Throwable t) {
        _d_print_throwable(t);
        import ldc.intrinsics : llvm_trap;
        llvm_trap();
    }
    assert(0);
}
```

With that, we are all set and the output when passing the crashing testcase "FUZ" looks like this:

```
> ldc2 -fsanitize=fuzzer -g ../example2.d -of=example2
> ./example2 crash-0eb8e4ed029b774d80f2b66408203801cb982a60
...
INFO: Seed: 3516036321
INFO: Loaded 1 modules (24 guards): [0x1096b7950, 0x1096b79b0),
./example2: Running 1 inputs 1 time(s) each.
Running: crash-0eb8e4ed029b774d80f2b66408203801cb982a60
core.exception.RangeError@../example2.d(7): Range violation
----------------
0   example2                            0x000000010957e9b9 object.Throwable.TraceInfo core.runtime.defaultTraceHandler(void*) + 137
1   example2                            0x000000010959218d _d_throw_exception + 77
2   example2                            0x000000010957ca6e onRangeError + 158
3   example2                            0x000000010957d537 _d_arraybounds + 7
4   example2                            0x000000010941fe8e bool example2.FuzzMe(const(ubyte[])) + 734
5   example2                            0x000000010941fedf int example2.LLVMFuzzerTestOneInput(const(ubyte[])) + 63
6   example2                            0x000000010941ff8f LLVMFuzzerTestOneInput + 159
7   example2                            0x000000010942baf0 _ZN6fuzzer6Fuzzer15ExecuteCallbackEPKhm + 336
8   example2                            0x0000000109420781 _ZN6fuzzer10RunOneTestEPNS_6FuzzerEPKcm + 257
9   example2                            0x0000000109424b59 _ZN6fuzzer12FuzzerDriverEPiPPPcPFiPKhmE + 6105
10  example2                            0x0000000109431fd2 main + 34
==27357== ERROR: libFuzzer: deadly signal
```

For fuzzing your own projects, I recommend copying the boilerplate code from `example2.d`.


Let's look at applying fuzz testing to real-world code. I'll use a testcase I'm familiar with: the D compiler.


# Fuzzing the compiler's frontend lexer (July 2017)

_(The fuzzing presented in this section I did in July 2017. Since then I've fixed the bugs in the frontend, so you won't be able to reproduce them with recent versions of the DMD package. I had to reproduce the bugs again to finish this article, and used DMD git hash `d822c9a18af7c7f7e71ae039f44373d028f2bfce`)_

Since July 2017, the compiler frontend's lexer is [available as a library](https://forum.dlang.org/post/okktlu$2bin$1@digitalmars.com), so it is easier to use inside another piece of software. [Lexical analysis](https://en.wikipedia.org/wiki/Lexical_analysis) is the first step of compilation and converts a sequence of characters to a sequence of tokens. The lexer should accept arbitrary data buffers, so how about fuzz testing it? :-)

The [interface of the Lexer class](https://dlang.org/phobos/dmd_lexer.html#.Lexer) requires that the source code buffer is null terminated[^nullterm]. The data provided by `libFuzzer` is not null-terminated, so we have to copy the input data to our own buffer and append a null-termination character, before passing it to the Lexer.
We have to dynamically allocate our own buffer, and we should use `malloc` instead of the GC. This is because we need to use ASan to catch bugs involving that buffer and currently LDC's ASan implementation does not catch buffer overreads with GC'd memory (see [my recent article on LDC and ASan]({% post_url 2017-12-25-LDC-and-AddressSanitizer %})). One final thing to note is that we have to make sure that our fuzz target is doing something that the compiler will think is useful. That is, the code must have an externally-observable effect, otherwise it will be completely eliminated by the optimizer. We do this by simply summing the values of all tokens in a global variable `sum`.

[^nullterm]: The Lexer constructor wants to know the size of the buffer, so I didn't expect that the buffer also needs to be null-terminated. Of course, fuzzing immediately found my wrong API use without null termination :-)

```d
// File fuzzlexer.d
import ddmd.tokens;
import ddmd.lexer;
import core.stdc.stdlib : malloc, free;
import core.stdc.string : memcpy;

// Add the boilerplate from example2.d.

TOK sum; // Accumulate the sum of token values to prevent dead-code elimination

int LLVMFuzzerTestOneInput(const(ubyte[]) data)
{
    if (data.length < 1)
        return 0;

    // Always null-terminate the input data, use malloc instead of GC
    ubyte* data_nullterminated = cast(ubyte*) malloc(data.length + 1);
    scope(exit) free(data_nullterminated);
    memcpy(data_nullterminated, data.ptr, data.length);
    data_nullterminated[data.length] = 0;

    scope lexer = new Lexer("test", cast(char*) data_nullterminated,
        0, data.length, false, false);

    do {
        sum += lexer.token.value;
    }
    while (lexer.nextToken != TOKeof);

    return 0;
}
```

The compiler commandline is long because it has to include the dmd lexer files too, but here it is for completeness. For maximum debug information I added `-disable-fp-elim` and `-link-debuglib` but it's not really needed.

```
ldc2 -fsanitize=fuzzer,address -disable-fp-elim fuzzlexer.d -of=fuzzlexer -g -O3 -link-debuglib -Jdmd -Jdmd/src/ddmd -Idmd/src dmd/src/ddmd/root/array.d dmd/src/ddmd/root/ctfloat.d dmd/src/ddmd/root/file.d dmd/src/ddmd/root/filename.d dmd/src/ddmd/root/hash.d dmd/src/ddmd/root/outbuffer.d dmd/src/ddmd/root/port.d dmd/src/ddmd/root/rmem.d dmd/src/ddmd/root/rootobject.d dmd/src/ddmd/root/stringtable.d dmd/src/ddmd/console.d  dmd/src/ddmd/entity.d  dmd/src/ddmd/errors.d  dmd/src/ddmd/globals.d  dmd/src/ddmd/id.d  dmd/src/ddmd/identifier.d  dmd/src/ddmd/lexer.d  dmd/src/ddmd/tokens.d  dmd/src/ddmd/utf.d
```

My expectation was that there would be a bug deep down with some rare cornercase of a cornercase, and that the fuzzer would have to run for hours and hours covering more and more of the lexer to, perhaps, finally find a bug. So I was preparing for a long fuzzing session, wanted to go to bed and kick this one off but... The fuzzer found a first failure within a second.

Running the fuzzer several times, sometimes it still hadn't found a bug after 15 seconds, but usually it found the bug within a second; the process is (semi-)random so that's expected.

Let's look at the output. I'm running the fuzz target with [`-close_fd_mask=3`](https://llvm.org/docs/LibFuzzer.html#options) to close stdout and stderr to silence the output from the lexer when it reports about strange invalid input (those are not bugs!).

```
> ./fuzzlexer -close_fd_mask=3
...
#18 NEW    cov: 272 ft: 348 corp: 11/6819b exec/s: 0 rss: 34Mb L: 3 MS: 1 InsertByte-
=================================================================
==13181==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x602000000f14 at pc 0x000103318050 bp 0x7ffeec901250 sp 0x7ffeec901248
READ of size 1 at 0x602000000f14 thread T0
    #0 0x10331804f in _D4ddmd5lexer5Lexer4scanMFPS4ddmd6tokens5TokenZv lexer.d:1047
    #1 0x10338760b in _D9fuzzlexer22LLVMFuzzerTestOneInputFxAhZi lexer.d:234
    #2 0x1033878ba in LLVMFuzzerTestOneInput fuzzlexer.d:78
...
0x602000000f14 is located 0 bytes to the right of 4-byte region [0x602000000f10,0x602000000f14)
allocated by thread T0 here:
    #0 0x1051ce5fc in wrap_malloc (libldc_rt.asan_osx_dynamic.dylib:x86_64h+0x575fc)
    #1 0x1033873f8 in _D9fuzzlexer22LLVMFuzzerTestOneInputFxAhZi fuzzlexer.d:42
    #2 0x1033878ba in LLVMFuzzerTestOneInput fuzzlexer.d:78
...
SUMMARY: libFuzzer: deadly signal
MS: 3 InsertByte-EraseBytes-InsertByte-; base unit: a59478b0623bc537405253419b0b0d5c9430effa
0xbe,0x60,0xae,
\xbe`\xae
artifact_prefix='./'; Test unit written to ./crash-7490356393f3fad31d4f0d3f5bbb6069342a02ed
```

ASan has found a heap-buffer-overflow inside the lexer when the input is [0xbe,0x60,0xae]. The crash location is reported as [lexer.d:1047](https://github.com/dlang/dmd/blob/d822c9a18af7c7f7e71ae039f44373d028f2bfce/src/ddmd/lexer.d#L1047), which is a `continue` statement. Due to a [bug in LDC's debuginfo for switches](https://github.com/ldc-developers/ldc/issues/2255), the crash location is reported wrong and the actual crash line is [the switch condition evaluation](https://github.com/dlang/dmd/blob/d822c9a18af7c7f7e71ae039f44373d028f2bfce/src/ddmd/lexer.d#L271).  
The memory overread happened because of a bug with unpaired string quotes. The first null-terminator of the buffer would finalize lexing the string token but would not terminate the overall lexing, and then the next character is read (beyond the null-terminator, i.e. beyond the buffer). The same happened for a few other cases (found by further fuzzing after a temporary fix), but the bug [is now fixed](https://github.com/dlang/dmd/pull/7050). Afaik, D fuzzing's first success!

Let's look at another piece of code that needs to deal with possibly unchecked user data: Phobos's `std.xml` library.

# Fuzzing std.xml

```d
// File fuzzxml.d
static import std.xml;

import ldc.attributes : optStrategy;
@optStrategy("none")
int LLVMFuzzerTestOneInput(const(ubyte[]) data)
{
    if (data.length < 1)
        return 0;

    try {
        // Note: last element is not necessarily `null`.
        string str = cast(string)data;
        // Check for well-formedness (throws)
        std.xml.check(str);
        // Make a DOM tree
        scope auto doc = new std.xml.Document(str);
    }
    catch (Throwable e) {
        // `Throwable`s thrown are not bugs (in contrast to `Errors`).
    }

    return 0;
}
```

In this case, I'm using `@(ldc.attributes.optStrategy("none"))` to disable dead code elimination in the function (note that `doc` is not used, so the compiler optimizer may remove it).

Let's compile with the fuzzer and ASan enabled, and run the fuzzer. The output is:


```
> ldc2 -fsanitize=fuzzer,address -disable-fp-elim ../fuzzxml.d -of=fuzzxml -O3 -g -link-debuglib
> ./fuzzxml
INFO: Seed: 1352319858
INFO: Loaded 1 modules (24 guards): [0x10f59d690, 0x10f59d6f0),
INFO: -max_len is not provided; libFuzzer will not generate inputs larger than 4096 bytes
INFO: A corpus is not provided, starting from an empty corpus
#0  READ units: 1
#2  INITED cov: 15 ft: 8 corp: 1/1b exec/s: 0 rss: 32Mb
#16384  pulse  cov: 15 ft: 8 corp: 1/1b exec/s: 8192 rss: 58Mb
#32768  pulse  cov: 15 ft: 8 corp: 1/1b exec/s: 8192 rss: 64Mb
#65536  pulse  cov: 15 ft: 8 corp: 1/1b exec/s: 7281 rss: 75Mb
#131072 pulse  cov: 15 ft: 8 corp: 1/1b exec/s: 6898 rss: 99Mb
```

Progress is slow (the corpus doesn't grow) and the output says `Loaded 1 modules (24 guards)`; hmm, only 24 guards? "Guards" are the coverage guards that track the execution flow and guide the fuzzer. 24 guards is _very_ low: the Lexer program had 7567 guards. It's not very likely that the `std.xml` code is written completely without branches. Indeed, we forgot something: the standard library was not compiled with coverage instrumentation enabled. The binary code for `std.xml.check` and `std.xml.Document` is inside the standard library library (nota bene).[^templ] Thus, we have to build the standard library with instrumentation enabled.  
Fortunately, LDC ships with the [`ldc-build-runtime` tool](https://wiki.dlang.org/Building_LDC_runtime_libraries) which makes it easy to rebuild the standard library with custom commandline flags. So let's build the runtime with `-fsanitize=fuzzer,address`;  as explained [in my recent ASan article]({% post_url 2017-12-25-LDC-and-AddressSanitizer %}), we need to specify an `asan_blacklist.txt`[^blacklist] for the standard library.
Now when we link our program with this instrumented standard library, we get the following fuzz output:

[^templ]: This is different for templated code, which is instantiated and codegenned inside the module where that code is called.

[^blacklist]: The ASan blacklist used:
    ```
    # File: asan_blacklist.txt
    fun:_D*callWithStackShell*
    fun:_D*getcacheinfoCPUID2*
    fun:*12conservative2gc*
    fun:_D4core5cpuid*
    fun:_D4core*
    fun:_D4rt*
    fun:_D6object*
    ```

```
> ldc-build-runtime --dFlags='-fsanitize=address;-fsanitize-blacklist=asan_blacklist.txt' BUILD_SHARED_LIBS=OFF
...
Runtime libraries built successfully into: ./ldc-build-runtime.tmp
> ldc2 -fsanitize=fuzzer,address -disable-fp-elim ../fuzzxml.d -of=fuzzxml -O3 -g -link-debuglib -L-Lldc-build-runtime.tmp/lib
> ./fuzzxml
...
INFO: Loaded 1 modules (48686 guards): [0x101cafa20, 0x101cdf2d8), 
INFO: -max_len is not provided; libFuzzer will not generate inputs larger than 4096 bytes
INFO: A corpus is not provided, starting from an empty corpus
#0  READ units: 1
#2  INITED cov: 1313 ft: 469 corp: 1/1b exec/s: 0 rss: 35Mb
#3  NEW    cov: 1313 ft: 470 corp: 2/2b exec/s: 0 rss: 35Mb L: 1 MS: 1 ShuffleBytes-
#4  NEW    cov: 1432 ft: 935 corp: 3/3b exec/s: 0 rss: 35Mb L: 1 MS: 2 ShuffleBytes-ChangeBinInt-
#5  NEW    cov: 1436 ft: 952 corp: 4/4099b exec/s: 0 rss: 35Mb L: 4096 MS: 3 ShuffleBytes-ChangeBinInt-CrossOver-
#8  NEW    cov: 1437 ft: 970 corp: 5/4101b exec/s: 0 rss: 35Mb L: 2 MS: 1 CrossOver-
...
```

Great, we now have many more guards that the fuzzer can use to direct the fuzzing. Also, we get ASan-enabled detection of bugs in (our use of) the standard library, so we can detect more kinds of bugs in `std.xml`.
Sorry to disappoint, but I haven't found bugs in `std.xml` while fuzzing! :-)

# The corpus

The fuzzer is creating a "corpus" of test inputs while it's running. Each corpus entry is a test input that results in a different execution flow in the program. Thus, the corpus forms an automatically generated collection of test cases with high total code coverage. Instead of restarting the fuzzer from scratch every time tests are run, it may be better to start the fuzzer with the collected corpus of the previous fuzz run. Passing a directory to the fuzz executable will read the corpus from that directory and will write new corpus entries there.  
See the ["Seed corpus" section of the libFuzzer tutorial](https://github.com/google/fuzzer-test-suite/blob/master/tutorial/libFuzzerTutorial.md#seed-corpus) for more information.

I let the `fuzzxml` program run for 6.5 minutes, which resulted in a corpus with 383 entries. One of the last entries looked like this:

```
<mmwx5lwwwwwwwwwwwwwwwx5lwwwwwwww-5lwwwwwwwwwwwwwwwx5lwwwwwwww-->
wwwwwowwwwwwwwwwwwwwwwx2lwwwwwwww-->
wwwwwowwwwwwwwx5lwwwwwwww-5lwwwwwwwwwwwwwwwx5lwwwwwwww-->
wwwwwowwwwwwwwwwwwwwwww=ww&#wwwo
```

We can see the early stages of the fuzzer "discovering" what XML is supposed to look like. But we can help the fuzzer a little to discover XML faster.

# Fuzzing deeper layers

Fuzzing with random data only gets you so far. To reach the deeper parts of your program, the input needs to be such that it passes beyond validity checks. We can use a [dictionary file](https://llvm.org/docs/LibFuzzer.html#dictionaries), to generate input that contains common keywords for the program we are testing. For our XML fuzz target, we can use [a dictionary from the AFL git repository](https://github.com/mirrorer/afl/blob/master/dictionaries/xml.dict), because [the syntax of dictionary files is shared between libFuzzer and AFL](https://github.com/google/fuzzer-test-suite/blob/master/tutorial/libFuzzerTutorial.md#dictionaries).
A snippet from the dictionary:

```
tag_open="<a>"
tag_open_close="<a />"
tag_open_exclamation="<!"
tag_open_q="<?"
tag_sq2_close="]]>"
tag_xml_q="<?xml?>"
```

With that dictionary, we let the fuzzer run for 6.5 minutes again (`fuzzxml corpus -dict=xml.dict`). In this case, we get a corpus with 752 entries; a few of the last entries (not sequential):

```
<a version="1" versioon=""/>
```

```
<a versiof="1" versioon=""/>
```

```
<a version="1" version="1"/>
```


For testing the layers of the compiler beyond the lexing and parsing, we need fuzz input that looks like valid code. But with just a dictionary for D code, the mutations performed by libFuzzer are very unlikely to result in valid D code. We need to add more knowledge about D to the fuzz target. There is a fuzz program for testing D compilers, called [Defuzzed](https://github.com/Ace17/defuzzed), but it doesn't take advantage of coverage-guided fuzzing. We need to write a smart fuzz target that takes the fuzzer input and morphs it in ways that result in valid (or almost-valid) D code. We then feed that code string to the compiler which needs to be compiled with instrumentation and linked into our fuzz target. That is: the compiler needs to be usable "as a library". Fortunately, the D compiler front-end code has been moving towards the "compiler as a library" goal. I hope this article inspires someone to work on the D equivalents of [`clang-fuzzer` and `clang-proto-fuzzer`](https://github.com/llvm-mirror/clang/tree/master/tools/clang-fuzzer)!


# Ideas for improvement

Fuzzing with AFL should already be possible with `-fsanitize-coverage=trace-pc-guard`, but I haven't tested it. For faster fuzzing with AFL, we could add plug-in capability to LDC similar to Clang, such that people can try to use the [AFL plugin](https://github.com/mirrorer/afl/tree/master/llvm_mode).

We can improve user-friendliness, by allowing the user to override a D-style function (slice as function parameter) instead of the C function. This should also initialize the D runtime and deal with uncaught exceptions. Basically, we should [add the boilerplate from `example2.d` to Phobos or druntime](https://github.com/ldc-developers/ldc/issues/2501) (dead-stripping by the linker can remove the extra code when unused).

Meanwhile, we can of course expect libFuzzer to become better and better. There are some [cool ideas](https://guidovranken.wordpress.com/2017/07/08/libfuzzer-gv-new-techniques-for-dramatically-faster-fuzzing/) out there.

# Feedback

Constructive feedback, positive and negative, is always appreciated. The best way to get in touch with me and the other LDC developers is via the [digitalmars.D.ldc forum](https://forum.dlang.org/group/digitalmars.D.ldc) and our [Gitter chat](https://gitter.im/ldc-developers/main).

# Footnotes
