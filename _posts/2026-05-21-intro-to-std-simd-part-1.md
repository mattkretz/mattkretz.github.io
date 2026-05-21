---
layout: post
title: Introduction to `std::simd` in C++26 (Part 1)
author: Matthias Kretz
tags: [SIMD, C++, "std::simd", C++26]
---

There's a lot to write about `std::simd` in C++—too much for a single post.
I'd like to show how `std::simd` helps solve actual problems.
Like every tool, it's not a one-size-fits-all solution.
Pick the tool that suits the problem.
And if you already have a good tool for your problem or the auto-vectorizer understands your code just fine 🤷 great, no one is asking you to switch.
However, maybe I'll be able to broaden your perspective on its capabilities and the specific challenges it addresses.

> **A crucial distinction:** `std::simd` is not the same as `std::experimental::simd`.
> Don't judge C++26's data-parallel types by benchmarking the experimental predecessor.
> There's a story behind that, but that's not for today.

## Some Examples

In this first post—of hopefully many more to come—I'll let the preliminary implementation in GCC 16 speak for itself:

Every following code example will assume the following setup:
```c++
#include <simd>
namespace simd = std::simd;
```

`std::simd` provides C++ types that map directly to hardware SIMD registers.
Where a `float` operates on one value, a `simd::vec<float>` operates on $$N$$ values in parallel—the width $$N$$ determined 
by the target architecture.

### Integer Division by 2

```c++
simd::vec<int> half(simd::vec<int> x)
{
  return x / 2;
}

int half(int x)
{
  return x / 2;
}
```
See [Compiler Explorer](https://compiler-explorer.com/z/Pcsn5TcTG):
```objdump
"half(std::simd::basic_vec<int, std::simd::_Abi<16, 1, 1ull>>)":
        vmovdqa32       zmm1, zmm0
        vpsrld  zmm0, zmm0, 31
        vpaddd  zmm0, zmm0, zmm1
        vpsrad  zmm0, zmm0, 1
        ret
"half(int)":
        mov     eax, edi
        shr     eax, 31
        add     eax, edi
        sar     eax
        ret
```

We can see:

1. `simd::vec<int>` compiles to ZMM registers (`-march=znver5`), i.e. it *scales to the target width*.
2. The division by `2` is optimized to shift instructions, using the exact same pattern as for `int`.
3. Change it to `vec<unsigned>` and it'll use a single `vpsrld zmm0, zmm0, 1` instruction.

(If you do change it to `unsigned`, notice that it refuses to implicitly convert `2` (an `int`) to `unsigned`.
This is a shortcoming of the C++ language that could have been done differently, but was ultimately rejected by WG21.
Use `2u` or `std::cw<2>` for the divisor.)

### Constant-Folding and Strength Reduction

```c++
simd::vec<float> six(simd::vec<float> x)
{
  x = 3.f;
  return x * 2.f;
}

simd::vec<float> double_val(simd::vec<float> x)
{
  return x * 2.f;
}
```
See [Compiler Explorer](https://compiler-explorer.com/z/Pzer8fGqz):
```objdump
"six(std::simd::basic_vec<float, std::simd::_Abi<16, 1, 1ull>>)":
        vbroadcastss    zmm0, DWORD PTR .LC1[rip]
        ret
"double_val(std::simd::basic_vec<float, std::simd::_Abi<16, 1, 1ull>>)":
        vaddps  zmm0, zmm0, zmm0
        ret
.LC1:
        .long   1086324736
```

We can see:

1. GCC constant-folds `six` into a "load constant `6.f`" instruction.
2. GCC minimizes `.rodata` by using a broadcast instruction rather than a full vector load.
3. GCC simplifies the multiply by 2 into an addition (`x + x`).
4. This demonstrates that the optimizer understands the semantics of `std::simd` operations, allowing algebraic 
   simplifications.

### Alignment: Automatic Where Possible, Explicit Where Needed

```c++
auto a8 = alignof(simd::vec<float, 8>);
auto a16 = alignof(simd::vec<float, 16>);
auto a32 = alignof(simd::vec<float, 32>);

auto b8 = simd::alignment_v<simd::vec<float, 8>>;
auto b16 = simd::alignment_v<simd::vec<float, 16>>;
auto b32 = simd::alignment_v<simd::vec<float, 32>>;

alignas(simd::alignment_v<simd::vec<float>>) float data[1024];

auto load()
{
  return simd::unchecked_load(data);
}

float data_u[1024];

auto loadu()
{
  return simd::unchecked_load(data_u);
}
```
See [Compiler Explorer](https://compiler-explorer.com/z/44ehPxnP3):
```objdump
"load()":
        vmovaps zmm0, ZMMWORD PTR "data"[rip]
        ret
"loadu()":
        vmovups zmm0, ZMMWORD PTR "data_u"[rip]
        ret
"data_u":
        .zero   4096
"data":
        .zero   4096
"b32":
        .quad   64
"b16":
        .quad   64
"b8":
        .quad   32
"a32":
        .quad   64
"a16":
        .quad   64
"a8":
        .quad   32
```

We can see:

1. The `simd::vec` types (for obvious reasons) communicate their alignment requirements as matching the register `sizeof`.
2. Note that `simd::vec` can span multiple registers when the requested width exceeds a single register's capacity.
   Here, `vec<float, 32>` uses two `ZMM` registers and thus has no higher alignment requirement than a single one.
3. The load call does not *require* specifying alignment and will figure this out itself, if the pointer internally carried the alignment information.
   The `simd::flag_aligned` argument is an optimization that users should use sparingly.

Explicitly typed pointers for alignment would be nice to have (there is some support in `mdspan`; but there's so much more to do here: alignment, aliasing, non-temporal access, ...), as well as simpler out-of-the-box over-aligned allocators so that `std::vector` becomes easier to use.

### `std::simd` Doesn't Integer-Promote / Guards Against Accidental Increase in Register Usage

```c++
auto f(simd::vec<std::int8_t> x, simd::vec<std::int8_t> y)
{
  std::int8_t two = {2};
  auto r = (x + y) / two;
  static_assert(std::is_same_v<decltype(r), simd::vec<std::int8_t>>);
  return r;
}
```
See [Compiler Explorer](https://compiler-explorer.com/z/6fYxj33cs):
```objdump
"f(std::simd::basic_vec<signed char, std::simd::_Abi<64, 1, 1ull>>, std::simd::basic_vec<signed char, std::simd::_Abi<64, 1, 1ull>>)":
        mov     eax, -2139062144
        vpaddb  zmm1, zmm0, zmm1
        vpbroadcastd    zmm0, eax
        vgf2p8affineqb   zmm0, zmm1, zmm0, 0
        vgf2p8affineqb   zmm0, zmm0, ZMMWORD PTR .LC1[rip], 0
        vpaddb  zmm0, zmm0, zmm1
        vgf2p8affineqb   zmm0, zmm0, ZMMWORD PTR .LC2[rip], 0
        ret
.LC1:
        ...
```

We can see:

1. `simd::vec<int8_t>` does not promote to `int`.
   This is probably the most important deviation from the design principle "a `simd::vec<T>` behaves like a `T`".
   The reason should be obvious: `x` in the example uses one register; promoted to `int` it would suddenly require *four* registers.
   This touches upon another design principle: "don't silently introduce performance gotchas, require explicit opt-in".
2. The code required the divisor to be of type `int8_t`. `(x + y) / 2` (where `2` is of type `int`) would have implied
   a conversion from `int8_t` to `int` and thus a silent change from one to four registers.
   `std::simd::basic_vec` requires that both operands have a common type, and since the conversion from `int` to
   `int8_t` is not value-preserving and `vec<int8_t>` is not convertible to `int`, there is no viable `operator/`.
3. The compiler decided to turn the division by `2` into a bizarre sequence of instructions.
   That's because x86 lacks native 8-bit integer vector shifts. Instead, the compiler emulates the shift using GF(2⁸) polynomial multiplication—one of the few byte-granular SIMD instructions available on modern hardware.

### Binary Compatibility Safeguards

```c++
auto f(simd::vec<float, 8> x)
{
  return x + x;
}
```
See [Compiler Explorer](https://compiler-explorer.com/z/qMGKc9dcE):
with `-march=x86-64-v2`:
```objdump
"f(std::simd::basic_vec<float, std::simd::_Abi<8, 2, 0ull>>)":
        movaps  xmm0, XMMWORD PTR [rsp+8]
        mov     rax, rdi
        addps   xmm0, xmm0
        movaps  XMMWORD PTR [rdi], xmm0
        movaps  xmm0, XMMWORD PTR [rsp+24]
        addps   xmm0, xmm0
        movaps  XMMWORD PTR [rdi+16], xmm0
        ret
```
and with `-march=x86-64-v3`:
```objdump
"f(std::simd::basic_vec<float, std::simd::_Abi<8, 1, 0ull>>)":
        vaddps  ymm0, ymm0, ymm0
        ret
```

Note the different ABI tag type for `basic_vec` (`std::simd::_Abi<8, 1, 0ull>` vs. `std::simd::_Abi<8, 2, 0ull>`).
The libstdc++ implementation uses an ABI tag that encodes

- the number of elements (`basic_vec::size()`);
- the number of registers;
- additional bits of differences (vec-mask vs. bit-mask and interleaved vs. contiguous `complex` at this point).

This is a safe-guard against linking code that is not binary compatible.
While this safe-guard is not complete (composition can hide it), it is better than a simple `<T, N>` which happily 
compiles and links and then does weird stuff at runtime.

The question then arises how to deploy binaries that support all kinds of ISA extensions.
It is, however, a question that the C++ standard simply cannot answer.
There do exist patterns and tooling support to make this work.
That's material for a different post.

## Conclusion

There are so many more examples that could be considered.
Send me requests or your favorite ones — especially if you find them surprising or they optimize badly.
That helps us improve the implementation.

While this post does not explain *why* `std::simd` is what it is, I hope these examples give you a taste of what's possible.
Time permitting, there will be more posts on the background and vision of `std::simd`.
Again, send me requests and questions and I'll look into covering them.

Stay tuned — I'll try to make time for more posts on this topic.

**Discuss**: [Mastodon](https://floss.social/@mkretz/116613710972507330)

***

## Postscript: Background & Context

Over the past decade, `std::simd` has evolved through extensive WG21 committee work, peer-reviewed publications, and my doctoral research. The standardization process took time precisely because I insisted on thoroughness: exploring the design space and ensuring every design choice was backed by evidence.

As we reach C++26, the response has been broad. I've received numerous success stories from researchers applying these tools, alongside renewed scrutiny regarding specific use cases. Much of this discussion centers on a natural mismatch in expectations: while all SIMD domains were considered during standardization, data-parallel scientific applications drove the primary design decisions. When API choices conflicted between domains, the needs of HPC codes led the way.

Rather than debating these points abstractly, I believe the best way to evaluate any tool is to look at what it actually does.
