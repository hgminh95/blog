---
title: "ASM for C++ developers"
date: 2022-09-17T00:38:02+08:00
draft: false
---

A quick post on how to read asm quickly as C++ developer. Note that this is NOT a guide on how to read assembly code. Rather, this post shows how common C++ constructs/routines are compiled into assembly. From there, it can help you reading asm faster.

<!--more-->

There are plenty of guides on assembly online, for example [here](https://www.cs.virginia.edu/~evans/cs216/guides/x86.html)

To get assembly generated from your code, you can

- [Compiler Explorer](https://godbolt.org/)
- [objdump](https://man7.org/linux/man-pages/man1/objdump.1.html) on object files (executables, shared library, static library, etc)
- [gdb with layout asm](https://stackoverflow.com/questions/589653/switching-to-assembly-in-gdb)
- [gcc -S](https://stackoverflow.com/questions/137038/how-do-you-get-assembler-output-from-c-c-source-in-gcc)

## Setup

This article show a list of C++ code and corresponding assembly. The examples use Intel syntax, and are compiled for Linux x86-64. Most of them are using GCC with optimization enabled (`-O3`). Other platforms and compilers might show different results.

## Examples

<style>
table.mytable tr.mt-code td {
  white-space: pre;
  font-family: monospace;
}

table.mytable tr.mt-code:hover {
  background-color: floralwhite;
}

table.mytable tr.mt-title td {
  background-color: silver;
  font-weight: bold;
}

table.mytable tr.mt-title td:hover {
  background-color: lightgrey;
}

table.mytable tr a {
  font-weight: bold;
  text-color: blue;
}

table.mytable tr span.comment {
  font-weight: bold;
}

table.mytable thead tr th {
  text-align: center;
}
</style>

<table class="mytable">
  <thead>
  <tr>
    <th>C++</th>
    <th>Possible Assembly</th>
  </tr>
  </thead>
  <tbody>

  <tr class="mt-title">
    <td colspan="2">Basic</td>
  </tr>

  <tr class="mt-code">
    <td><span class="comment">// Global variable</span>
int x;</td>
    <td>x:
        .zero   4</td>
  </tr>

  <tr class="mt-code">
    <td><span class="comment">// String literals</span>
std::cout << "ABCDEF";</td>
    <td>.LC0:
        .string "ABCDEF"
foo():
        mov     edx, 6
        mov     esi, OFFSET FLAT:.LC0
        mov     edi, OFFSET FLAT:_ZSt4cout
        ...</td>
  </tr>

  <tr class="mt-code">
    <td><span class="comment">// Array literals</span>
std::array&lt;int, 4&gt; a{1, 2, 3, 4};</td>
    <td>.LC0:
        .long   1
        .long   2
        .long   3
        .long   4</td>
  </tr>

  <tr class="mt-code">
    <td><a href="https://stackoverflow.com/questions/33666617/what-is-the-best-way-to-set-a-register-to-zero-in-x86-assembly-xor-mov-or-and/33668295#33668295">// Set a variable to 0</a>
x = 0;</td>
    <td>	xor   eax, eax</td>
  </tr>

  <tr class="mt-code">
    <td><a href="https://stackoverflow.com/questions/41174867/whats-the-easiest-way-to-determine-if-a-registers-value-is-equal-to-zero-or-no">// Test if variable equals 0</a>
return x == 0;</td>
    <td>        xor     eax, eax
        test    edi, edi
        sete    al
        ret</td>
  </tr>

  <tr class="mt-code">
    <td>struct A {
    int *x;
    A() {
        x = new int[50];
    }
};<br/>
int foo() {
    <a href="https://stackoverflow.com/questions/54281316/why-using-static-variables-inside-a-function-makes-it-runs-slower">// Static variable inside function</a>
    static A a;
    return a.x[0];
}</td>
    <td>foo():
        movzx   eax, BYTE PTR guard variable for foo()::a[rip]
        test    al, al
        je      .L16
        mov     rax, QWORD PTR foo()::a[rip]
        mov     eax, DWORD PTR [rax]
        ret
.L16:
        push    rbx
        mov     edi, OFFSET FLAT:guard variable for foo()::a
        call    __cxa_guard_acquire
        test    eax, eax
        jne     .L17
        mov     rax, QWORD PTR foo()::a[rip]
        pop     rbx
        mov     eax, DWORD PTR [rax]
        ret
.L17:
        mov     edi, 200
        call    operator new[](unsigned long)
        mov     edi, OFFSET FLAT:guard variable for foo()::a
        mov     QWORD PTR foo()::a[rip], rax
        call    __cxa_guard_release
        mov     rax, QWORD PTR foo()::a[rip]
        pop     rbx
        mov     eax, DWORD PTR [rax]
        ret
        mov     rbx, rax
        jmp     .L5
foo() [clone .cold]:
.L5:
        mov     edi, OFFSET FLAT:guard variable for foo()::a
        call    __cxa_guard_abort
        mov     rdi, rbx
        call    _Unwind_Resume</td>
  </tr>

  <tr class="mt-code">
    <td><a href="https://stackoverflow.com/questions/72552/why-does-volatile-exist">// Volatile</a>
void foo(volatile int &x, volatile int &y) {
    y = x;
    y = x;
}<br/>
void bar(int &x, int &y) {
    y = x;
    y = x;
}
    </td>
    <td>foo(int volatile&, int volatile&):
        mov     eax, DWORD PTR [rdi]
        mov     DWORD PTR [rsi], eax
        mov     eax, DWORD PTR [rdi]
        mov     DWORD PTR [rsi], eax
        ret
bar(int&, int&):
        mov     eax, DWORD PTR [rdi]
        mov     DWORD PTR [rsi], eax
        ret</td>
  </tr>

  <tr class="mt-code">
    <td>a = 100;
a = a + 10;
b = b + 10;</td>
    <td><span class="comment">	// Without optimization</span>
	mov     DWORD PTR [rdi], 100 
	add     DWORD PTR [rdi], 10
	add     esi, 10
	mov     DWORD PTR [rdi], esi<br/>
	<span class="comment">// With optimization</span>
	mov     DWORD PTR [rdi], 110
        add     DWORD PTR [rsi], 10</td>
  </tr>

  <tr class="mt-code">
    <td>if (b % 2 == 0) {
    a = a + 20;
}</td>
    <td>	and     esi, 1
	jne     .L1
	add     DWORD PTR [rdi], 20
.L1:
	ret</td>
  </tr>

  <tr class="mt-code">
    <td><span class="comment">// Change division by a variable
// to division by constants</span>
switch (b) {
    case 2:
        return a / 2;
    case 3:
        return a / 3;
    [[likely]] case 7:
        return a / 7;
    default:
        return a / b;
}</td>
    <td>        mov     esi, DWORD PTR [rsi]
        mov     ecx, DWORD PTR [rdi]
        cmp     esi, 7
        jne     .L8
	<span class="comment">// [[likely]] reorder branch. This is a / 7</span>
	<a href="https://en.wikipedia.org/wiki/Division_algorithm#Division_by_a_constant">// Division by a constant</a>
        movsx   rax, ecx
        imul    rax, rax, -1840700269
        shr     rax, 32
        add     eax, ecx
        sar     ecx, 31
        sar     eax, 2
        sub     eax, ecx
        ret
.L8:
        jg      .L3
        cmp     esi, 2
        je      .L4
        cmp     esi, 3
        jne     .L3
        movsx   rdx, ecx
        sar     ecx, 31
        imul    rdx, rdx, 1431655766
        shr     rdx, 32
        mov     eax, edx
        sub     eax, ecx
        ret
.L3:
        mov     eax, ecx
        cdq
        idiv    esi
        ret
.L4:
        mov     edx, ecx
        shr     edx, 31
        lea     eax, [rdx+rcx]
        sar     eax
        ret</td>
  </tr>

  <tr class="mt-code">
    <td>switch (b) {
    case 1:
        return a + 1;
    case 2:
        return a / 2;
    case 3:
        return (a + 5) / 7;
    case 4:
        return (a + 9) / 10;
    case 5:
        return (a - 7) / 8;
    default:
        return a;
}</td>
    <td>        cmp     esi, 5
        ja      .L9
        mov     esi, esi
        jmp     [QWORD PTR .L4[0+rsi*8]]
.L4:
	<a href="https://en.wikipedia.org/wiki/Branch_table">// Jump Table</a>
        .quad   .L9
        .quad   .L8
        .quad   .L7
        .quad   .L6
        .quad   .L5
        .quad   .L3
.L3:
        mov     eax, edi
        sub     eax, 7
        cmovns  edi, eax
        mov     eax, edi
        sar     eax, 3
        ret
.L8:
        lea     eax, [rdi+1]
        ret
.L7:
        mov     eax, edi
        shr     eax, 31
        add     edi, eax
        mov     eax, edi
        sar     eax
        ret
.L6:
        lea     edx, [rdi+5]
        movsx   rax, edx
        imul    rax, rax, -1840700269
        shr     rax, 32
        add     eax, edx
        sar     edx, 31
        sar     eax, 2
        sub     eax, edx
        ret
.L5:
        lea     edx, [rdi+9]
        movsx   rax, edx
        sar     edx, 31
        imul    rax, rax, 1717986919
        sar     rax, 34
        sub     eax, edx
        ret
.L9:
        mov     eax, edi
        ret</td>
  </tr>

  <tr class="mt-code">
    <td><span class="comment">// Normal loop, without optimization</span>
for (auto elem : arr) {
    sum += elem;
}</td>
    <td>        mov     rax, QWORD PTR [rdi]
        mov     rcx, QWORD PTR [rdi+8]
        xor     edx, edx
        cmp     rcx, rax
        je      .L1
.L3:
        add     edx, DWORD PTR [rax]
        add     rax, 4
        cmp     rax, rcx
        jne     .L3
.L1:
        mov     eax, edx
        ret</td>
  </tr>

  <tr class="mt-code">
    <td><a href="https://en.wikipedia.org/wiki/Loop_unrolling">// With loop unrolling</a>
__attribute__((optimize("unroll-loops")))</br>
...
for (auto elem : arr) {
    sum += elem;
}</td>
    <td>        mov     rdx, QWORD PTR [rdi]
        mov     rsi, QWORD PTR [rdi+8]
        xor     eax, eax
        cmp     rsi, rdx
        je      .L4
        mov     rcx, rsi
        sub     rcx, rdx
        sub     rcx, 4
        shr     rcx, 2
        add     rcx, 1
        and     ecx, 7
        je      .L3
        cmp     rcx, 1
        je      .L27
        cmp     rcx, 2
        je      .L28
        cmp     rcx, 3
        je      .L29
        cmp     rcx, 4
        je      .L30
        cmp     rcx, 5
        je      .L31
        cmp     rcx, 6
        jne     .L42
.L32:
        add     eax, DWORD PTR [rdx]
        add     rdx, 4
.L31:
        add     eax, DWORD PTR [rdx]
        add     rdx, 4
.L30:
        add     eax, DWORD PTR [rdx]
        add     rdx, 4
.L29:
        add     eax, DWORD PTR [rdx]
        add     rdx, 4
.L28:
        add     eax, DWORD PTR [rdx]
        add     rdx, 4
.L27:
        add     eax, DWORD PTR [rdx]
        add     rdx, 4
        cmp     rdx, rsi
        je      .L43
.L3:
        add     eax, DWORD PTR [rdx]
        add     rdx, 32
        add     eax, DWORD PTR [rdx-28]
        add     eax, DWORD PTR [rdx-24]
        add     eax, DWORD PTR [rdx-20]
        add     eax, DWORD PTR [rdx-16]
        add     eax, DWORD PTR [rdx-12]
        add     eax, DWORD PTR [rdx-8]
        add     eax, DWORD PTR [rdx-4]
        cmp     rdx, rsi
        jne     .L3
        ret
.L42:
        mov     eax, DWORD PTR [rdx]
        add     rdx, 4
        jmp     .L32
.L43:
        ret
.L4:
        ret</td>
  </tr>

  <tr class="mt-code">
    <td><a href="https://en.wikipedia.org/wiki/Automatic_vectorization">// With vectorize instructions</a>
for (auto elem : arr) {
    sum += elem;
}</td>
    <td>        mov     rdx, QWORD PTR [rdi]
        mov     rdi, QWORD PTR [rdi+8]
        cmp     rdi, rdx
        je      .L7
        lea     rcx, [rdi-4]
        mov     rax, rdx
        sub     rcx, rdx
        mov     rsi, rcx
        shr     rsi, 2
        add     rsi, 1
        cmp     rcx, 8
        jbe     .L8
        mov     rcx, rsi
        pxor    xmm0, xmm0
        shr     rcx, 2
        sal     rcx, 4
        add     rcx, rdx
.L4:
        movdqu  xmm2, XMMWORD PTR [rax]
        add     rax, 16
        paddd   xmm0, xmm2
        cmp     rcx, rax
        jne     .L4
        movdqa  xmm1, xmm0
        psrldq  xmm1, 8
        paddd   xmm0, xmm1
        movdqa  xmm1, xmm0
        psrldq  xmm1, 4
        paddd   xmm0, xmm1
        movd    eax, xmm0
        test    sil, 3
        je      .L1
        and     rsi, -4
        lea     rdx, [rdx+rsi*4]
.L3:
        lea     rcx, [rdx+4]
        add     eax, DWORD PTR [rdx]
        cmp     rdi, rcx
        je      .L1
        lea     rcx, [rdx+8]
        add     eax, DWORD PTR [rdx+4]
        cmp     rdi, rcx
        je      .L1
        add     eax, DWORD PTR [rdx+8]
        ret
.L7:
        xor     eax, eax
.L1:
        ret
.L8:
        xor     eax, eax
        jmp     .L3</td>
  </tr>

  <tr class="mt-title">
    <td colspan="2">Function</td>
  </tr>

  <tr class="mt-code">
    <td>
KxVector<KxLogObserver*, unsigned int>::find(KxLogObserver* const&) const
std::allocator<wchar_t>::allocator()</td>
    <td>	<a href="https://en.wikipedia.org/wiki/Name_mangling">// Name Mangling</a>
	_ZNK8KxVectorIP13KxLogObserverjE4findERKS1_
	_ZNSaIwEC1Ev</td>
  </tr>

  <tr class="mt-code">
    <td>return foo(a) + 5;
    </td>
    <td>	<a href="https://en.wikipedia.org/wiki/X86_calling_conventions">// x86 calling conventions</a>
	sub     rsp, 8
        call    foo(int)
        add     rsp, 8
        add     eax, 5
        ret</td>
  </tr>

  <tr class="mt-code">
    <td><span class="comment">// More complex function call</span>
struct B {
    int a;
    int b;
};<br/>
int foo(int, B, std::string);<br/>
...
x = foo(3, B{.a=1, .b=2}, "abcdef");</td>
    <td>        push    rbp
        mov     eax, 26213
        mov     edi, 3
        movabs  rsi, 8589934593
        push    rbx
        sub     rsp, 40
        lea     rbp, [rsp+16]
        mov     rdx, rsp
        mov     WORD PTR [rsp+20], ax
        mov     QWORD PTR [rsp], rbp
        mov     DWORD PTR [rsp+16], 1684234849
        mov     QWORD PTR [rsp+8], 6
        mov     BYTE PTR [rsp+22], 0
        call    foo(...)
        mov     rdi, QWORD PTR [rsp]
        mov     ebx, eax
        cmp     rdi, rbp
        je      .L2
        mov     rax, QWORD PTR [rsp+16]
        lea     rsi, [rax+1]
        call    operator delete(void*, unsigned long)
.L2:
        add     rsp, 40
        lea     eax, [rbx+1]
        pop     rbx
        pop     rbp
        ret
        mov     rbx, rax
        jmp     .L3
    </td>
  </tr>

  <tr class="mt-code">
    <td><span class="comment">// Virtual function call</span>
struct Foo {
    virtual int foo() = 0;
};<br>
int GetFoo(Foo *foo) {
    return foo->foo() + 5;
}</td>
    <td>        sub     rsp, 8
        mov     rax, QWORD PTR [rdi]
        call    [QWORD PTR [rax]]
        add     rsp, 8
        add     eax, 5
        ret</td>
  </tr>

  <tr class="mt-code">
    <td><span class="comment">// syscall via glibc</span>
#include &lt;fcntl.h&gt;<br/>
auto fp = ::open(path.c_str(), O_APPEND);
    </td>
    <td>	sub     rsp, 8
	mov     rdi, QWORD PTR [rdi]
	mov     esi, 1024
	xor     eax, eax
	call    open
	add     rsp, 8</td>
  </tr>

  <tr class="mt-title">
    <td colspan="2">Multi-threading</td>
  </tr>

  <tr class="mt-code">
    <td>thread_local int x;
...
return x;</td>
    <td>	<a href="https://stackoverflow.com/questions/10810203/what-is-the-fs-gs-register-intended-for">// fs register</a>
	mov     eax, DWORD PTR fs:x@tpoff
        ret</td>
  </tr>

  <tr class="mt-code">
    <td>std::lock_guard<std::mutex> _(mutex);
    </td>
    <td>        mov     eax, OFFSET FLAT:_ZL28__gthrw___pthread_key_createPjPFvPvE
        push    rbx
        mov     rbx, rdi
        test    rax, rax
        je      .L2
        call    __gthrw_pthread_mutex_lock(pthread_mutex_t*)
        test    eax, eax
        je      .L2
        mov     edi, eax
        call    std::__throw_system_error(int)</td>
  </tr>

  <tr class="mt-code">
    <td><a href="https://en.cppreference.com/w/cpp/atomic/atomic">std::atomic</a>&lt;int&gt; x;
...
x.store(1);</td>
    <td>        mov     eax, 1
        xchg    eax, DWORD PTR [rdi]</td>
  </tr>

  <tr class="mt-code">
    <td>x = 1;
<a href="https://en.wikipedia.org/wiki/Memory_barrier">// Memory barrier</a>
std::atomic_thread_fence(std::memory_order_seq_cst);
x = 3;</td>
    <td>	mov     DWORD PTR [rdi], 1
        lock or QWORD PTR [rsp], 0
        mov     DWORD PTR [rdi], 3</td>
  </tr>

  <tr class="mt-title">
    <td colspan="2">Misc.</td>
  </tr>

  <tr class="mt-code">
    <td>try {
    if (x == 3)
        throw std::runtime_error("Test");
    else
        return 4;
} catch (...) {
    return 1;
}</td>
    <td>.LC0:
        .string "Test"
bar(int):
        cmp     edi, 3
        je      .L8
        mov     eax, 4
        ret<br/>
<a href="https://llvm.org/devmtg/2019-10/slides/Kumar-HotColdSplitting.pdf">// Hot Cold Spliting</a>
bar(int) [clone .cold]:
.L8:
        push    rbp
        mov     edi, 16
        push    rbx
        push    rcx
        call    __cxa_allocate_exception
        mov     esi, OFFSET FLAT:.LC0
        mov     rdi, rax
        mov     rbp, rax
        call    std::runtime_error::runtime_error(char const*)
		[complete object constructor]
        mov     edx, OFFSET FLAT:_ZNSt13runtime_errorD1Ev
        mov     esi, OFFSET FLAT:_ZTISt13runtime_error
        mov     rdi, rbp
        call    __cxa_throw
        mov     rbx, rax
        mov     rdi, rbp
        call    __cxa_free_exception
        mov     rdi, rbx
.L4:
        call    __cxa_begin_catch
        call    __cxa_end_catch
        pop     rdx
        mov     eax, 1
        pop     rbx
        pop     rbp
        ret
        mov     rdi, rax
        jmp     .L4
    </td>
  </tr>

  <tr class="mt-code">
    <td>for (int i = 0; i < len; ++i) {
    src[i] = 42;
}</td>
    <td><a href="https://wiki.osdev.org/Stack_Smashing_Protector">	// Stack Smashing Protector</a>
	sub     rsp, 24
        mov     rax, QWORD PTR fs:40
        mov     QWORD PTR [rsp+8], rax
        xor     eax, eax
        test    esi, esi
        jle     .L1
        mov     rax, QWORD PTR [rsp+8]
        sub     rax, QWORD PTR fs:40
        jne     .L6
        mov     edx, esi
        add     rsp, 24
        mov     esi, 42
        jmp     memset
.L1:
        mov     rax, QWORD PTR [rsp+8]
        sub     rax, QWORD PTR fs:40
        jne     .L6
        add     rsp, 24
        ret
.L6:
        call    __stack_chk_fail</td>
  </tr>

  <tr class="mt-code">
    <td>long long sum = 0;
for (int i = 0; i < n; ++i) {
    sum += i;
}<br/>
return sum;</td>
    <td>        test    edi, edi
        jle     .LBB0_1
        lea     eax, [rdi - 1]
        lea     ecx, [rdi - 2]
        imul    rcx, rax
        shr     rcx
        lea     eax, [rdi + rcx]
        dec     eax
        ret
.LBB0_1:
        xor     eax, eax
        ret
    </td>
  </tr>

  <tr class="mt-code">
    <td>int foo(int a) {
    while (a) {}
    return 1;
}</td>
    <td>foo(int):
        mov     eax, 1
        ret</td>
  </tr>

  <tr class="mt-code">
    <td>
    </td>
    <td>
    </td>
  </tr>
  </tbody>
</table>
