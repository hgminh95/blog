---
title: "Which Loop Is Faster"
date: 2022-03-23T20:54:08+08:00
tags: ["performance", "os", "memory"]
draft: false
---

The first loop access the memory sequentially from the beginning to the end of the array. The second one skip N steps between 2 iterations. **Which one is faster?**

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

I have run it and here is the result (you can find the code at [Github](https://github.com/hgminh95/which_loop_is_faster) - it is not exactly the same but should be close enough for this discussion)

```
---------------------------------------------------------------------
Benchmark                           Time             CPU   Iterations
---------------------------------------------------------------------
BM_LoopNo1/16                     131 ns          108 ns      8960000
BM_LoopNo1/64                    1965 ns         1726 ns       407273
BM_LoopNo1/512                 129987 ns        97656 ns         5600
BM_LoopNo1/4096               9786057 ns      8789062 ns          112
BM_LoopNo1/8192              39223035 ns     21354167 ns           30
BM_LoopNo2/16                     128 ns         73.4 ns     10000000
BM_LoopNo2/64                    1975 ns         1420 ns       407273
BM_LoopNo2/512                 784854 ns       619071 ns         1792
BM_LoopNo2/4096             139502128 ns    111111111 ns            9
BM_LoopNo2/8192             794375658 ns    656250000 ns            1
```

You can clearly see that loop #1 is faster. Now, let's find out why that is the case.

## Page fault

[Page fault](https://en.wikipedia.org/wiki/Page_fault) happens when we try to access memory that is not in the main memory. The kernel need to delay the program and load the memory from storage like hard drive, which is very expensive. Since loop #2 jumps back and forth between different pages, it could trigger more [page fault](https://en.wikipedia.org/wiki/Page_fault) than #1 if N is big enough.

## TLB misses

The address refered by the application is virtual address. To access the memory, we actually need to translate them into physical address, by walking through the [page table](https://en.wikipedia.org/wiki/Page_tabl) (depsite the name, it is actually like a tree, hence "walking"). Since page table can be very large (say we have 16GB of RAM, which each page is 4KB, you do the math), we have something called [TLB](https://en.wikipedia.org/wiki/Translation_lookaside_buffer) to cache the page table. TLB miss happen when we try to access a page which is not in TLB.

Again, going back and forth between different pages won't help in this case either.

To see number of TLB misses, you can run below commands:

```bash
$ perf stats -e <tlb something something> ./bazel-bin/bm_loop --benchmark_filter BM_LoopNo1

....

$ perf stats -e <tlb something something> ./bazel-bin/bm_loop --benchmark_filter BM_LoopNo2
```

## CPU Cache

Fetching memory from RAM is actually very slow, it could cost hundreds of clock cycles. That's why we have L1, L2, L3 cache, which will save the memory near the CPU, so in case if we need to access them again, it would be fast (a few cycles if in L1 cache). But we only access each element once, how can cache help? Turn out we don't load 1 element to the cache at one time, we load a cache line, which is 64 bytes at a time. So everytime we access an element, it will load 15 other elements (assuming int is 32-bit) to the cache, speeding up the next 15 memory access if we iterate them in sequential order.

You can check the cache miss number with below commands:

```bash
$ perf stat -e .... ./bazel-bin/bm_loop --benchmark_filter BM_LoopNo1
```

## Cache Prefetching

You might realize something is off by looking at above number. How can number of cache misses are so small compared to 1:15 ratio? Again, CPU is one step ahead of us. It has something called prefetcher which will try to find out the memory access pattern to predict the next memory access, fetching them into the cache before we even access them.

## Compiler Optimization

Magic magic

## References
