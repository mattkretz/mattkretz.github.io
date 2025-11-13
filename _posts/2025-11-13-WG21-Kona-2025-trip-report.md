---
layout: post
title: "Virtual Trip Report: WG21 Kona 2025"
tags: [C++, SIMD, "std::simd", WG21, constant_wrapper]
author: Matthias Kretz
---

After the CD (committee draft) is out, WG21 is in bug fixing mode. Consider NB 
comments as the tickets that force us to consider a resolution and provide an 
answer. Such issues can either be design issues or wording issues (where 
wording does not match stated design intent). Besides NB comments, many library 
issues had been filed and prioritized independently, which led to several 
categorized as must/should fix before the standard is finalized.

The first week of November, I should have been in Kona, Hawaii for our first of 
two NB comments resolving C++ committee meetings. However, with the current 
political situation in the US, I preferred to attend remotely. Remote 
participation meant to work via Zoom from 19:00–04:00. Since we also had Covid 
at home it was tough, but somehow I survived. 😉 But let's get to the technical 
bits of this post …

## `std::simd` updates

For `simd` we had a few issues that fell out of making `complex` vectorizable. 
But also `constructible_from<float16_t, float>` being `false` was something 
that was not reflected in the wording and broke design intent. Summary of 
changes:

Given:

```c++
std::simd::vec<float> vf;
std::span<std::complex<float>> range_of_complex;
std::span<std::float16_t> range_of_float16;
```

| code | CD | after Kona |
|------|----|------------|
| `convertible_to<array<string, 4>, vec<int, 4>>` | `true` | `false` |
| `convertible_to<vec<complex<float>, 4>, vec<float>, 4>` | `true` | `false` |
| `vec<float16_t>(1.f)` | ill-formed | well-formed (explicit conversion) |
| `unchecked_load<vec<float>>(range_of_complex, flag_convert)` | instantiation failure | `static_assert` |
| `unchecked_store(v, range_of_float16, flag_convert)` | no viable overload found | well-formed |

---

The return type of unary plus/minus on `basic_mask` was unimplementable for me 
after I added `complex` support. This was fixed by relaxing the wording on the 
return type. At the same time this made it harder for users to explicitly spell 
the best integral `vec` type when implementing branchless algorithms. So a 
deduction guide from `basic_mask` to `basic_vec` was added:

```c++
basic_vec zero_or_one = vf > 0.f;
```

The code above computes a mask when comparing multiple `float` values against 
`0.f`, which in turn is converted to a `basic_vec<int, Abi>` (where `Abi` is 
determined by the implementation) with values `0` or `1`. This is analogous to 
conversion from `bool` to `int`.

---

Then we fixed an inconsistency with regard to the `complex` API:
```c++
complex<double> c = {};
c.real(1.f); // OK

simd<complex<double>> sc = {};
sc.real(simd<float, sc.size()>(1.f)); // was ill-formed, now allowed
```

This same issue resolution also enables conversions on `vec<complex<T>>` 
construction:
```c++
simd<complex<double>> sc0(1., 1.); // #1
simd<complex<double>, 4> sc1(simd<double, 4>(1.), simd<float, 4>(1.f)); // #2
```

The conversion from `double` to `vec<double>` for the constructor call in `#1` 
is now possible, as is the conversion from `simd<float, 4>` to `simd<double, 
4>` in `#2`.

---

We also agreed that, for now, users should not specialize any template in 
`std::simd`. This seems to be necessary to keep the design space open for 
making user-defined types vectorizable. The wording for this still needs to be 
done and voted into the DIS.
Furthermore, the [missing reduction overloads for enabling simd-generic 
code](https://wg21.link/p3690), which were on track for C++26 at the last 
meeting but didn't make the cut were reconfirmed to be added to C++26.

---

The maybe biggest and important change, however, was my comment to restore some 
compatibility with the TS interface (`std::experimental::simd`). Consider the 
status quo of the CD:

```c++
template <simd_floating_point V>
V f(V x)
{ return x + 1; }

f(vec<double>()); // OK
f(vec<float>()); // ill-formed
f(vec<std::float16_t>()); // ill-formed
```

Since the conversion from `int` to `float`/`float16_t` isn't value-preserving, 
the implicit constructor call to `basic_vec` is not allowed. This is 
frustrating, because the compiler can actually see the value, not just the type 
and determine that converting `1` to `float16_t` is actually not changing the 
value. Consequently, it should be allowed. But arbitrary conversion from `int`, 
or values that do change should be ill-formed. With P3844 applied we get:

```c++
template <simd_floating_point V>
V f(V x) {
  return x + 0x5EAF00D;
}

f(vec<double>()); // OK
f(vec<float>()); // ill-formed
f(vec<std::float16_t>()); // ill-formed
```

This is implemented via a `consteval` broadcast constructor overload that can 
halt the compilation process if the value changes on conversion. As a 
consequence `simd` code will require fewer explicit conversions, which can be a 
source of hard to find bugs.

Another final example:
```c++
simd::vec<float> x = ...;
pow(x, 3); // x³
```
This is okay in the TS, ill-formed with the CD wording, and will be well-formed 
again with P3844.

## SG6 Numerics

Lisa and I chaired SG6 for one and a half days. We had relatively large 
attendance, which is great! 🎉

We looked at 5 integer related papers on Wednesday. Topics included bit
operations, rounding options for division, and bit-precise integers. All 5 of
the papers were forwarded.

On Thursday we switched gears to complex and real numbers. 4 papers got
feedback and need to return to us. A paper on `constexpr` floating-point
`<charconv>` was forwarded.

### Personal take on `_BitInt`

I am not happy about `_BitInt`. C made it a relatively small extension by just 
adding a new fundamental type with almost identical behavior to existing 
integer types. The only difference is that `_BitInt` doesn't promote to `int`. 
But in terms of library functions … virtually nothing.

Now C++ comes along and needs to restore C compatibility, adopting millions of 
new fundamental types (signed and unsigned `_BitInt(N)`, where `N` is allowed 
to be huge). Our standard library is bigger, and more importantly generic. Now 
consider that every interface that allows deduction of an integer type now also 
allows `_BitInt`. That's an explosion of the API surface. Even more troubling 
are simple functions such as:
```c++
template <typename T>
  T abs(T);
```
If this isn't constrained to "reasonable" `_BitInt(N)` sizes then the main cost 
of this function is just `memcpy`. I'm convinced that large `_BitInt(N)` needs 
different interfaces (e.g. modify in-place) and rarely — if ever — needs 
by-value passing.

The other issue I take with `_BitInt` is behavior. There are so many complaints 
about how unsafe integers in C and C++ behave. E.g. `-1 < 1u` is `false`. And 
`_BitInt` just piles onto that. C didn't have to do it like this. But C++ now 
has little choice other than to stop basing on top of C. Except … we can 
incorporate `_BitInt` and then tell users "here's another feature we have for 
reasons outside of our control; just don't use it". At the same time we add our 
own type that has less troubling semantics and is spelled `std::bit_int<N>`. 
Such as this [implementation](https://github.com/mattkretz/bit_int/) 
([demo](https://compiler-explorer.com/z/TeYKP5E89)):

```c++
bool bad(_BitInt(8) a, unsigned _BitInt(8) b) {
  // wrong unless a >= 0 / b <= 127
  return a == b;
}

bool good(bit_int<8> a, bit_uint<8> b) {
  // always correct
  return a == b;
}
```

But it seems like that won't happen unless I do the work and write the 
necessary papers. It's not like I'm already overworked with `std::simd`. 😰

## `constant_wrapper`, `function_ref`, and `nontype`/`constant_arg`

The discussion about the `function_ref(nontype_t<fn>)` constructor went on 
between meetings and seemed to come to a conclusion in Kona. However, I did and 
still do not understand the concern with using `constant_wrapper` instead of 
`nontype_t` (or `constant_arg_t` as will be called in the DIS if nothing else 
happens). Up to now I assumed this was due to me not understanding 
`function_ref` … I finally took the time to ensure I do understand it.

### Background

```c++
// libfoo.h:
int foo(std::function_ref<int(float)>);

// app.cpp:
void something() {
  // stores lambda object -> indirect call:
  foo([](float x) { return int(x * 0.5); });

  // nullptr for function -> direct call:
  foo(std::constant_arg<[](float x) { return int(x * 0.5); }>);
}
```

The `constant_arg` class template does nothing other than work around the 
inability to pass the callable as a constant expressions (template argument). 
The `function_ref` constructor can optimize, if the given callable is a 
constant expression. That's why this is useful.

However, in C++26 we also added `std::constant_wrapper` / `std::cw`, which is 
first and foremost a tool for *passing constant expressions as function 
arguments*. So if `std::cw<[](float x) { return int(x * 0.5); }>` is the 
standard library's vocabulary to pass a constant expression, then why does 
`function_ref` use a different type to do the exact same thing?

### Fear, Uncertainty, and Doubt?

The issue revolves around two (perceived) issues:

1. `constant_wrapper` implements *all* operators itself *in the type space*. So 
   `std::cw<1> + std::cw<2>` has the same type as `std::cw<3>`. And 
   `std::cw<[](int x) { return x + 1; }>(std::cw<2>)` thus also is 
   `std::cw<3>`. This feature was initially motivated by UDLs, which require a 
   unary minus operator. But as we realized with `integral_constant`, adding 
   such an operator is a breaking change. Consequently, any operator that we 
   don't add to `constant_wrapper` now is a breaking change if we add it later. 
   So we added everything. Because you can use such operators for a DSL or some 
   other (crazy) meta programming. But these operators need to be understood as 
   a niche feature. And it's a feature that isn't easy to trigger accidentally.

2. `constant_wrapper` implements a conversion operator to the type that it 
   holds. Thus `std::cw<1> + 2` is `3` due to calling builtin `operator+(int, 
   int)`. Notably, `constant_wrapper` does not need to implement `operator+` 
   for this to work. But it does opt in to ADL for the type that it holds in 
   order to find more operators on name lookup. Sadly, the language is not 
   doing enough ADL (not a popular thing to say, I know) and only finds 
   operators defined as hidden friends. Therefore `std::cw<&some_function>(1)` 
   calls `some_function(1)` but `std::cw<[](int) { ... }>(1)` is ill-formed. 
   This ability of `constant_wrapper` to already pass and use a function 
   pointer/reference as a constant expression while not doing the same for 
   lambdas is brought up as argument against `constant_wrapper`.

3. Concerns were voiced about users trying to use `cw<callable>` instead of 
   `callable` in every other function wrapper interface and get frustrated when 
   it doesn't compile. It was said that the standard library interfaces should 
   not suggest to use of `constant_wrapper` together with function wrappers.
   (Note that — right now — `constant_wrapper` can be used to optimize some 
   function wrapper uses: E.g. given `move_only_function<int(int)> 
   f0(std::cw<fn>)` and `move_only_function<int(int)> f1(fn)`, `f0(1)` needs 
   one call indirection less than `f1(1)`.)

The status quo:

- `function_ref` uses `constant_arg_t` to pass a constant expression as 
function argument
- `constant_wrapper` is a facility to pass constant expressions as function 
arguments
- `constant_arg_t` works nowhere except `function_ref`
- `constant_wrapper` works in many places (it composes)
- `function_ref(std::cw<fn>)` is equivalent to 
`function_ref(std::constant_arg<fn>)`

No explanation of this so far made sense to me. 🙁

## Closing words

Thanks to everyone in the committee for who you are and what you bring to the 
table! As always, I enjoyed the technical work and the occasional jokes. I am 
sad that I missed the face-to-face interactions: hallway discussions, going 
swimming, running, for dinner, hacking papers in the night, etc. Next time!
