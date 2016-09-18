---
layout: post
title:  "LDC: Speed up incremental builds with object file caching"
categories: LDC
---

_To speed up total compile time, LDC 1.1.0 can create an object file cache ([#1572](https://github.com/ldc-developers/ldc/pull/1572)). When cache lookup succeeds in a second compilation pass, optimization and machine codegen can be skipped, reducing the compile time significantly. Tested on a large codebase, the total build time is reduced from ~3m30 (empty cache) to ~2m30 (no changes, all cache hits). All that's needed is passing `-ir2obj-cache=/tmp/ldccache` to LDC, and... read the end to find out about an important detail._

# Parallel builds

Compiling a D program is _fast_. It's one of its selling points. The first time I compiled [DMD](https://wiki.dlang.org/DMD) + [druntime](https://github.com/dlang/druntime) + [Phobos](https://github.com/dlang/phobos) (the whole compiler and the standard library), it was so quick that I thought it had failed somewhere! Compiling with DMD is that fast. [LDC](https://wiki.dlang.org/LDC) is not quite as fast, but its codegen is massively better than DMD's. Better optimization comes at a cost. Especially for optimized builds (`-O3`), optimization and machine codegen take the lion's share of the total LDC execution time.

D compilation is so fast that you can often suffice with compiling everything at once. But for larger projects, splitting the build process in multiple parallel threads is a good way to get faster builds. Your project is compiled to a number of object files, which are linked together into an executable at the end of the build process. I hope this is familiar territory.

For such a parallel build process, changing a bit of code in a file that is imported by many others and restarting the build (an incremental build) will lead to a large recompile. All files that import the modified file will need to be recompiled, because the build system only sees that an imported file has changed, but doesn't know whether the change was only a 'local' change (e.g. changing a `1` to a `2` in a non-template function) or a non-local change (e.g. changing a template) that will affect the module that imports the file. If the change was local, a bunch of files will be recompiled needlessly, because the resulting object files are the same as before. We are going to use that to speed-up our incremental builds.

The optimization is this: as soon as we know that the output will be exactly the same as the previously output object file, we skip further compilation and simply use the previous object file. A little more general: as soon as we know that the output will be the same as an earlier output, we abort and use that earlier output.

(The optimization also speeds up non-parallel non-`singleobj` builds, because each module will be compiled to a separate object file.)

# Compilation steps and cache lookup

These are LDC's compile steps:

1. Parse the input files -> modules;
2. Do semantic analysis for all modules;
3. For each module do:
   a. Generate [LLVM IR](http://llvm.org/docs/LangRef.html) (IR codegen);
   b. Run LLVM optimization passes;
   c. Output object file (LLVM machine codegen);
4. Optionally: link all output object files into an executable.

At which point do we know enough to tell whether the output is going to be the same as before? Right after semantic analysis. After semantic analysis the compiler knows enough to identify what the output for each module is going to be. We must calculate a 'hash' that (almost) uniquely identifies the object file output. Upon recompilation, we recalculate the hash and check whether we've seen that hash before. When we recognize the hash, we use the previously generated object file and skip steps 3b and 3c.

The 'output' of semantic analysis (a decorated [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree)) is not well-defined nor easily serialized, so it is hard to calculate a good hash at that point. However, the generated LLVM IR (step 3a) is very well-defined and we can use the LLVM IR serialization to calculate the hash. The generated LLVM IR is _all_ that is needed (plus compile flags like `-O3`) to create the object file, and using it for the hash prevents bugs related to e.g. changes in the AST.

The new compile steps are ([#1572](https://github.com/ldc-developers/ldc/pull/1572)):

1. Parse the input files -> modules;
2. Do semantic analysis for all modules;
3. For each module do:
   a. Generate LLVM IR (IR codegen);
   b. Calculate IR hash and do cache lookup. Upon cache hit: use cached object file and continue with next module.
   c. Run LLVM optimization passes;
   d. Output object file (LLVM machine codegen);
   e. Store the object file in the cache using the calculated hash from step b as key.
4. Optionally: link all output object files into an executable.


# A little more about the hash calculation

The hash is calculated using LLVM's MD5 hasher. The LLVM binary bitcode representation is fed directly into the hasher. Other hash inputs are the compiler version, the LLVM version, and commandline flags that alter the object file output such as the optimization level and the target CPU. Commandline flags like `-g` or `-d-version=Demo` are not used for the hash; such flags have no effect beyond the LLVM IR generation in step 3a and are therefore already part of the hash (so to speak).

You can use the same cache for all your projects; and that you can share the cache with multiple users. I challenge you to find a hash collision of two LDC invocations with different object file outputs.

# Code changes -> hash changes

However, finding a hash _difference_ for two LDC invocations with _the same_ object file output is not hard.
This D code does the trick:

```cpp
version (FIRST)
{
    int foo(int a)
    {
        return a + 2;
    }
}
else
{
    int foo(int differentname)
    {
        return differentname + 2;
    }
}
```

Compilation with and without `-d-version=FIRST` will result in the same object file output, but with different hashes.

And then, very recently, [David](https://github.com/klickverbot) found out about an LLVM setting that is [new in LLVM 3.9](https://reviews.llvm.org/D17946): `setDiscardValueNames` ([#1749](https://github.com/ldc-developers/ldc/issues/1749)). What this does is discard the names of local variables in the LLVM IR. We figured this would improve compile times because some string concatenations in the compiler could be skipped, or because the generated bitcode would be smaller and hence the hash would be calculated faster ([#1759](https://github.com/ldc-developers/ldc/pull/1759)). But it does something better: it means that the code above will result in the same hash ([#1767](https://github.com/ldc-developers/ldc/pull/1767)). Parameters `a` and `differentname` are local variables and they can be discarded exactly because they do not affect the object file output. Using the new LLVM setting, the names are discarded _in the generated LLVM IR_, and the calculated hashes will be the same: your cached build will be fast under (local) variable name changes!

So now, I again challenge you: find two pieces of code that result in identical object files but have a different hash.

Oh wait...

```cpp
version (FIRST)
{
    void foo();
    void bar();
}
else
{
    void bar();
    void foo();
}
```

Great, another opportunity for improvement! ;-)

# How to use it?

All you need to do to benefit from object file caching is pass the `-ir2obj-cache` commandline flag to LDC: `ldc2 -ir2obj-cache=/tmp/ldccache`. The cache files (`ircache_<MD5 hash>.o`) will be stored in `/tmp/ldccache`.

# What's the gain?

The direct motivation for implementing object file caching was to improve the build times for [Weka.io](http://www.weka.io). It's a large codebase, so building it takes a lot of time. Ahem, 3.5 minute. But faster is always better! The result with cache hits for all object files is..., 2.5 minute! Even though this is the maximum speed-up obtainable, that's a 30% reduction in build time! These time measurements are for development builds with `-g -O0`. If your project needs `-O3` during development, you can expect large build time reductions with caching enabled. Weka's codebase with `-g -O3 -release` goes from 5m50 to... 2m30; the optimization `-O3` is completely skipped for cache hits and the IR codegen time _reduction_ by `-release` is insignificant. The build time with maximum cache hits will be the same for `-g` and `-g -O3 -release -flag-of-the-awesome-optimization-over-9000`.

The only downside I see to using caching is that the hash calculation takes time and it cannot be skipped, not even when the cache is empty and there is no chance to get a cache hit. To store the object in the cache in step 3e, we must calculate the hash at some point. For a very large compiled module, say an object file of a few hundred MBs (Weka has a number of these), the hash calculation time is a few seconds. So 'cold' builds will be a few seconds slower.

# Disk space

All was good. Weka's (re-)builds became 30% faster, with pretty much no downside. I was happy, and started thinking about how to cut codegen of the 100k+ template instantiations that are only used for CTFE (I think)...

And then, weird build servers errors started occurring. Out of disk space. Root cause: LDC's object file cache for some users had grown to more than 60 GB! Oops.

Obviously, a cache expiry/pruning scheme is needed. Cache pruning can be done by LDC ([#1753](https://github.com/ldc-developers/ldc/pull/1753)), using `--ir2obj-cache-prune`, and is also accessible through the stand-alone tool `ldc-prune-cache`. The pruning will take the last access time[^accesstime] of cached files into account and will (try to) leave files that have been accessed recently untouched. The default minimum pruning interval is 20 minutes, meaning that if you try to prune within 20 minutes of the previous prune, simply nothing will happen. This is done so you can simly pass `--ir2obj-cache-prune` to LDC, without the need to worry about multiple compile threads pruning concurrently. Please read about the details of configuring the pruning parameters in the [Github PR #1753](https://github.com/ldc-developers/ldc/pull/1753), or run `ldc-prune-cache --help`.

[^accesstime]: (18 Sep 2016) Because a file's last access timestamp is unreliable (e.g. it's [disabled per default on Windows 7](http://www.groovypost.com/howto/microsoft/enable-last-access-time-stamp-to-files-folder-windows-7/), and it's disabled for [`noatime` partitions](https://www.howtoforge.com/reducing-disk-io-by-mounting-partitions-with-noatime)), LDC [touches a cached file upon a cache hit](https://github.com/ldc-developers/ldc/blob/df4cd3fc20a916d449fe5737cfb6d19733d906d9/driver/ir2obj_cache.cpp#L218-L240).

For parallel builds, I recommend using `ldc-prune-cache` at the end of the build process, instead of simply adding `--ir2obj-cache-prune` to the D compile flags which (most likely) results in a cache prune after compilation of the first file finishes (depending on whether the prune interval has passed). It will probably not make a big difference, but I like to think that I can still get a cache hit on an old cached file during a build. Pruning at the start of the build cycle may delete a cached file that could have been used later on in the build cycle. Basically, I would prune at the start of a long period of no cache usage (e.g. testing the program and changing some code), instead of at the start of a period with active cache usage (building).

# Feedback

Constructive feedback, positive and negative, is always appreciated. And needed: while writing this post, I re-read [my pruning D code](https://github.com/ldc-developers/ldc/blob/master/driver/ir2obj_cache_pruning.d) and discovered two bugs ([#1770](https://github.com/ldc-developers/ldc/pull/1770)).
The best way to get in touch with me and the other LDC developers is either via the [digitalmars.D.ldc forum](https://forum.dlang.org/group/digitalmars.D.ldc) or our [Gitter chat](https://gitter.im/ldc-developers/main).

# Notes
