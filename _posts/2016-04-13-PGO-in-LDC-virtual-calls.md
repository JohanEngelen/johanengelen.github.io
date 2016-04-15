---
layout: post
title:  "PGO: Optimizing D's virtual function calls"
categories: LDC
---

Part 1 on profile-guided optimizations (PGO) in [LDC](https://wiki.dlang.org/LDC).
An article about optimization of virtual (class) function calls using profile data by transforming indirect calls to direct calls, with a description of how this is implemented in LDC using [LLVM](http://llvm.org/).

_Edit (15 Apr 2016)_: The D code of LDC performs 7% faster (on the testcase) when compiled with PGO. I added "function" to the title so my father no longer thinks this is about video phone calls.

*[PGO]: Profile-Guided Optimization
*[ICP]: Indirect Call Promotion
*[VCP]: Virtual Call Promotion

# Profile-Guided Optimization (PGO)
In this article I'll discuss the implementation of optimizations that the compiler can do when it is given details of a typical execution flow of your program. These details are the "profile" of your program: how many times is this function called, how often was a control-flow branch taken, how often is function `A` called by function `B`, what are likely values of variable `X`, how much memory is usually allocated for something, etc.  
A profile can be obtained through different means, but here I will only discuss _instrumented_ profiling. The compiler adds 'instrumentation' code in a first compilation pass. You then run your program with input that is representative of the use case for which you want to optimize the code, and upon exit the program outputs the profile data in a separate file. This profile data file is  used by the compiler in a second compilation pass to compile and optimize your program. You do not have to generate a new profile very often. Profile data can be reused when recompiling a program with small changes; LDC will detect changes in control flow and will ignore profile data accordingly.

A simple build process using PGO looks like[^pgowiki]:

```
> ldc2 -fprofile-instr-generate=profile.raw yourprogram.d -of=instrumented_program
> ./instrumented_program
> ldc-profdata merge -output=profile.data profile.raw
> ldc2 -fprofile-instr-use=profile.data yourprogram.d -of=optimized_program
```

Instrumentation makes your program significantly slower, which makes instrumented PGO unsuitable for example for programs whose behavior depends on specific timings or interactions with other systems.

PGO is not yet available in released LDC versions, but there is a [PR on Github](https://github.com/ldc-developers/ldc/pull/1219) that adds PGO based on execution counts which I hope can be merged into master soon.
For ease of writing, I've written this article as if these optimizations are already in LDC. For now, they are available in my [`pgo-indirectcall` GH branch](https://github.com/JohanEngelen/ldc/tree/pgo-indirectcall).

[^pgowiki]: PGO in LDC is not very well documented yet. There is only a draft wiki page: [http://wiki.dlang.org/LDC_LLVM_profiling_instrumentation](http://wiki.dlang.org/LDC_LLVM_profiling_instrumentation)




# Direct and indirect calls
A direct call is a call to a function whose address is known and is encoded in the machine instruction code.
An indirect call is a call to some address stored in memory or in a register. An earlier part of the program stored the address of a function in a variable, and the indirect call calls the function pointed to by that variable. Indirect calls are used in many programs: they enable polymorphic objects, are used for dynamic library bindings, select execution of hardware interrupt routines, etc. In D, you probably use them all the time when calling class methods.

What makes indirect calls "slower" than direct calls is the extra indirection: the CPU has to read a piece of memory before it can jump to the function to continue execution. Modern CPUs are marvelous machines and I believe indirect calls are just as fast as direct calls in many cases. But a direct call can be _inlined_ during compilation and allows analysis of what is happening inside the called function and what is returned by it. This should help in generating faster executables. [At the moment](http://reviews.llvm.org/D17864) LLVM developers are adding an "Indirect Call Promotion" pass that transforms an indirect call into a branch + direct call using profiling information, and of course LDC should take advantage of it.

# Indirect Call Promotion (ICP)

The presentation that got me started on this topic is ["Profile-based Indirect Call Promotion" by Ivan Baev](http://llvm.org/devmtg/2015-10/slides/Baev-IndirectCallPromotion.pdf). The ICP optimization is transforming this code fragment

```cpp
// An unknown function address is loaded into fptr, ...
void function() fptr = get_function_ptr();
// ... and is called
fptr();
```

to this:

```cpp
void function() fptr = get_function_ptr();
// We check if the address is the address of void likely_function().
if (fptr == &likely_function)
    likely_function(); // Yes! Call likely_function() /directly/
else
    fptr(); // Nope, do the indirect call.
```

Note that you can manually transform code to a functionpointer comparison + direct call and you will get the same machine code _without_ the need for profiling. If you have a good hunch about what the function pointer could point to, you can rewrite your code like this and you'll perhaps get a performance improvement. With a helper template function it's easy to do so:

```cpp
auto is_likely(alias Likely, Fptr, Args...)(Fptr fptr, Args args) {
    return (fptr == &Likely) ? Likely(args) : fptr(args);
}
// ...
void function() fptr = get_function_ptr();
fptr.is_likely!likely_function();
```

I recommend taking a look at the generated LLVM IR (`ldc2 -output-ll ...`) and see for yourself what difference this makes with optimizations on (`-O3`) and with an inlinable `likely_function` function.

# Indirect Call profiling

In the compile pass that adds instrumentation (`ldc2 -fprofile-instr-generate -fprofile-indirect-calls`), code like this

```cpp
void somefunction() {
    fptr(); // typeof(fptr) = void function()
}
```

is transformed to (somewhat simplified)

```cpp
void somefunction() {
    increment_counter(&__profc_somefunction, 0);
    __llvm_profile_instrument_target(fptr, &__profd_somefunction, 0);
    fptr();
}
```

Instrumentation works on a per-function basis: each instrumented function gains a set of data structures that store profiling information during execution. These data structures are automatically added to the code by LLVM when it detects the presence of LLVM profiling intrinsic function calls. The `__profc_...` data structure is an array storing execution counts; the second parameter for `increment_counter` is the counter index. For the rest of this article, I will exclude the execution counters; I plan to write a second article about execution counting PGO in LDC in the near future. Before exiting an instrumented program, the linked-in profiling runtime library dumps these instrumentation data structures in a profile file (in raw format).

The `__profd_...` data structure is of primary interest here: it stores "value" profile data. In this case, the value is the raw pointer value stored in `fptr`. `__llvm_profile_instrument_target` sends the `fptr` value as a `IPVK_IndirectCallTarget` value type to value-profiling slot `0`. LLVM's profiling machinery then takes this value and eventually presents the compiler with a list of values and how often they occured for each value-profiling slot. This is done by the [profiling runtime library](https://llvm.org/svn/llvm-project/compiler-rt/trunk/lib/profile/) of [compiler-rt](http://compiler-rt.llvm.org/), the profile processing tool [ldc-profdata](http://llvm.org/docs/CommandGuide/llvm-profdata.html)[^profdata], and the [`llvm::InstrProfReader` class](http://llvm.org/docs/doxygen/html/classllvm_1_1InstrProfReader.html). But, how can you use function address values when they [can be different](https://en.wikipedia.org/wiki/Position-independent_code) for each program run?

[^profdata]: LDC builds include a renamed `llvm-profdata` (to `ldc-profdata`) to ensure that the version used corresponds exactly to the LLVM version with which LDC was built.

The `__profd_...` data structure contains more information than just the value profile data. Among other things, it contains a reference (a hash) to the function name and it contains the function address (aha!). `ldc-profdata` reads all `__profd_` data structures and translates address values to function name hashes (for the `IPVK_IndirectCallTarget` value type) if it can find the pointer value in a `__profd_` structure. This mechanism determines that ICP is _only_ possible on function call targets whose address+hash are stored in `__profd_` structures. Currently, the only way to have LLVM generate (correct) `__profd_` structures for function `X` is to add LLVM profiling intrinsic function calls _inside_ function `X`. If your function pointer is pointing to `printf` a lot, ICP can not do anything (yet?). More on this later.

Here is a snippet from the output of `ldc-profdata show` for a profile obtained for LDC itself, showing the profile data for the `extern(C++)` function [`checkAccess(AggregateDeclaration*, Loc, Scope*, Dsymbol*)`](https://github.com/ldc-developers/ldc/blob/v1.0.0-alpha1/ddmd/access.d#L135):

```
_Z11checkAccessP20AggregateDeclaration3LocP5ScopeP7Dsymbol:
    <snip>
    Indirect Call Site Count: 3
    Indirect Target Results:
    [ <call site>, <symbol name>, <count> ]
    [ 0, _ZN20AggregateDeclaration22isAggregateDeclarationEv, 196415 ]
    [ 0, _ZN7Dsymbol22isAggregateDeclarationEv, 13102 ]
    [ 1, _ZN11Declaration4protEv, 195211 ]
    [ 2, _ZN7Dsymbol7toCharsEv, 20 ]
```

Three indirect calls ([0](https://github.com/ldc-developers/ldc/blob/v1.0.0-alpha1/ddmd/access.d#L144), [1](https://github.com/ldc-developers/ldc/blob/v1.0.0-alpha1/ddmd/access.d#L158), [2](https://github.com/ldc-developers/ldc/blob/v1.0.0-alpha1/ddmd/access.d#L191)) are profiled for this function. At [call site 0](https://github.com/ldc-developers/ldc/blob/v1.0.0-alpha1/ddmd/access.d#L144), two function pointer values were registered; 94% of the time it was a call to [`AggregateDeclaration::isAggregateDeclaration()`](https://github.com/ldc-developers/ldc/blob/v1.0.0-alpha1/ddmd/aggregate.d#L700) that screams "inline me!" with ICP. For the other two call sites, only one function pointer value was registered. ICP at [call site 1](https://github.com/ldc-developers/ldc/blob/v1.0.0-alpha1/ddmd/access.d#L158) will definitely also inline [`Declaration::prot()`](https://github.com/ldc-developers/ldc/blob/v1.0.0-alpha1/ddmd/declaration.d#L377).

Like the three indirect calls in `checkAccess(AggregateDeclaration*, Loc, Scope*, Dsymbol*)`, most indirect calls in D are special: they are [_virtual_ function](https://dlang.org/spec/function.html#virtual-functions) calls. Enter "Virtual Call Promotion".


# Virtual Call Promotion (VCP)

What I've been working on is going one step further than ICP in LDC: promote a virtual call to a direct call. In addition to profiling the function call address, we can also profile the object's [vtable](https://en.wikipedia.org/wiki/Virtual_method_table) address to promote vtable lookups to direct function calls. This eliminates _two_ indirections: object to vtable to function pointer. Consider the following code:

```cpp
class A {
    int foo(int a) {
        return a * 2;
    }
};
// ...
void somefunction(A a) {
    // ...
    auto b = a.foo(2);
    // ...
}
```

Because `somefunction` must work for `a`'s of type `A` and for `a`'s of types derived from `A`, the virtual call to `foo` is 'lowered' by the compiler into this pseudo code:

```cpp
void somefunction(A a) {
    // ...
    auto vtable = a.__vptr; // (a is actually a pointer: 0th indirection)
    int function(A, int) fptr = vtable[index of A.foo]; // vtable lookup (1st indirection)
    auto b = fptr(a, 2); // indirect call (2nd indirection)
    // ...
}
```

When the profile data indicates that `somefunction` is usually called with an object of exactly type `B` (derived from `A`) we can optimize the code to this[^mutabletypeid]:

```cpp
void somefunction(A a) {
    // ...
    auto b =
        (cast(void*)a.__vptr == cast(void*)typeid(B).vtbl.ptr)
            ? b = a.B.foo(2) // a direct call
            : b = a.foo(2);  // the original indirect call with vtable lookup
    // ...
}
```

The code is transformed into a comparison of object `a`'s vtable pointer with the vtable pointer for objects of type `B`: if true, do a direct call; otherwise, do the original virtual call. The `a.B.foo(2)` is [D syntax](https://dlang.org/spec/function.html#virtual-functions) for explicitly calling the `foo` function of class `B`, regardless of whether `a` is of type `B`.  
(Unfortunately, `cast(void*)typeof(B).vtbl.ptr` is not a constant and the D code isn't quite what LDC will generate[^mutabletypeid].)

(For comparison, GCC has an option to do [speculative devirtualization](http://hubicka.blogspot.nl/2014/02/devirtualization-in-c-part-4-analyzing.html), _without_ profile data.)

[^mutabletypeid]: The D code is not exactly what LDC will generate with PGO. LDC generates code that does a comparison of `a.__vptr` with the (link-time constant) address of the vtable for class `B`. `cast(void*)typeof(B).vtbl.ptr` is not a (link-time) constant, because `typeid(B)` is [a _mutable_ `TypeInfo` object](https://forum.dlang.org/thread/entjlarqzpfqohvnnwjb@forum.dlang.org). [Until recently](https://github.com/ldc-developers/ldc/pull/1380), LDC _did_ have immutable `TypeInfo` objects for `typeid` expressions, but we [had to make them mutable](https://github.com/ldc-developers/ldc/issues/1377) in order to support `synchronize(typeid(B)){}` statements (`synchronize` may write to the object's `__monitor` and is therefore invalid for immutable objects, even though [DMD accepts it](https://github.com/D-Programming-Language/dmd/pull/5564)). With immutable `TypeInfo`, the code would have been optimized to a comparison with a constant. I have not yet found a way to do a comparison with the vtable pointer as a constant (other than with the non-portable inline assembly `__asm!(void *)("leaq __D" ~ B.mangleof[1..$] ~ "6__vtblZ(%rip), $0", "=r")`). If you find a way, please do tell!

# Coaxing LLVM

The instrumentation of a virtual call is similar to that of an indirect call. The instrumented version (`ldc2 -fprofile-instr-generate -fprofile-virtual-calls`) of the virtual call pseudo-code is:

```cpp
void somefunction(A a) {
    // ...
    auto vtable = a.__vptr; // (a is actually a pointer: 0th indirection)
    __llvm_profile_instrument_target(vtable, &__profd_somefunction, 0);
    int function(A, int) fptr = vtable[index of A.foo]; // vtable lookup (1st indirection)
    auto b = fptr(a, 2); // indirect call (2nd indirection)
    // ...
}
```

Here is a snippet from a profile obtained for LDC itself with VCP enabled, again showing the profile data for the `extern(C++)` function [`checkAccess(AggregateDeclaration*, Loc, Scope*, Dsymbol*)`](https://github.com/ldc-developers/ldc/blob/v1.0.0-alpha1/ddmd/access.d#L135):

```
_Z11checkAccessP20AggregateDeclaration3LocP5ScopeP7Dsymbol:
    <snip>
    Indirect Call Site Count: 3
    Indirect Target Results:
    [ <call site>, <symbol name>, <count> ]
    [ 0, , 181132 ] *
    [ 0, , 13102 ]
    [ 0, , 12316 ]  *
    [ 0, , 2921 ]   *
    [ 0, , 46 ]     *
    [ 1, , 131771 ]
    [ 1, , 53053 ]
    [ 1, , 6526 ]
    [ 1, , 3502 ]
    [ 1, , 296 ]
    [ 1, , 40 ]
    [ 1, , 21 ]
    [ 1, , 2 ]
    [ 2, , 20 ]
```

Why are there no symbol names?  
To get vtable value profiling to work, I have to trick LLVM a little. As mentioned above, the `__profd_` data structures are used to translate raw pointer values to hashes that are given by the compiler (and are constant across program runs). But `__profd_X` is only generated by LLVM when it encounters a call to an LLVM profiling instrinsic function _inside_ function `X`. And, a vtable is not a function! What LDC has to do is manually create `__profd_` data structures for vtables exactly like LLVM does for functions, such that LLVM can do the pointer value to hash lookup. (And a valid `__profd_` data structure also contains a pointer to a `__profc_` array with counter entries, so we also have to generate _that_.)  
LLVM keeps a list of all function names for which it autogenerated `__profd_` and puts them in the executable in a compressed format; `__profd_` contains a function name hash that indexes into that compressed data. Because I generate my own `__profd_` data structures, the symbol names are not in LLVM's list of profiled names and the hash lookup fails. Not a big deal: LDC only needs to know the hashes of the vtable pointers, not the actual names. This really needs a better solution; perhaps an LLVM patch to allow pointer->hash->name lookup for arbitrary symbols, which would also help solve the problem of unavailable ICP for function targets that were not compiled with instrumentation on.

Back to the profile data excerpt. At [call site 0](https://github.com/ldc-developers/ldc/blob/v1.0.0-alpha1/ddmd/access.d#L144), five different vtable pointer values were registered, and 86% of the time it's the first one. A clear opportunity for VCP. Let's look at the generated LLVM [IR](http://llvm.org/docs/LangRef.html)[^IR] at `-O3`:

[^IR]: If you use LDC and want to learn how your code is executed or want to try and optimize your code, it is very rewarding to learn the basics of LLVM IR representation and study the output of `ldc2 -c -output-ll` on pieces of your program. An interesting presentation that starts with an explanation of the basics: [Understanding Compiler Optimization - Chandler Carruth - Opening Keynote Meeting C++ 2015](https://www.youtube.com/watch?v=FnGCDLhaxKU).


```llvm
  %4 = tail call %ddmd.dsymbol.Dsymbol* @_ZN7Dsymbol8toParentEv(%ddmd.dsymbol.Dsymbol* nonnull %smember_arg) ; [#uses = 7]
;...
  %6 = getelementptr inbounds %ddmd.dsymbol.Dsymbol, %ddmd.dsymbol.Dsymbol* %4, i64 0, i32 0
  %7 = load %ddmd.dsymbol.Dsymbol.__vtbl*, %ddmd.dsymbol.Dsymbol.__vtbl** %6, align 8
  %8 = icmp eq %ddmd.dsymbol.Dsymbol.__vtbl* %7, bitcast (%ddmd.dstruct.StructDeclaration.__vtbl* @_D4ddmd7dstruct17StructDeclaration6__vtblZ to %ddmd.dsymbol.Dsymbol.__vtbl*)
  br i1 %8, label %pgo.vtable.true, label %pgo.vtable.false, !prof !185

pgo.vtable.true:
  %9 = bitcast %ddmd.dsymbol.Dsymbol* %4 to %ddmd.aggregate.AggregateDeclaration*
  br label %pgo.vtable.end

pgo.vtable.false:
  %"smemberparent.isAggregateDeclaration@vtbl" = getelementptr inbounds %ddmd.dsymbol.Dsymbol.__vtbl, %ddmd.dsymbol.Dsymbol.__vtbl* %7, i64 0, i32 54
  %10 = load %ddmd.aggregate.AggregateDeclaration* (%ddmd.dsymbol.Dsymbol*)*, %ddmd.aggregate.AggregateDeclaration* (%ddmd.dsymbol.Dsymbol*)** %"smemberparent.isAggregateDeclaration@vtbl", align 8
  %11 = tail call %ddmd.aggregate.AggregateDeclaration* %10(%ddmd.dsymbol.Dsymbol* nonnull %4)
  br label %pgo.vtable.end

pgo.vtable.end:
  %12 = phi %ddmd.aggregate.AggregateDeclaration* [ %9, %pgo.vtable.true ], [ %11, %pgo.vtable.false ]
  %13 = icmp eq %ddmd.aggregate.AggregateDeclaration* %12, null

;...
!185 = !{!"branch_weights", i32 181133, i32 28386}
```

Line by line:  
`%4` stores the value of `smemberparent = smember.toParent();` (direct call);  
`%6` gets a pointer to the location of the vtable pointer `__vptr` of `smemberparent`;  
`%7` loads the vtable pointer;  
`%8` is the result of comparing `%7` with the link-time constant vtable pointer of `StructDeclaration`;  
What follows is a branch on condition `%8`, with extra branch weights metadata `!prof !185` (recognize the numbers (+1)?);  
`pgo.vtable.true:` If true, the call `smemberparent.StructDeclaration.isAggregateDeclaration()` is completely inlined (`return this;`) and only a bitcast remains;  
`pgo.vtable.false:` If false, the call `smemberparent.isAggregateDeclaration()` is executed by indexing into the vtable (`%"smemb...@vtbl"` and `%10`) and calling that function pointer (`%11`);  
The branches merge at `pgo.vtable.end:` where a PHI node selects the return value `%9` or `%10` depending on the execution path taken and stores it in `%12`;  
Finally, `%13` checks whether the return value was `null` (the `!`) and `!smemberparent.isAggregateDeclaration()` is done, _with_ VCP!

Some fun implementation hurdles:

* support VCP of virtual tables (classes) in other modules compiled without instrumentation;
* an LLVM bug in processing profile data with unknown raw pointer values;
* correctly deal with the return values (oh wait, some functions return `void`);
* some calls are normal, others are 'invokes' with exception handling;
* what about interfaces?
* the 'same' function slots in related vtables have a slightly different function type (the `this` pointer changes type and perhaps covariant return types).


# ICP and VCP interaction
Depending on class hierarchy, different vtables may refer to the same function.

```cpp
class B : A {
    override int foo(int a) {
        return a + 1;
    }
}
class C : A {
    void bar() {}
}
void somefunction(A a) {
    // ...
    auto b = a.foo(2);
    // ...
}
```

`A.foo` and `C.foo` point to the same function, but the vtable pointers are different. Depending on the statistics of the object types passed to `somefunction`, you either want ICP of the indirect call after vtable lookup or VCP. When mostly `C'`s are passed, you'd want VCP (assuming that it is faster than ICP, perhaps it really doesn't matter). When it is mostly a mix of `A`'s and `C`'s being passed, you'd want ICP.

For a nice interaction between ICP and VCP, both the indirect call and the vtable lookup should be profiled (`-fprofile-indirect-calls -fprofile-virtual-calls`). I've currently implemented a simple scheme that ignores indirect call profile data when VCP is considered profitable (occurance count > 1000, and 80% or higher chance of occurance for a vtable pointer value). I am certain that smarter heuristics can be found for a better balance between VCP and ICP.

In the profile data given above, the vtable pointer occurances marked with a `*` all resulted in calls to `AggregateDeclaration::isAggregateDeclaration()` (combine the given profile data and do the math). An open question is whether it is better to do VCP (86% hit rate) or ICP (96% hit rate, but with extra vtable lookup "penalty").


> Measuring gives you a leg up on experts who are too good to measure.
> -- Walter Bright

# Performance measurements

The removal of the indirections is nice, but I think the big performance improvement potential is the possibility of inlining and eliminating the function call entirely, possibly leading to other optimizations. It does increase code size and the statistics could be different from the profile data, which both could degrade performance...

I write embarassingly little D and have no project of my own to compile and test to see whether ICP and VCP bring any increase (or decrease) in performance. The best I have is LDC itself: the front-end code is [forked](https://github.com/ldc-developers/ldc/tree/master/ddmd) from [DMD](https://dlang.org/dmd-osx.html) and is written in D. So what I did is compile LDC with and without PGO and see if there is a difference in compiling a "performance" testcase: some unittests ([Weka.io](http://www.weka.io/) codebase) with a lot of template code that take about 65 seconds to compile into an object file. I'm going to cheat too: the profile will be generated with the performance test case, i.e. it will be the best possible PGO profile for the performance test. Also, execution count PGO is turned on, so there's that, too.

|-----------------+-----------------------------------------------|
| LDC version     | `time ldc2 ...`    |
|:--------------|:----------------------------------------------|
| normal          |`55.63s user 6.45s system 97% cpu 1:03.75 total` |
|----
| w/ PGO          |`54.04s user 6.62s system 97% cpu 1:02.26 total` |
|=====

<br> I've simply measured the compilation time using the `time` command, 6 or more times, and here I've listed the fastest time measured.  The error margin is large, but the build with PGO is consistently faster by about 1.5s; i.e. ~2% better performance with PGO. Nothing stunning, but I'm already very happy the performance difference is observable in these _very naive_ tests (and in the right direction!).  
The instrumented versions are approximately two times slower and the raw profile data file size is about 1MB.  
The binaries with PGO are ~0.5% larger than without PGO.

Please don't use these measurement data to build an argument in any discussion. I know they are terribly inaccurate and I have put them here only for illustration. I apologize for not having anything better at the moment.

***Edit (15 Apr 2016)***:
Execution in LDC is a mix of three parts: the D front-end (parsing and semantic analysis), the C++ "middle-end", and the LLVM back-end (C++). In the measurements above, I measured the total execution time to show how much performance gain there is for LDC and because the middle-end also calls into the front-end code. However, since then I've found that a large fraction of the time is spent in C++ code and is thus not changed at all with PGO. Perhaps it is more fair to the PGO work to look at results from D code only. Here are measurements of the same binaries compiling the same code but with the extra option `-o-`, which stops compilation after semantic analysis:

|-----------------+-----------------------------------------------|
| LDC version     | `time ldc2 -o- ...`    |
|:--------------|:----------------------------------------------|
| normal          |`22.63s user 3.56s system 97% cpu 26.888 total` |
|----
| w/ PGO          |`20.78s user 3.57s system 97% cpu 25.081 total` |
|=====

We see that less than half of the time is spent on parsing and semantic analysis. The difference in total execution time is about the same as for the full compilation to object files. But now we see that the D code became about 7% faster with PGO!


# Final thoughts

The [DDMD D source code](https://github.com/D-Programming-Language/dmd/tree/master/src) uses the keyword `final` a lot. `final` helps tremendously in devirtualizing calls: when used [on a class](https://dlang.org/spec/class.html#final) or [on a class method](https://dlang.org/spec/function.html#virtual-functions), it makes it illegal to derive from that class or override that function in a derived class. So it is easy for the compiler to deduce that it can devirtualize a call (and `final` in a base class means that that function will never be a virtual function).

For fun, I've removed a bunch of `final` from the D source to see if it makes LDC slower, and then what PGO does to improve it. Removing _all_ `final`s does not work, because removing final may mean a non-virtual function becomes virtual, risking a mismatch with the C++ headers used for interfacing the C++ and D parts of LDC. But removing `final` from all 292 occurances of `final class` and from all 138 occurances of `override final` still results in a functioning LDC compiler.  
It is fun to browse through the profile data to get a feel for indirect call statistics. For example in `ddmd.parse.Parser.parsePrimaryExp()`, the removal of `final` means going from 4 to 82 indirect calls. Even without deleting `final`, some functions just have many indirect call sites: `BinExp::checkOpAssignTypes(Scope*)` has 84 indirect call sites, `VarDeclaration::semantic(Scope*)` has 138 and was called 887810 times. Don't worry, many of these indirect call sites do not have profile data, i.e. execution never reached those paths. Ah, and `Array<Dsymbol*>::opIndex(unsigned long)` was called 13'759'873'768 times (which does not fit in an `uint`).

|-----------------+-----------------------------------------------|
| LDC version     | `time ldc2 ...`    |
|:--------------|:----------------------------------------------|
| removed `final` |`56.43s user 6.43s system 97% cpu 1:04.40 total` |
|----
| removed `final` w/ PGO  |`54.35s user 6.30s system 97% cpu 1:02.27 total` |
|=====

<br> After the removal of `final`, the compile times went up a little. PGO appears to compensate for the performance loss, with compile times just a little slower than with `final` (again, the error margin is larger than these differences...).

***Edit (15 Apr 2016)***:
See the edit above. Here measurements of just the parsing and semantic analysis (D code only), with `final` removed:

|-----------------+-----------------------------------------------|
| LDC version     | `time ldc2 ...`    |
|:--------------|:----------------------------------------------|
| removed `final` |`23.44s user 3.65s system 97% cpu 27.704 total` |
|----
| removed `final` w/ PGO  |`20.85s user 3.51s system 97% cpu 24.942 total` |
|=====

The results are the same as before, but more dramatic. The PGO build without `final` is as fast as the PGO build with `final`, PGO appears to be able to completely compensate for the loss of devirtualization with `final`. The performance improvement with PGO is 10%!

# Try it yourself

Unfortunately, there are no binaries available (yet?). To test PGO, you will need to build LDC yourself. I have an [open PR](https://github.com/ldc-developers/ldc/pull/1219) that adds execution count instrumentation and PGO to LDC. The ICP and VCP optimizations described in this atricle are built upon that PR and can be found in my [`pgo-indirectcall` GH branch](https://github.com/JohanEngelen/ldc/tree/pgo-indirectcall). ICP is done by LLVM and will only work with LLVM trunk patched with [D17864](http://reviews.llvm.org/D17864); VCP is [done by LDC](https://github.com/JohanEngelen/ldc/blob/pgo-indirectcall/gen/tocall.cpp#L886-L944) and should work without that patch. In all cases, you will need a very recent LLVM development trunk version (r266047 or newer).

I invite you to play with PGO in LDC yourself, although for the moment it is a little tedious to get started. I am curious to hear from your experiences: does performance improve / degrade or is it all the same? For what kind of code do you see differences in performance? Bugs? What PGO ideas do you have? Let me know what you think, for example on the [D forums](http://forum.dlang.org/)!

# Footnotes