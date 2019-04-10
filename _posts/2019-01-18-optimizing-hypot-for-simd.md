---
layout: post
title: "Optimizing *std::hypot* for *simd* arguments"
tags: [C++, SIMD, "std::simd", math, optimization, benchmarking]
author: Matthias Kretz
---

## Pythagoras — it's complicated
Do you know about `std::hypot(a, b)` and `std::hypot(a, b, c)`? (The 
3-argument overload exists since C++17, sometimes referred to as hypot3.)  Why 
do the C and C++ standard libraries provide a function that is as simple as 
`sqrt(a*a + b*b)`? It doesn't save enough characters to warrant its existence, 
right? Have you ever considered what happens if the input values are "very" 
small or large? Or if an input is an IEEE754 special value such as infinity or 
NaN?  Have you ever considered how precise the calculation is, especially if an 
exact answer is obvious if one of the inputs is 0?

The standard function is specified to avoid overflow and underflow in 
intermediate calculations. Consider the following example:
```cpp
float a = 0x1p70f;  // = 2⁷⁰ (~1.2e21)
float b = 0;
float r = std::sqrt(a * a + b * b); // = inf
```
`a * a` thus is `0x1p140f` (~1.4e42) which overflows the 8-bit exponent 
(subnormal, -126 – 127, inf/nan) for single precision. However, mathematically 
it's obvious that `r = a` is the right answer, not `r = inf`. The naïve 
`sqrt(a*a + b*b)` thus doesn't work for inputs greater than `0x1.fffffep63f` 
(~1.8e19) (if the other input is 0, otherwise the cut-off is even lower) as 
well as for inputs smaller than (let's ignore subnormals for now) `0x1p-63f` 
(~1e-19).

Regarding precision, `sqrt(a*a + b*b)` isn't really bad since `std::sqrt` is 
required by IEEE754 to have .5 ULP precision and can thus potentially reduce 
the error of `a*a + b*b` (where each operation has .5 ULP precision). I'd 
expect a total error within 1 ULP (if anyone can give me a good pointer to a 
formalism of fp error analysis, please do). The precision issue becomes 
relevant when you want to support the full range of possible input values 
without over-/underflow.

## How to avoid over-/underflow
The idea to support the full range of input values is a simple transformation
$$ \sqrt{a^2 + b^2} = |b|\cdot\sqrt{(\frac{a}{b})^2+1} $$ where $$|b|\ge|a|$$.
Consequently $$\frac{a}{b} \le 1$$ and thus $$(\frac{a}{b})^2$$ won't overflow.
If $$(\frac{a}{b})^2$$ underflows, we didn't have to care about its value 
anyway because the addition with 1 would have discarded all bits anyway.

### 1. Optimization idea
Instead of factoring out $$|b|$$, we can use a value that is of the same 
magnitude but might reduce the errors introduced through intermediate 
calculations. The most obvious candidate is to round $$|b|$$ down to the next 
power-of-2 value, which I'll call $$b'$$. This is a trivial bitwise `and` 
operation, setting all mantissa bits to 0. The resulting floating point value 
will only scale the exponent bits when used in a multiplication or division.
Our `hypot` implementation thus calculates 
$$b'\cdot\sqrt{(\frac{a}{b'})^2+(\frac{b}{b'})^2}$$. The final multiplication 
($$b'\cdot\sqrt{\cdot}$$) and both fractions ($$\frac{a}{b'}$$ and 
$$\frac{b}{b'}$$) are without loss of precision. Thus, the resulting error is 
equivalent to the error of $$\sqrt{a^2+b^2}$$.

### 2. Division is slow
Division and square root instructions run on the divider port (pipeline) of the 
CPU and will therefore dominate the execution time of the `hypot` 
implementation. We cannot avoid the square root, but we can avoid the division.
The first idea is to turn the two divisions into one division and two 
multiplications: $$
b'\cdot\sqrt{(\frac{a}{b'})^2+(\frac{b}{b'})^2} = 
b'\cdot\sqrt{(a\cdot\hat{b})^2+(b\cdot\hat{b})^2}$$, with $$\hat{b} = 
\frac{1}{b'}$$.

The $$\frac{1}{b'}$$ division is a trivial operation, though, because it just 
needs to flip the sign of the exponent. After all, $$b'$$ was constructed to be 
a power-of-2 value: $$b'=2^n \Rightarrow \hat{b} = 2^{-n}$$. If you recall how 
the exponent is stored in an IEEE754 floating point value, you might realize 
that flipping all exponent bits almost produces the correct result. It's just 
off by one. And if the exponent is off by one, the resulting floating point 
value is off by a factor of 2.

$$\Rightarrow \hat{b} = 2b' $$`^`$$ ∞$$ (or in code: `b_hat = ((b & infinity) ^ 
infinity) * .5`; the xor operator is not defined for `float`, but you get the 
idea).  Note that using an addition instead of multiply by two is faster for 
two reasons (An optimizing compiler will turn `2*x` into `x+x` by itself, so 
feel free to forget about this optimization.):
1. it doesn't require loading a constant value (`0x4000'0000` in this case);
2. addition instructions typically have a lower latency than multiplication 
   instructions.

$$\frac{b}{b'} = b\cdot\hat{b}$$ is another trivial bit operation if we look 
closely. $$b'$$ was constructed to store the exponent of $$b$$. Thus 
$$\frac{b}{b'}$$ is $$b$$ with its exponent set to $$2^0 = 1$$. Using a 
bitwise-and and bitwise-or operation, we can easily overwrite the exponent of 
$$b$$ to avoid the multiplication $$b\cdot\hat{b} = $$`(b & (min - denorm_min)) 
| 1`.  The CPU can therefore schedule $$(b\cdot\hat{b})^2$$ earlier and thus 
also execute the following FMA and square root earlier, reducing the total 
latency of the `hypot` function.

### Subnormal, NaN, Zero, and Infinity
There are corner cases. A proper implementation must care for the Annex F 
requirements (C Standard, but imported into C++). As is often the case with 
floating point, the .01% corner cases lead to significant effort in the 
implementation, and if done carelessly to unfortunate slowdown. I'll let you 
figure it out from the implementation below.

## I was talking about *std::experimental::simd\<float\>*
You probably didn't notice other than by the title, but everything I said I 
didn't write for plain `float` but for [`std::experimental::simd<T, 
Abi>`](https://www.iso.org/standard/70588.html) ([last 
draft](https://wg21.link/n4793)). Here's my implementation, targeting 
libstdc++:
```cpp
template <typename T, typename Abi>
simd<T, Abi> hypot(const simd<T, Abi>& x, const simd<T, Abi>& y)
{
  if constexpr (simd<T, Abi>::size() == 1) {
    return std::hypot(T(x[0]), T(y[0]));
  } else if constexpr (is_fixed_size_abi_v<Abi>) {
    return fixed_size_apply<simd<T, Abi>>(
      [](auto a, auto b) { return hypot(a, b); },
      x, y);
  } else {
    using namespace __proposed::float_bitwise_operators;
    using Limits = std::numeric_limits<T>;
    using V      = simd<T, Abi>;
    V absx       = abs(x);
    V absy       = abs(y);
    V hi         = max(absx, absy);
    V lo         = min(absy, absx);

    constexpr V inf(Limits::infinity());
    // if hi is subnormal, avoid scaling by inf & final mul by 0 (which
    // yields NaN) by using min()
    V scale = 1 / Limits::min();
    // invert exponent w/o error and w/o using the slow divider unit:
    where(hi > Limits::min(), scale) = ((hi & inf) ^ inf) * T(.5);
    // adjust final exponent for subnormal inputs
    V hi_exp                          = Limits::min();
    where(hi > Limits::min(), hi_exp) = hi & inf;
    constexpr V mant_mask = Limits::min() - Limits::denorm_min();
    V h1 = (hi & mant_mask) | V(1);
    where(hi < Limits::min(), h1) -= V(1);
    V l1 = lo * scale;
    V r  = hi_exp * sqrt(h1 * h1 + l1 * l1);

#ifdef STDC_IEC_559__
    // fixup for Annex F requirements
    V fixup                                     = hi;
    where(isunordered(x, y), fixup)          = Limits::quiet_NaN();
    where(isinf(absx) || isinf(absy), fixup) = inf;
    where(!(lo == 0 || isunordered(x, y) || isinf(absx) || 
            isinf(absy)), fixup) = r;
    r = fixup;
#endif
    return r;
  }
}
```

## Does it fly?
First of all, I made sure it is conforming and within 1 ULP of a precise 
implementation.

I tried to measure both latency and throughput.
Measurements details:
- Intel Xeon W-2123 @ 3.60GHz
- GCC 9
- `-O2 -march=native` (i.e. using AVX512VL instructions for `xmm` and `ymm` vectors)
- Intel pstate set to `no_turbo` and `scaling_governor` set to `performance`
- Ubuntu 18.10

### Throughput

| `T hypot(T, T)` | TSC cycles/call | Speedup per value relative to `T` |
| --- | --- | --- |
| `float`                 | 17.1208 |
| `simd<float, scalar>`   | 17.218  | 0.994355
| `simd<float, __sse>`    | 15.5264 | 4.41074
| `simd<float, __avx>`    | 15.9294 | 8.59831
| `simd<float, __avx512>` | 18.5829 | 14.7411
| `double`                 | 30.0783 |
| `simd<double, scalar>`   | 30.2174 | 0.995398
| `simd<double, __sse>`    | 15.5236 | 3.87517
| `simd<double, __avx>`    | 16.9682 | 7.09053
| `simd<double, __avx512>` | 26.4416 | 9.1003

### Latency

| `T hypot(T, T)` | TSC cycles/call | Speedup per value relative to `T` |
| --- | --- | --- |
| `float`                  | 37.6397 |
| `simd<float, scalar>`    | 38.4039 | 0.980102
| `simd<float, __sse>`     | 41.2072 | 3.6537
| `simd<float, __avx>`     | 41.4811 | 7.25916
| `simd<float, __avx512>`  | 53.6256 | 11.2304
| `double`                 | 51.806  |
| `simd<double, scalar>`   | 51.8265 | 0.999604
| `simd<double, __sse>`    | 42.0938 | 2.46145
| `simd<double, __avx>`    | 42.6324 | 4.86072
| `simd<double, __avx512>` | 57.4461 | 7.21455

I can also disable the Annex F requirements via `-ffast-math`. Let's see how 
that changes the results:

### Throughput (*-ffast-math*)

| `T hypot(T, T)` | TSC cycles/call | Speedup per value relative to `T` |
| --- | --- | --- |
`float`                 | 12.0874 |
`simd<float, scalar>`   | 11.572  | 1.04453
`simd<float, __sse>`    | 11.0433 | 4.37815
`simd<float, __avx>`    | 11.6043 | 8.33306
`simd<float, __avx512>` | 13.6978 | 14.1189
`double`                | 27.1595 |
`simd<double, scalar>`  | 26.0718 | 1.04172
`simd<double, __sse>`   | 11.6566 | 4.65993
`simd<double, __avx>`   | 13.2002 | 8.23001
`simd<double, __avx512>`| 25.5888 | 8.49107

### Latency (*-ffast-math*)

| `T hypot(T, T)` | TSC cycles/call | Speedup per value relative to `T` |
| --- | --- | --- |
`float`                 | 36.8928 |
`simd<float, scalar>`   | 37.1042 | 0.994301
`simd<float, __sse>`    | 41.357  | 3.56822
`simd<float, __avx>`    | 41.7432 | 7.07042
`simd<float, __avx512>` | 52.2087 | 11.3062
`double`                | 51.3563 |
`simd<double, scalar>`  | 50.9219 | 1.00853
`simd<double, __sse>`   | 42.4579 | 2.41916
`simd<double, __avx>`   | 42.326  | 4.85341
`simd<double, __avx512>`| 56.3338 | 7.29314
