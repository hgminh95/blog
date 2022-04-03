---
title: "Which Loop Is Faster?"
date: 2022-03-23T20:54:08+08:00
tags: ["performance", "os", "memory"]
thumbnail: which-loop-is-faster.png
draft: false
---

The first loop accesses the memory sequentially from the beginning to the end of the array. The second one skips N steps between 2 iterations. **Which one is faster?**

<!--more-->

```cpp
int a[N * N];

// Loop #1
for (int i = 0; i < N; ++i)
  for (int j = 0; j < N; ++j)
    sum += a[i * N + j];

// Loop #2
for (int i = 0; i < N; ++i)
  for (int j = 0; j < N; ++j)
    sum += a[j * N + i];
```

I have run it and here is the result (you can find the code on [Github](https://github.com/hgminh95/which_loop_is_faster) - it is not exactly the same but should be close enough for this discussion)

```bash
----------------------------------------------------------------------------
Benchmark                                  Time             CPU   Iterations
----------------------------------------------------------------------------
BM_LoopNo1/16                            119 ns          119 ns      5831556
BM_LoopNo1/64                           1852 ns         1854 ns       377141
BM_LoopNo1/512                        118845 ns       118953 ns         5832
BM_LoopNo1/4096                      9011987 ns      9019854 ns           80
BM_LoopNo1/8192                     36637359 ns     36666974 ns           18
BM_LoopNo2/16                            124 ns          124 ns      5577102
BM_LoopNo2/64                           2011 ns         2012 ns       349585
BM_LoopNo2/512                        851109 ns       851729 ns          810
BM_LoopNo2/4096                    153951705 ns    154055943 ns            4
BM_LoopNo2/8192                    801699400 ns    802239396 ns            1
```

You can clearly see that loop #1 is faster. Now, let's find out why that is the case.

**NOTES**:

- In this blog, I use [Google Benchmark Library](https://github.com/google/benchmark) to run benchmark code and [perf](https://perf.wiki.kernel.org/index.php/Main_Page) to collect performance counters from kernel and CPU. You can check them out if you are not familiar.
- All examples are run under Linux and Intel CPU. Other platforms might show different results.

## Page fault

[Page fault](https://en.wikipedia.org/wiki/Page_fault) happens when we try to access memory that is not in the main memory. The kernel need to delay the program and load the memory from storage like hard drive, which is very expensive. Since loop #2 jumps back and forth between different pages, it could trigger more page fault than #1 if N is big enough.

To check on the number of page faults, you can use below command

```bash
$ perf stat -e page-faults,minor-faults,major-faults <program>
```

Where `minor-faults` is when the page is in memory but is not allocated to the process or not registed to the memory management unit (e.g. when you first access the memory), `major-faults` is when the page is not in main memory (that is very costly operation since we might need to read the memory from disk), and `page-faults` is sum of them. Below is one possible result,

```bash
'./bazel-bin/bm_loop --benchmark_filter=BM_LoopNo1Long':

           785,074      page-faults
            33,743      major-faults
           751,331      minor-faults

'./bazel-bin/bm_loop --benchmark_filter=BM_LoopNo2Long`':

           579,736      page-faults
           157,724      major-faults
           422,012      minor-faults
```

**NOTES**:
- I wrote a [small utility](https://github.com/hgminh95/which_loop_is_faster/blob/master/lock_mem.cpp) to lock abitrary amount of memory in RAM, to simulate the low memory scenario.

## TLB misses

The address refered by the application is virtual address. To access the memory, we actually need to translate them into physical address, by walking through the [page table](https://en.wikipedia.org/wiki/Page_tabl) (depsite the name, it is actually like a tree, hence "walking"). Since page table can be very large (say we have 16GB of RAM, which each page is 4KB, you do the math), we have something called [Translation Lookaside Buffer or TLB](https://en.wikipedia.org/wiki/Translation_lookaside_buffer) to cache the page table. TLB miss happen when we try to access a page which is not in TLB.

Again, going back and forth between different pages won't help in this case either.

To see number of TLB misses, you can run below commands:

```bash
$ perf stat -e dTLB-loads,dTLB-load-misses <program>
```

where `dTLB` is data TLB (there is `iTLB` for instruction), and `load-misses` means number of lookup that miss TLB.

```bash
'./bazel-bin/bm_loop --benchmark_filter=BM_LoopNo1/8192':

         2,494,435      dTLB-load-misses          #    0.06% of all dTLB cache hits 
     4,338,977,389      dTLB-loads

'./bazel-bin/bm_loop --benchmark_filter=BM_LoopNo2/8192':

        49,024,649      dTLB-load-misses          #   25.15% of all dTLB cache hits 
       194,919,514      dTLB-loads
```

## CPU Cache

Fetching memory from RAM is actually very slow, it could cost hundreds of clock cycles. That's why we have [CPU cache](https://en.wikipedia.org/wiki/CPU_cache), which will store the memory inside the CPU, making subsequence accesses faster. There multiple levels of CPU cache (e.g. L1, L2, L3), with increasing latency and capacity.

But since we only access each element once in this example, how can cache help? Turn out we don't load 1 element to the cache at one time, we load a cache line, which usually are 64 bytes. So everytime we access an element, it will load 15 other elements (assuming int is 32-bit) to the cache, speeding up the next 15 memory access if we iterate them in sequential order.

You can check the cache miss number with below commands:

```bash
$ perf stat -e L1-dcache-load-misses,L1-dcache-loads <program>
```

where `L1-dcache-loads` is number of time we try to load data from L1 cache, and `L1-dcache-load-misses` is the number of it fails to do so.


```bash
'./bazel-bin/bm_loop --benchmark_filter=BM_LoopNo1Long':

        36,969,006      L1-dcache-load-misses     #    4.77% of all L1-dcache hits  
       774,500,777      L1-dcache-loads


'./bazel-bin/bm_loop --benchmark_filter=BM_LoopNo2Long':

       781,431,687      L1-dcache-load-misses     #  100.77% of all L1-dcache hits  
       775,442,911      L1-dcache-loads
```

**NOTES**:

- You might notice that the original benchmark code has some flaws. The cache line between runs is not flushed, so it could affect the benchmark result a little bit. In this example, I only run the the code once to make sure it won't affect the result.
- `L1-dcache-load-misses` are bigger than `L1-dcache-modes` in the second examples, which seems quite weird. Turn outs, those cache related parameters can be a bit misleading sometimes. This [blog](https://sites.utexas.edu/jdm4372/2013/07/14/notes-on-the-mystery-of-hardware-cache-performance-counters/) might shed some lights into this.

## Cache Prefetching

You might realize something is off by looking at above number. How can number of cache misses are so small compared to 1:15 ratio? Again, CPU is one step ahead of us. It has something called [prefetcher](https://en.wikipedia.org/wiki/Cache_prefetching) which will try to find out the memory access pattern to predict the next memory access, fetching them into the cache before we even access them.

To test this, you can make each array element 64 bytes, which is equal to the cache line, to see if the miss rate is the same between loop#1 and loop#2. `perf` also has some performance counter related to that; e.g. LLC-prefetches, L1-dcache-prefetches, etc; but I haven't been able to test on a CPU that support those counters yet (**TODO**)

## Compiler Optimization

Iterating through the data sequentially also makes the compiler's job easier. One important optimization is [auto vectorization](https://en.wikipedia.org/wiki/Automatic_vectorization), which will use vector instructions. These instructions operate at multiple elements at a time, potentially speeding up the program multiple times. You can see it in action in the below examples:

- GCC: https://godbolt.org/z/GWq55rsTr
- Clang: https://godbolt.org/z/4Tcsodf8s

In GCC, loop #1 sum multiple elements at once (see the instructions with `xmm` registers), while loop#2 add to the sum one by one.

## What else?

Is there anything else that could cause the speed differences? That is a common problem that I encounter a lot. Unfortunately, I don't know for if there is an easy way to determine that. One thing we can do is to eliminiate all the above problems and then see if the speed of these 2 loops are the same.

Here are what we can do:

- [Donwload more RAM](https://downloadmoreram.com/) to prevent major page fault.
- Prefault the page with [`explicit_bzero`](https://man7.org/linux/man-pages/man3/bzero.3.html) to prevent minor page fault.
- Use [huge page](https://www.kernel.org/doc/html/latest/admin-guide/mm/hugetlbpage.html) to reduce TLB misses.
- Add padding bytes to make sure each element is in 1 cache line.
- Flush all caches between runs
- Disable all CPU prefetch. You can do that via BIOS (in my machine, I disabled Adjacent Cache Line Prefetch, and Hardware Prefetch) or [MSR](https://en.wikipedia.org/wiki/Model-specific_register) if supported.
- Disable vectorization optimization. In GCC, you can use `__attribute__((optimize("no-tree-vectorize")))`.

After all of that, you can have something like [this](https://github.com/hgminh95/which_loop_is_faster/blob/master/loop_what_else.cpp). Let's see the result

```bash
$ ./bazel-bin/bm_loop_what_else --benchmark_filter=BM_Loop.*/64
2022-04-02T20:11:07+08:00
Running ./bazel-bin/bm_loop_what_else
Run on (6 X 4359.9 MHz CPU s)
CPU Caches:
  L1 Data 32 KiB (x6)
  L1 Instruction 32 KiB (x6)
  L2 Unified 256 KiB (x6)
  L3 Unified 12288 KiB (x1)
Load Average: 0.78, 0.63, 0.47
--------------------------------------------------------------------
Benchmark                          Time             CPU   Iterations
--------------------------------------------------------------------
BM_LoopNo1/64/manual_time       6894 ns        47600 ns       101546
BM_LoopNo2/64/manual_time       8360 ns        49097 ns        84305
```

Loop #2 is still slower and I have no idea why 😢. If you know why that is the case, let me know in the comment below.

## References

- [Linux Virtual Memory](https://docs.kernel.org/vm/index.html)
- [Intel Software Developer's Manual Volume 3](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-system-programming-manual-325384.pdf)
- [Clang Vectorization](https://llvm.org/docs/Vectorizers.html)
- [GCC Vectorization](http://gcc.gnu.org/projects/tree-ssa/vectorization.html)
