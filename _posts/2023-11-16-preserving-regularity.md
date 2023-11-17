---
layout: post
title: "The story of regularity and std::simd"
tags: [C++, SIMD, "std::simd", regularity, concepts]
author: Matthias Kretz
---

In this post I will talk about regularity and why 
`std::regular<std::simd<int>>` needs to be `false` in order to preserve 
regularity at the level where it matters: equational reasoning. The issue of 
regularity came up repeatedly when discussing the design of `std::simd` for 
C++26. (It also came up in 2017 for `std::experimental::simd`.)  My goal for 
this post is the exploration of options and their consequences. There's a lot 
more to be said, but this post is already too long. In any case, when talking 
about regularity, we need start with "Elements of Programming", the book that 
introduced the concept:
> A type is *regular* if and only if its basis includes equality, assignment, 
destructor, default constructor, copy constructor, total ordering, and 
underlying type. [...]
>
> Algorithms are *abstract* when they can be used with different models 
satisfying the same requirements, such as associativity. Code optimization 
depends on equational reasoning; unless types are known to be regular, few 
optimizations can be performed.
<div style="text-align: right">
Alexander Stepanov, Paul McJones — Elements of Programming (EoP)
</div>

* * *

In other words we're looking at operations and their semantics on types. For 
example two objects `a` and `b` of type `int` can be compared (`a == b`), 
assigned (`a = b`), destructed (trivial), default constructed (`int a{};`), 
copy constructed `int a{b}` and there exists a total order defined by `a < b` 
(the latter is not included in `std::regular`). These are pretty basic 
guarantees, which we take for granted for builtin types of the language.

## `std::simd` is meant to be used in equations

This is where the confusion about `simd<int>` begins. `simd<int>` is certainly 
meant to be used in equations. Why doesn't that imply `std::regular<simd<int>>` 
must be `true`? Short answer: because the equations apply to the `int` not the 
`simd<int>`.

Let me elaborate. Stepanov and McJones define e.g. *associativity* as *op(op(a, 
b), c) = op(a, op(b, c))*. Without the ability to test for equality, 
associativity cannot even be defined / tested. Nevertheless, intuitively (i.e. 
without considering any definitions — using our intuition from what we learned 
in school) everybody would likely identify `simd<int>` as associative. Why is 
that? Because we recognize that `(a + b) + c` yields the same result as
`a + (b + c)`. Hmm "same result"... How can I say that without regularity?

When writing an equation using `simd<int>` types, the mathematically important 
equation is the one expressed per element. The `simd<int>` object as a whole 
does not identify a single *entity*¹; a `simd` type is not a *value type*².

The `simd` abstraction is a tool for expressing the same equation on multiple 
`int` objects in parallel. It's a tool for manipulating multiple entities — a 
tool for applying operators/operations on multiple entities in parallel. 
Therefore, equational reasoning must work on the level of `int`s. Thus, in 
order to prove associativity of a single SIMD lane of `int`s, equality must be 
testable on a per-element basis. This is exactly what the status quo `simd` 
paper does. It therefore implements regularity for the *value type*. The 
alternative implies that the comparison result of a different SIMD lane 
(unrelated element) influences the result of other elements.

## What if `std::regular<std::simd<T>>` were `true`?

To answer this question, consider the basic code transformation underlying the 
`std::simd` design. For this purpose let's look at an example that involves the 
inequality operator in an equation. (The need for `<`/`<=`/`>=`/`>` in the 
implementation of equations is a lot more common. However, since `std::regular` 
requires only equality I wanted to keep the example on point with using `!=`.) 
In the example we will consider a simple character transformation function for 
scrambling/unscrambling text.

The design intent of `std::simd` is to replace e.g. a loop over 16 `char`s with 
one `simd<char, 16>` chunk:

```c++
    for (int i = 0; i < 16; ++i) {  // 16 times do:
      char x = text[i];             //   load 1 character
      x = unscramble(x);            //   unscramble 1 character
      text[i] = x;                  //   store 1 character
    }
```
becomes:
```c++
    std::simd<char, 16> x(text.begin()); // load 16 characters
    x = unscramble(x);                   // unscramble 16 characters
    x.copy_to(text.begin());             // store 16 characters
```

where `unscramble` could be:
```c++
auto unscramble(auto x) {
  // always flip bit 0b0010;
  // flip bit 0b0001 if x has bit 0b1000 or 0b0100 set (the bool result from
  // the comparison implicitly converts to an integer with value 0 or 1)
  return x ^ (2 | ((x & 0b1100) != 0));
}
```

Let's now embed either the loop or the `simd<char, 16>` solution into a simple 
`main` function:
```c++
int main() {
    std::string text = "pfdvocp\"fmwjwjfq\n";
#if USE_LOOP
    for (int i = 0; i < 16; ++i) {
      char x = text[i];
      x = unscramble(x);
      text[i] = x;
    }
#else
    std::simd<char, 16> x(text.begin());
    x = unscramble(x);
    x.copy_to(text.begin());
#endif
    std::cout << text;
}
```

- With `USE_LOOP=1` the program prints **"regular entities"**.

- With `USE_LOOP=0`:

  - If `simd::operator==` returns `bool` (i.e. `true` only if all elements are 
    equal; and `!=` is `true` if any element compares not equal), then the 
    second program prints **"segul\`s!entitier"**.

  - However, if `operator==` is not reduced to a single `bool`, returning a 
    `simd_mask`, then the second program prints **"regular entities"** (status 
    quo of the current "merge `simd` from the TS" paper).

Making `simd<char>` itself regular breaks equational reasoning on the level of 
the `simd` elements (i.e. breaking regularity of `char`). Making `simd<char>` 
regular at the level of the `simd<char>` elements enables generic code, 
composition, equational reasoning, and thus allows applying equational 
optimizations developed for the element type.

## Proposed alternative: no comparison operators for `simd`

If `simd::operator==` returning `bool` is such a foot-gun in terms of silently 
introducing logic errors into the code, and `simd::operator==` returning 
`simd_mask` is considered unacceptable (because `operator==` should only ever 
return `bool` and nothing else), then why not remove all comparison operators 
and use free functions instead? For example, instead of `a == b` write 
`std::simd_equal(a, b)`?

While, technically, this seems like a workable compromise, it still breaks 
regularity of the value types contained in the `simd`. Consequently, according 
to the definition of associativity, `std::simd<int>` would not be associative 
anymore: there is no equality operator to establish equality of the LHS and 
RHS.

Furthermore, this breaks generic code. This compromise is going against the 
basic design principle of `simd` that, as much as the language allows, 
vectorization of an algorithms should only require a change of type from `T` to 
`simd<T>`. Now, generic code — i.e. also code that doesn't use `simd` — needs 
to be written with `std::simd_equal` in place of `==`. The next logical step 
would then replace `simd_mask::operator&&` with `std::simd_logic_and`. And the 
generic overload for `bool` arguments would then have to break 
short-circuiting.

It has been stated that `simd` should not overload any operators at all, 
because `==` isn't regular (or because `simd` itself isn't a *value type*). 
It's probably a solution in the sense that `std::simd` adoption would simply 
not happen and thus not ever confuse anybody. 😉

## Conclusion
We have to choose one of the following:

1. `std::regular<std::simd<T>>`: Making `simd` itself regular silently breaks 
   generic code, because equational reasoning on the level of the 
   `simd::value_type` is broken.

2. No `operator==` overloads at all: Not implementing any comparison operators 
   makes generic code dependent on a different syntax for equational reasoning.

3. Regularity of the `simd` elements: Keeping regularity of `simd::value_type` 
   intact breaks the use cases where users want to use `simd` as a product type 
   (where `simd` itself is a *value type*). However, it allows using `simd` as 
   a tool for expressing parallel evaluation of equations that can be reasoned 
   about on the `simd::value_type` level using the concepts introduced in 
   "Elements of Programming".

In most cases, using `std::simd` as a product type is *wrong* in the sense that 
it doesn't help with expressing data-parallelism for higher performance. In 
many cases the use of `std::simd` as a product type trades the use of SIMD 
instructions for ILP (instruction level parallelism), resulting in a slowdown 
rather than performance improvement. Therefore, the `std::simd` API tries to 
discourage such uses. Consequently, `std::regular<std::simd<T>>` must be 
`false`.

## Footnotes

¹.
> An *abstract entity* is an individual thing that is eternal and unchangeable, 
while a *concrete entity* is an individual thing that comes into and out of 
existence in space and time. [...] Blue and 13 are examples of abstract 
entities. Socrates and the United States of America are examples of concrete 
entities. [...]
>
> An *abstract species* describes common properties of essentially equivalent 
abstract entities. Examples of abstract species are natural number and color. A 
*concrete species* describes the set of attributes of essentially equivalent 
concrete entities. Examples of concrete species are man and U.S. state.
<div style="text-align: right">
Alexander Stepanov, Paul McJones — Elements of Programming (EoP)
</div>

².
> A *datum* is a finite sequence of 0s and 1s. A *value type* is a 
correspondence between a species (abstract or concrete) and a set of datums. 
<div style="text-align: right">
Alexander Stepanov, Paul McJones — Elements of Programming (EoP)
</div>
