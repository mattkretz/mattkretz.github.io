---
layout: post
title: "Vectorized conversion from UTF-8 using *stdx::simd*"
tags: [C++, SIMD, "std::simd", optimization, benchmarking]
author: Matthias Kretz
---

Bob Steagall [presented his high-speed UTF-8 
conversion](https://www.youtube.com/watch?v=5FQ87-Ecb-A) at CppCon and C++Now 
where he showed that his approach outperformed most existing conversion 
algorithms. For some extra speed, he implemented a function for converting 
ASCII to `char16_t`/`char32_t` using SSE intrinsics. This latter part got me 
hooked, because:
* `stdx::simd` (my contribution to the Parallelism TS 2; note that I use 
`namespace stdx = std::experimental`, because the latter is just way too long.) 
was just sent off for publication by the C++ committee and should have made 
reliance on intrinsics unnecessary.
* I had no prior experience with vectorizing string operations (which is one of 
the reasons my previous vector types library 
[Vc](https://github.com/VcDevel/Vc) didn't have 8-bit integer support). I was 
curious, how hard can it be?
* Bob's presentation made it look like one needs access to special instructions 
like `movmskb` to get good performance.
* Scalability to different vector widths is unclear. The SSE intrinsics 
certainly won't scale. But how much can performance actually scale, knowing 
that the larger the vector, the lower the chance the full vector of chars is 
only made up of ASCII?
* And what about newer ISA extensions such as SSE4.1 which adds instructions 
for converting `unsigned char` to `short` or `int`? Will it help?
* Most important to me, can the code be more readable and portable and at least 
as fast at the same time?
* And is there a chance for vectorization of non-ASCII code point conversions?

## Step 1: Getting `stdx::simd` ready

OK, so first I needed to get my `simd` implementation into shape. I've been 
working on the implementation for a long time already, but I had not achieved a 
state where I was confident to let others use it yet.  But since the GCC 9.1.0 
release, [my implementation that's targeting libstdc++ inclusion is good enough 
for experimental use](https://github.com/VcDevel/std-simd/releases).  It's not 
part of libstdc++ just yet, so you need to install it over GCC9's libstdc++ 
(don't worry, it doesn't actually overwrite anything, it only adds the 
`<experimental/simd>` header):

1. Install GCC 9.1 and make sure `$CXX` points to its `g++`.
2. Install std-simd:
```sh
git clone https://github.com/VcDevel/std-simd
cd std-simd
./install.sh
```
If you need `sudo` to install into the GCC9 prefix use `sudo ./install.sh 
--gxx=$CXX` instead.

At this point, GCC9 will have a conforming (modulo bugs) implementation of the 
"data-parallel types" in the Parallelism TS 2. All you need to do in your code 
is `#include <experimental/simd>` and use the right machine compiler flags 
(e.g. `-march=skylake`) and of course C++17 (`-std=c++17`).

## Step 2: Getting *utf_utils*

I forked [my repository](https://github.com/mattkretz/utf_utils) from 
[BobSteagall/utf_utils](https://github.com/BobSteagall/utf_utils) and applied a 
few cleanups:

1. The cmake code enforced usage of the `g++` binary in the `$PATH`. That cost 
   me more time to realize than it should have. Deleted.
2. The build-type flags were unusual: I just removed them in favor of general 
   warning and the `-std=c++17` flags and passing `-march` flags via 
   `CXXFLAGS`.
3. Added a convenience Makefile for building and benchmarking easily from the 
   toplevel source dir.
4. Removed some dead code from `utf_utils.h`

## Step 3: Putting `simd` and *utf_utils* together

I copied the SSE implementation and reimplemented it to use `stdx::simd`.

The two original functions in *utf_utils*, reproduced below, that use SSE 
intrinsics are `UtfUtils::ConvertAsciiWithSse` overloaded for `char32_t` and 
`char16_t` output. Feel free to skip reading the sources. I'll explain below.
```cpp
void UtfUtils::ConvertAsciiWithSse(char8_t const*& pSrc, char32_t*& pDst) noexcept
{
  __m128i     chunk, half, qrtr, zero;
  int32_t     mask, incr;

  zero  = _mm_set1_epi8(0);                           //- Zero out the interleave register
  chunk = _mm_loadu_si128((__m128i const*) pSrc);     //- Load a register with 8-bit bytes
  mask  = _mm_movemask_epi8(chunk);                   //- Determine which octets have high bit set

  half = _mm_unpacklo_epi8(chunk, zero);              //- Unpack bytes 0-7 into 16-bit words
  qrtr = _mm_unpacklo_epi16(half, zero);              //- Unpack words 0-3 into 32-bit dwords
  _mm_storeu_si128((__m128i*) pDst, qrtr);            //- Write to memory
  qrtr = _mm_unpackhi_epi16(half, zero);              //- Unpack words 4-7 into 32-bit dwords
  _mm_storeu_si128((__m128i*) (pDst + 4), qrtr);      //- Write to memory

  half = _mm_unpackhi_epi8(chunk, zero);              //- Unpack bytes 8-15 into 16-bit words
  qrtr = _mm_unpacklo_epi16(half, zero);              //- Unpack words 8-11 into 32-bit dwords
  _mm_storeu_si128((__m128i*) (pDst + 8), qrtr);      //- Write to memory
  qrtr = _mm_unpackhi_epi16(half, zero);              //- Unpack words 12-15 into 32-bit dwords
  _mm_storeu_si128((__m128i*) (pDst + 12), qrtr);     //- Write to memory

  //- If no bits were set in the mask, then all 16 code units were ASCII, and therefore
  //  both pointers are advanced by 16.
  //
  if (mask == 0)
  {
    pSrc += 16;
    pDst += 16;
  }

  //- Otherwise, the number of trailing (low-order) zero bits in the mask indicates the number
  //  of ASCII code units starting from the lowest byte address.
  else
  {
    incr  = GetTrailingZeros(mask);
    pSrc += incr;
    pDst += incr;
  }
}

void UtfUtils::ConvertAsciiWithSse(char8_t const*& pSrc, char16_t*& pDst) noexcept
{
  __m128i     chunk, half;
  int32_t     mask, incr;

  chunk = _mm_loadu_si128((__m128i const*) pSrc);     //- Load the register with 8-bit bytes
  mask  = _mm_movemask_epi8(chunk);                   //- Determine which octets have high bit set

  half = _mm_unpacklo_epi8(chunk, _mm_set1_epi8(0));  //- Unpack lower half into 16-bit words
  _mm_storeu_si128((__m128i*) pDst, half);            //- Write to memory

  half = _mm_unpackhi_epi8(chunk, _mm_set1_epi8(0));  //- Unpack upper half 
  into 16-bit words
  _mm_storeu_si128((__m128i*) (pDst + 8), half);      //- Write to memory

  //- If no bits were set in the mask, then all 16 code units were ASCII, and therefore
  //  both pointers are advanced by 16.
  //
  if (mask == 0)
  {
    pSrc += 16;
    pDst += 16;
  }

  //- Otherwise, the number of trailing (low-order) zero bits in the mask indicates the number
  //  of ASCII code units starting from the lowest byte address.
  else
  {
    incr  = GetTrailingZeros(mask);
    pSrc += incr;
    pDst += incr;
  }
}
```

The functions are rather long and are a bit scary on first sight. But the task 
is actually quite simple. Both functions load 16 `char`s from the input string 
and convert all 16 values from 8-bit to 32/16-bit integers. The converted 
values are all stored to the destination string. Finally, the source and 
destination pointers are advanced by how many consecutive ASCII chars there 
were in the source string (implicitly discarding the corresponding results from 
the destination string, as they will get overwritten in the next steps of the 
algorithm). In short:

1. Read 16 characters from source.
2. Convert all 16 characters to destination string type.
3. Store 16 converted values.
4. Count how many characters in the source (starting from index 0) are `< 0x80` 
   (i.e. ASCII).
5. Advance source and destination pointers accordingly.

This task can be expressed much clearer in the `stdx::simd` implementation:
```cpp
using char8v = stdx::native_simd<UtfUtils::char8_t>;

template <class T>
void UtfUtils::ConvertAsciiWithSimd(char8_t const*& pSrc, T*& pDst) noexcept
{
  const char8v chunk(pSrc, stdx::element_aligned); // (1')
  chunk.copy_to(pDst, stdx::element_aligned);      // (2')+(3')
  if (none_of(chunk > 0x7f)) {                     // (4a)
    pSrc += chunk.size();                          // (5)
    pDst += chunk.size();                          // (5)
  } else {
    const int n_valid = find_first_set(chunk > 0x7f); // (4b)
    pSrc += n_valid;                               // (5)
    pDst += n_valid;                               // (5)
  }
}
```

In the `stdx::simd` implementation it was trivial to generalize the code on the 
type of the destination string, since the converting store (`copy_to`) 
automatically does the right thing, depending on the type of `pDst`. I believe 
the code is readable as is, and especially with the pointers to the steps in 
the algorithm, this is self-explanatory. There is one important difference, 
though: `std::native_simd<unsigned char>` will not necessarily contain 16 
bytes. It can also contain less (e.g. 64-bit NEON) or more (e.g. 256-bit AVX2).

Since `stdx::simd` is portable by definition, we just created an implementation 
that's readable and portable. So what about the performance. (and performance 
portability?)

## Results, Take 1

I copied the remainder of the conversion algorithm, so that the only difference 
is the `ConvertAsciiWithSse` function. Using the benchmarks in the *utf_utils* 
repository, I got the following results on an Intel Core i7-6700 @ 3.40GHz (for 
all benchmarks, I disabled turbo mode, used the performance governor, and ran 
the benchmark binary in `chrt -f 50`):

![benchmark results skylake part 1](/assets/images/skylake.svg)
![benchmark results westmere part 1](/assets/images/westmere.svg)
![benchmark results core2 part 1](/assets/images/core2.svg)

This is looking very promising for a straightforward implementation.  
Everything that's mostly ASCII is consistently faster. But all the other cases 
are slightly slower. Let's dive a little deeper.

## `pmovmskb`

Regarding the carefully chosen `pmovmskb` instruction Bob mentioned, here's 
what we can find in the `-march=core2` binary:
```
0000000000404380 <uu::UtfUtils::SseBigTableConvert(unsigned char const*, unsigned char const*, char32_t*)>:
...
  movdqu xmm0,XMMWORD PTR [rdi]
  pmovmskb eax,xmm0
  movdqa xmm1,xmm0
  punpckhbw xmm0,xmm3
  punpcklbw xmm1,xmm3
  test   eax,eax
  movdqa xmm4,xmm1
  punpckhwd xmm1,xmm2
  movups XMMWORD PTR [r9+0x10],xmm1
  movdqa xmm1,xmm0
  punpcklwd xmm4,xmm2
  punpckhwd xmm0,xmm2
  punpcklwd xmm1,xmm2
  movups XMMWORD PTR [r9],xmm4
  movups XMMWORD PTR [r9+0x20],xmm1
  movups XMMWORD PTR [r9+0x30],xmm0
  je     .fullvec
  bsf    eax,eax
  cdqe
  lea    r9,[r9+rax*4]
  add    rdi,rax
...

0000000000405800 <long uu::UtfUtils::SimdBigTableConvert<char32_t>(unsigned char const*, unsigned char const*, char32_t*)>:
...
.loop:
...
  movdqu xmm0,XMMWORD PTR [rdi]
  movdqa xmm2,xmm0
  movdqa xmm1,xmm0
  pmovmskb ecx,xmm0
  punpcklbw xmm2,xmm4
  punpckhbw xmm1,xmm4
  movdqa xmm6,xmm2
  movdqa xmm5,xmm1
  punpcklwd xmm6,xmm3
  punpckhwd xmm2,xmm3
  punpcklwd xmm5,xmm3
  punpckhwd xmm1,xmm3
  and    ecx,0xffff
  movups XMMWORD PTR [rax],xmm6
  movups XMMWORD PTR [rax+0x10],xmm2
  movups XMMWORD PTR [rax+0x20],xmm5
  movups XMMWORD PTR [rax+0x30],xmm1
  jne    .partial
  add    rdi,0x10
  add    rax,0x40
  jmp    .loop
.partial:
  movsxd rcx,ecx
  bsf    rcx,rcx
  movsxd rcx,ecx
  lea    rax,[rax+rcx*4]
  add    rdi,rcx
  jmp    .loop
```

Note that `SseBigTableConvert` uses
```objdump
  pmovmskb eax,xmm0
  test     eax,eax
  je       ...
```
and `SimdBigTableConvert` uses
```objdump
  pmovmskb ecx,xmm0
  and      ecx,0xffff
  jne      ...
```
to determine whether all entries in the vector are ASCII. I.e. GCC 9 is able to 
elide the comparison (`chunk > 0x7f`), and work with the same efficiency.

## Conversion instruction sequence

On the other hand, the instruction sequence for the `char8_t` -> `char32_t` 
conversion is obviously longer, so I tried how much difference it would make if 
I add a special case into the `stdx::simd` conversion functions to handle this 
case better.  This required an extra *constexpr-if* case to the internal 
`__convert_all` function and improved the `simd` based implementation 
noticably:

![benchmark results core2 fast conversion](/assets/images/core2-fastcvt.svg)

That's a visible improvement. Obviously conversions have a bit more room for 
optimization in the *std-simd* implementation.

## SSE4.1 and PTEST

Note, that with `-march=(westmere|skylake)`, both conversion and branching are 
done differently because SSE4.1 includes the 
[pmovzxb[wd]](https://www.felixcloutier.com/x86/pmovzx) and 
[ptest](https://www.felixcloutier.com/x86/ptest) instructions.
Consequently, the `pmovmskb` optimization does not trigger. Instead an 
optimization folding the comparison with `ptest` should be done, but [that 
issue is still open](https://gcc.gnu.org/PR90483). Thus, the `-march=westmere` 
binary contains the following instruction sequence for `ConvertAsciiWithSimd`:
```
0000000000405680 <long uu::UtfUtils::SimdBigTableConvert<char32_t>(unsigned char const*, unsigned char const*, char32_t*)>:
...
.loop:
...
  movdqu xmm0,XMMWORD PTR [rdi]
  movdqa xmm7,xmm6
  pcmpgtb xmm7,xmm0
  movdqa xmm3,xmm0
  movdqa xmm2,xmm0
  movdqa xmm1,xmm0
  psrldq xmm3,0x4
  pmovzxbd xmm4,xmm0
  psrldq xmm2,0x8
  pmovzxbd xmm3,xmm3
  movups XMMWORD PTR [rax],xmm4
  ptest  xmm7,xmm5
  pmovzxbd xmm2,xmm2
  movups XMMWORD PTR [rax+0x10],xmm3
  psrldq xmm1,0xc
  movups XMMWORD PTR [rax+0x20],xmm2
  pmovzxbd xmm1,xmm1
  movups XMMWORD PTR [rax+0x30],xmm1
  jne    .partial
  add    rdi,0x10
  add    rax,0x40
  jmp    .loop
.partial:
  pmovmskb ecx,xmm7
  movzx  ecx,cx
  bsf    rcx,rcx
  movsxd rcx,ecx
  lea    rax,[rax+rcx*4]
  add    rdi,rcx
  jmp    .loop
```
The instruction sequence that tests for all ASCII is (`xmm5` is initialized to 
all bits 1):
```
  pcmpgtb xmm7,xmm0
  ptest   xmm7,xmm5
  jne     .partial
```
It's not obvious from the latency numbers on 
[InstLatx64](https://github.com/InstLatx64/InstLatx64/) whether this is less
efficient as the instruction sequence with `pmovmskb`. In any case, it could be 
more efficient once [PR90483](https://gcc.gnu.org/PR90483) is resolved (`xmm5` 
would now be initialized to a vector with each byte set to `0x80`):
```
  ptest   xmm0,xmm5
  jne     .partial
```

Using either inline assembly magic or the `_mm256_testz_si256` intrinsic, it's 
possible to test how we could perform once 
[PR90483](https://gcc.gnu.org/PR90483) were resolved:
```cpp
template <class T>
void UtfUtils::ConvertAsciiWithSimd(char8_t const*& pSrc, T*& pDst) noexcept
{
  const char8v chunk(pSrc, stdx::element_aligned);
  constexpr char8v signbit = 0x80;
  asm ("vptest %1,%0" :: "x"(chunk), "x"(signbit) : "cc");  // note how you can
                                  // use simd<T> objects directly in inline asm
  chunk.copy_to(pDst, stdx::element_aligned);
  asm goto("jne %l0" :::: partial);
  pSrc += chunk.size();
  pDst += chunk.size();
  return;
partial:
  const int n_valid = find_first_set(chunk > 0x7f);
  pSrc += n_valid;
  pDst += n_valid;
}

// OK, inline asm is frightening. Here's the intrinsics variant:
template <class T>
void UtfUtils::ConvertAsciiWithSimd(char8_t const*& pSrc, T*& pDst) noexcept
{
  const char8v chunk(pSrc, stdx::element_aligned);
  chunk.copy_to(pDst, stdx::element_aligned);
  constexpr char8v signbit = 0x80;
  if (_mm256_testz_si256(__m256i(signbit), __m256i(chunk))) {
    pSrc += chunk.size();
    pDst += chunk.size();
  } else {
    const int n_valid = find_first_set(chunk > 0x7f);
    pSrc += n_valid;
    pDst += n_valid;
  }
}
```

With the inline assembly hack, we see how performance improves slightly 
(*kewb-vir-simd-ptest*). Note that the *-fastcvt* change from above doesn't 
apply here because `-march=skylake` uses `pmovzxbd` for the conversion.
![benchmark results skylake ptest](/assets/images/skylake-ptest.svg)
The takeaway from this exercise is:
*You're not painted into a corner when using `stdx::simd`. You can cast to 
 intrinsic or builtin vector types and thus also make use of intrinsics, 
 compiler builtins, or inline assembly if you have a pressing performance 
 issue.*

## What's up with *stress_test_2.txt*?

You probably noticed that *stress_test_2.txt* shows the longest times. This 
file contains text of the nasty kind: "涁 騑 枢 狾 鱄 尟 [...]", i.e. a 
multibyte code point followed by a space - lots of them. It's obvious that the 
vectorized ASCII decoder is not doing us a favor for this kind of input.  
Whenever it sees a space, it decodes 16 or 32 bytes. However, only the first 
one is good for use. All the extra work is wasted. A simple branch before the 
SIMD code might help:
```cpp
  if (char8v::size() > 1 && pSrc[1] > 0x7f) {
    *pDst++ = *pSrc++;
  } else {
    const char8v chunk(pSrc, stdx::element_aligned);
    chunk.copy_to(pDst, stdx::element_aligned);
    // ...
```

So let's try it:

![benchmark results skylake ptest](/assets/images/skylake-quickexit.svg)

It's a huge improvement for *stress_test_2.txt* and a slight improvement for a 
few other files, while also a making *russian_wiki.txt* noticeably slower. The 
obvious explanation is that we get many branch mispredictions on the new branch 
because of either just a single space or punctuation + space or HTML between 
Cyrillic words.

## Conversion instructions

The `pmovzxbd` instruction of SSE4 is the other important difference to the 
`core2` variant. However, after all my benchmarking and checking resulting 
instruction sequences, I conclude that the newer SSE4 instruction does neither 
improve nor degrade the performance. With AVX2 `pmovzxbd` is the only variant I 
implemented, because it can then convert to 8 `char32_t` easily. The unpack 
sequence, on the other hand, is no fun to implement and test, because the AVX2 
unpack instructions act per 128-bit part of the vector register and thus 
require some clever shuffling to not mess up the order of characters in the 
string. Those shuffles surely aren't going to make it faster. (in short: I am 
convinced `pmovzxbd` is more efficient with AVX2.)

## Conclusions and more ideas

1. `stdx::simd` is quite ready for taking it for a test drive and learning how 
   to best vectorize your applications. You can certainly do serious work with 
   it, but at this point, of course, you don't have full portability and no 
   100% guarantee for standardization in the proper C++ document. An earlier 
   implementation of `stdx::simd` was used for a [large distributed application 
   running on a Xeon Phi cluster](https://journals.sagepub.com/doi/10.1177/1094342018819744).

2. I (and Jonathan) need to work on getting `simd` into libstdc++, to make it 
   easier for you!

3. UTF-8 conversion of non-ASCII characters is possible. I tried it and it can 
   improve the conversion efficiency even more (though not much). But this post 
   is too long already. I hope to find time to write a follow up. Let me know 
   if there's interest.
