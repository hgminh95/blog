---
title: "Solving Project Euler 820 on different hardwares (part 1)"
date: 2023-01-20T20:38:02+08:00
tags: ["performance"]
thumbnail: euler820.png
draft: false
---

For those of you who don't know, [Project Euler](https://projecteuler.net/) is a series of challenging mathematical/computer programming problems. Though the problem is more on figuring out the mathematical insight, this post will focus on the engineering side. We are going to look at problem [#820](https://projecteuler.net/problem=820), and try to solve it on various kinds of hardwares, ranging from CPU to FPGA, to see how much speed gains we can get from engineering alone.

Also, it is fun to see how different hardware affects on performance of the same codebase.

All the source code is available at [Github](https://github.com/hgminh95/euler820).

This is part 1 of a 2-part series.

<!--more-->

## The Solution

At the heart of the problem is how to calculate $n^{th}$ digit of $\frac{1}{x}$. To do so, we multiply $\frac{1}{x}$ with $10^n$, and get the last digit of the integral part.

$$ digit = \left\lfloor \frac{10^n}{x} \right\rfloor \\; mod \\; 10 $$

[Intuitively,](https://stackoverflow.com/questions/804934/getting-a-specific-digit-from-a-ratio-expansion-in-any-base-nth-digit-of-x-y)

$$ digit = \left\lfloor \frac{(10^{n-1} \\; mod \\; x) * 10}{x} \right\rfloor $$

<p style="text-align:right; font-style: italic">(if you do division like <a href="https://baiontap.com/wp-content/uploads/2020/09/2-1.png" target="_blank">this</a>, the formula above probably comes more naturally)</p>

To calculate $10^{n - 1} \\; mod \\; x$ quickly, we can use [binary exponentiation](https://en.wikipedia.org/wiki/Modular_exponentiation#Left-to-right_binary_method). The time complexity would be $O(n log_n)$. With the problem asking for result of $n = 10^7$, this algorithm should be able to produce an answer within a blink.

## CPU

### Recursive

The most obvious implementation would be to calculate the exponential modulo recursively. At each step, you reduce `power` by half, until it reaches 0.

```cpp
constexpr int32_t ExpModuloRecursive(int32_t power, int32_t modulo) {
  if (power == 0)
    return 1;

  auto res = ExpModuloRecursive(power / 2, modulo);
  if (power % 2 == 0) {
    return (static_cast<int64_t>(res) * res) % modulo;
  } else {
    return ((static_cast<int64_t>(res) * res) % modulo * 10) % modulo;
  }
}
```

### Looping through the bit

Another way is to decompose `N` into

$$10'000'000 = 2^7 + 2^9 + 2^{10} + 2^{12} + 2^{15} + 2^{19} + 2^{20} + 2^{23}$$

To calculate $10^{2^x}$, we just need to repeatedly square itself by $x$ times. Multiply all of them together and you get the result.

The nice aspect about this is that you don't need recursion, which is usually costly, especially for non-[tail recursion](https://en.wikipedia.org/wiki/Tail_call).

```cpp
constexpr int GetKBitOfN(int32_t n, int k) {
  return (n >> k) & 1;
}

// Get index of highest set bit in a 32-bit integer.
// In 1-index.
constexpr int GetHighestSetBit(int32_t n) {
  return sizeof(int32_t) * 8 - __builtin_clz(n);
}

constexpr int32_t ExpModuloBit(int32_t power, int32_t modulo) {
  int64_t res = 1;
  int64_t curr_power = 10;

  for (int i = 0; i < GetHighestSetBit(power); ++i) {
    if (GetKBitOfN(power, i)) {
      res = (res * curr_power) % modulo;
    }
    curr_power = (curr_power * curr_power) % modulo;
  }

  return res;
}
```

### Faster Modulo

Another thing to look for is modulo operation. Modulo and integer division are among the slowest arithmetic operations (even slower than floating point operation). You could probably execute a hundred integer addition/subtraction before finishing a [single integer division](https://www.agner.org/optimize/instruction_tables.pdf).

There are various methods to speed up modulo reductions: [Barrett Reduction](https://en.wikipedia.org/wiki/Barrett_reduction), [Montgomery](https://en.wikipedia.org/wiki/Montgomery_modular_multiplication), etc. In fact, your compiler uses a lot of tricks to not have to divide integers; e.g. converting division by power of 2 to left shift, convert division by constant to multiplication and left shift, etc.

[This blog post](https://lemire.me/blog/2016/06/27/a-fast-alternative-to-the-modulo-reduction/) provides a pretty good summary.

Below is an example of the code snippet using [`fastmod`](https://github.com/lemire/fastmod).

```cpp
constexpr int32_t ExpModuloBitWithFastMod(int32_t power, int32_t modulo) {
  int64_t res = 1;
  int64_t curr_power = 10;

  auto M = fastmod::computeM_u64(modulo);

  for (int i = 0; i < GetHighestSetBit(power); ++i) {
    if (GetKBitOfN(power, i)) {
      res = fastmod::fastmod_u64(res * curr_power, M, modulo);
    }
    curr_power = fastmod::fastmod_u64(curr_power * curr_power, M, modulo);
  }

  return res;
}
```

In the GitHub repos, I tested it with `fastmod` and [`libdivide`](https://libdivide.com/).

### Loop Unroll

This is a pretty common [technique](https://en.wikipedia.org/wiki/Loop_unrolling) to optimize stuff. The idea is to run multiple iterations at 1 go, to avoid the cost of checking the loop conditions, and also allow more instructions to be [pipelined](https://en.wikipedia.org/wiki/Instruction_pipelining).

Since `GetNthDigitFromFracX` is quite a complex function, compilers won't be able to unroll it. We will have to help ourselves in this case,

```cpp
// Assume N % 4 == 0
for (int i = 0; i < N / 4; ++i) {
  std::array<int64_t, 4> digits{1, 1, 1, 1};
  std::array<int64_t, 4> curr_power{10, 10, 10, 10};

  for (int k = 0; k < GetHighestSetBit(N - 1); ++k) {
    if (GetKBitOfN(N - 1, k)) {
      for (int j = 0; j < 4; ++j) {
        digits[j] = (digits[j] * curr_power[j]) % (4 * i + j + 1);
      }
    }

    for (int j = 0; j < 4; ++j) {
      curr_power[j] = (curr_power[j] * curr_power[j]) % (4 * i + j + 1);
    }
  }

  for (int j = 0; j < 4; ++j) {
    digits[j] *= 10;
    digits[j] /= 4 * i + j + 1;
  }

  for (int j = 0; j < 4; ++j) {
    res += digits[j];
  }
}
```

### Multi-Thread

Since each digit can be computed independently, multi-thread is an obvious choice. We can just split up the work between different threads, and then add them up together at the end.

There will be a bit of overhead to managing these threads, and there could be lower performance if they have to compete for the same cores; but overall, this should be pretty straightforward speed up.

```cpp
for (int i = 0; i < NUM_THREADS; ++i) {
  threads.push_back(std::thread([&sum, i]() {
      // Run from i * N / NUM_THREADS + 1
      auto start = i * N / NUM_THREADS + 1;
      auto end = (i + 1) * N / NUM_THREADS;
      sum.fetch_add(CalculatePartialS(
          N, start, end,
          [](int32_t n, int32_t x) { return ExpModuloBit(n, x); }
      ));
  }));
}
```

You can also combine with above optimization to yield even better result.

## Results and Explanation

The code is compiled under GCC 10 and Clang 15 under 3 CPUs:

- Apple M2 - with Clang
- AMD Ryzen 9 (Zen) - with GCC
- Intel Xeon Gold (Cascade Lake) - with GCC
- Intel Core i9 10th gen (Comet Lake) - with GCC

Each cell has 2 numbers, the first one for $N = 10^7$ and another for $N = 10^8$. All numbers are in seconds. They are measured with `time` command, and the runtime environment is not very well-controlled, so don't expect very accurate results.

Also, don't look too much into absolute latency between different CPUs, as I don't really choose the same equivalent CPUs. I found numbers of different variants in the same CPU tell us more interesting things in this case.

Variant         |  M2          |  Zen         | Cascade Lake | Comet Lake   |
-------         | :--:         | :---:        | :----------: | :---:        |
Recursive 	| 1.29 / 11.16 | 1.29 / 15.9  | 3.10 / 35.62 | 2.27 / 26.10 |
Loop 		| 1.23 / 10.04 | 1.94 / 24.17 | 3.49 / 40.47 | 2.57 / 29.88 |
With fastmod 	| 1.54 / 13.21 | 1.29 / 15.12 | 1.76 / 20.03 | 1.29 / 14.90 |
With libdivide  | 1.44 / 12.41 | 0.82 / 9.3   | 1.29 / 14.42 | 0.96 / 10.72 |
Loop Unroll 	| 0.70 / 4.21  | 1.83 / 23.79 | 2.82 / 34.02 | 2.12 / 25.51 |
Multi-Thread 	| 0.53 / 2.54  | 0.65 / 7.63 | -            | - |

(I forgot to record the time of some Multi-Thread programs, but they are pretty predictable so we can safely ignore it)

M2 seems to be fastest here. But please note that this workload is not a common one, since there is pretty much no memory access. So don't generalize this result.

GCC partially unrolls and inlines the recursive functions (last 8 steps are unrolled in my test), while Clang keeps the recursive structure. However, when we loop through each bit, GCC fails to do this optimization, leading to faster code in `recursive` compared to `loop`. This is possible because in my code, `N` is a compiled-time constant so GCC knows exactly how the recursion is gonna be.

```asm
  # Asm generated by GCC for the recursive case.

  400987:       e8 34 02 00 00          call   400bc0 <_Z18ExpModuloRecursiveii.part.0>
    return (static_cast<int64_t>(res) * res) % modulo;
  40098c:       48 98                   cdqe   
  40098e:       48 0f af c0             imul   rax,rax
  400992:       48 99                   cqo    
  400994:       48 f7 f9                idiv   rcx
    return (static_cast<int64_t>(res) * res * 10) % modulo;
  400997:       48 0f af d2             imul   rdx,rdx
  40099b:       48 8d 04 92             lea    rax,[rdx+rdx*4]
  40099f:       48 01 c0                add    rax,rax
  4009a2:       48 99                   cqo    
  4009a4:       48 f7 f9                idiv   rcx
  4009a7:       48 0f af d2             imul   rdx,rdx
  4009ab:       48 8d 04 92             lea    rax,[rdx+rdx*4]
  4009af:       48 01 c0                add    rax,rax
  4009b2:       48 99                   cqo    
  4009b4:       48 f7 f9                idiv   rcx
  4009b7:       48 0f af d2             imul   rdx,rdx
  4009bb:       48 8d 04 92             lea    rax,[rdx+rdx*4]
  4009bf:       48 01 c0                add    rax,rax
  ...
```

The latency differences between ways of doing modulo mostly due to different latency/throughput of arithmetic operations on each hardware.

`fastmod` is slower than `libdivide` here, but note that this is 64-bit modulo; it could be faster on 32-bit.

`fastmod`/`libdivide` is slower in M2, suggesting faster integer division in Apple hardware. [This benchmark](https://ridiculousfish.com/blog/posts/benchmarking-libdivide-m1-avx512.html) also shows similar results. This is actually pretty impressive. It is unclear on how Apple does it, but likely because there are multiple units for integer division.

The loop unroll version is much faster in M2 compared to other architectures. This should be due to inability to pipeline integer division efficiently in x86. A bit tricky to be 100% sure though as I don't know any tooling or document to give more insight on performance implication of M2.

As mentioned above, you can combine them to get faster result, depending on the arch. For example, in M2, you can run [multiple threads with loop unroll](https://github.com/hgminh95/euler820/blob/main/multi_thread_with_unroll.cpp). The test on my machine show that it can run with $N = 10^8$ in ~ 0.8 seconds, more than 10 times faster than the original solution.

## Future Works

SIMD, GPU and FPGA coming soon...
