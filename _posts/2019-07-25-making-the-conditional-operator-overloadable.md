---
layout: post
title: "Making the C++ conditional operator overloadable"
tags: [C++, SIMD, "std::simd", GCC]
author: Matthias Kretz
---

Why are `operator?:` overloads not allowed in C++? See, basically every 
operator in C++ is overloadable. Sure, you can do stupid things with such 
power, but that's a general problem with humans that have power. C++ gives 
power to programmers, we need to use it wisely. Apparently `operator?:` is not 
overloadable because: “There is no fundamental reason to disallow overloading 
of `?:`. I just didn't see the need to introduce the special case of 
overloading a ternary operator. Note that a function overloading 
`expr1?expr2:expr3` would not be able to guarantee that only one of `expr2` and 
`expr3` was executed.” \[[Stroustrup: C++ Style and Technique 
FAQ](http://www.stroustrup.com/bs_faq2.html#overload-dot)\]

## My motivation

So why would one want to overload `?:`? And what happened that it became 
important now? Consider this simple absolute value function (yes, please ignore 
`-0.`):
```cpp
template <class T> T abs(T x) {
  return x < 0 ? -x : x;
}
```
Instantiating `abs` with `int` or `float` works just fine. How about 
`abs<simd<int>>`? Operators on `simd` are all defined to act element-wise. 
Consequently, `x < 0` produces `simd<int>::size()` booleans stored in a 
`simd_mask<int>`. Now if only the conditional operator would apply 
element-wise. That would be consistent. And we'd easily be able to write 
generic code that works for builtin and `simd` types. But wait, someone in the 
GCC team thinks so, too:
```cpp
using V [[gnu::vector_size(16)]] = int;
V f(V x) {
  return abs(x);
}
```
This compiles and produces perfect machine code. Can I has nice things, too? 
No, I'd have to define
```cpp
template <typename T, typename Abi>
simd<T, Abi> operator?:(simd_mask<T, Abi> cond, simd<T, Abi> a, simd<T, Abi> b) {
  where(cond, b) = a;
  return b;
}
```
in the `simd` library. But `?:` is not overloadable.

## Extending C++

So I wrote [P0917](https://wg21.link/P0917). Because as [Herb Sutter 
wrote](https://wg21.link/P0745R0):
* “Be consistent: Don’t make similar things different, including in spelling, 
  behavior, or capability. Don’t make different things appear similar when they 
  have different behavior or capability.”
* “Be orthogonal: Avoid arbitrary coupling. Let features be used freely in 
  combination.”
* “Be general: Don’t restrict what is inherent. Don’t arbitrarily restrict a 
  complete set of uses. Avoid special cases and partial features.”

`?:` not being overloadable fails those design principles. (Orthogonality in 
this case is about not *requiring* user-defined `operator?:` to be lazy 
evaluated. That's an orthogonal problem that `operator&&` and `operator||` 
could use a solution for, just as many other use cases. And 
[P0927](https://wg21.link/P0927) wants to solve it for all functions.)

The main use cases for user-defined `operator?:` are:
* Numeric types (currently often specializing `std::common_type`), e.g. 
  [BOUNDED_CONDITIONAL for 
  bounded::integer](http://doublewise.net/c++/bounded/reference/)
* Expression templates, e.g. 
  [boost::yap](https://boostorg.github.io/yap/doc/html/BOOST_YAP_USER_EX_idm15635.html)
* Blend operations, e.g. blending SIMD vectors as shown above.
* *If you have a use case that I did not cover, please let me know!* I want to 
  make sure it is considered in the standardization process (“be general”).

## Extending GCC

After P0917 was discussed in the C++ EWG Incubator in Cologne, I wanted to know 
what it would take for a compiler to implement my proposal. I asked Jason, who 
told me that the conditional operator actually had been overloadable at one 
point in the history of GCC. So I dived into the GCC history and found that the 
feature was removed before the GCC 2.95 release date. So maybe egcs 1.1.2 had 
it. That's a long time ago... Let's ignore the distant past.

With the help if Iain I was able to understand enough about the GCC frontend 
code to define and call user-defined `operator?:` overloads. The first positive 
surprise was that I only needed to fix a minor parser omission (GCC didn't 
recognize `operator?:` as an operator overload and expected to see a cast 
operator) and comment turn an error message into a `return true` and I was able 
to define conditional operators. When disassembling the resulting object file 
(with C++ demangling enabled) the operator was actually shown as `operator?:`. 
So the mangling for the operator has been defined all this time, just waiting 
for C++ to finally make it overloadable. ;-)

If you want to try it out, clone my [patched GCC 9 
branch](https://github.com/mattkretz/gcc/tree/mkretz/overload_ternary) and 
build GCC following the standard build instructions.

## Putting simd and `operator?:` overloading together

Of course I had to try it out. My test case:
```cpp
#include <experimental/simd>

namespace std
{
namespace experimental
{
inline namespace parallelism_v2
{
template <typename _From, typename _To,
          typename = std::enable_if_t<std::is_convertible_v<_From, _To>>>
using _Convertible = _From;

#ifdef __GXX_CONDITIONAL_IS_OVERLOADABLE__
template <class T, class Abi>
simd<T, Abi> operator?:(simd_mask<T, Abi> k, simd<T, Abi> a, simd<T, Abi> b)
{
  where(k, b) = a;
  return b;
}
template <class T, class Abi, class U>
simd<T, Abi> operator?:(simd_mask<T, Abi> k,
                        _Convertible<U, simd<T, Abi>> a, simd<T, Abi> b)
{
  where(k, b) = a;
  return b;
}
template <class T, class Abi, class U>
simd<T, Abi> operator?:(simd_mask<T, Abi> k,
                        simd<T, Abi> a, _Convertible<U, simd<T, Abi>> b)
{
  where(!k, a) = b;
  return a;
}
#endif  // __GXX_CONDITIONAL_IS_OVERLOADABLE__

}  // namespace parallelism_v2
}  // namespace experimental
}  // namespace std

namespace stdx = std::experimental;

using V = stdx::native_simd<int>;

#ifdef __GXX_CONDITIONAL_IS_OVERLOADABLE__
V f(V x) { return x < 0 ? 0 : x; }
V g(V x) { return x < 0 ? x : 0; }
#endif  // __GXX_CONDITIONAL_IS_OVERLOADABLE__
```

My patched GCC compiles this into:
```
f(std::experimental::parallelism_v2::simd<int, std::experimental::parallelism_v2::simd_abi::_AvxAbi<32> >):
  vmovdqa  ymm1, ymm0
  vpsrad   ymm0, ymm0, 31
  vpandn   ymm0, ymm0, ymm1
  ret
g(std::experimental::parallelism_v2::simd<int, std::experimental::parallelism_v2::simd_abi::_AvxAbi<32> >):
  vmovdqa  ymm1, ymm0
  vpsrad   ymm0, ymm0, 31
  vpand ymm0, ymm0, ymm1
  ret
```

Nice. I want. (Except for the missed optimization for `x < 0 ? 0 : x`, which 
should use [`vpminsd`](https://www.felixcloutier.com/x86/pminsd:pminsq).)

Discuss on [Hacker News](https://news.ycombinator.com/item?id=) or 
[Reddit](https://www.reddit.com/r/cpp/comments/)
