---
title: "Make `std::make_from_tuple` SFINAE friendly"
document: D0000R1
date: today
audience:
  - Library Evolution
  - Library
revises: D0000R0
author:
  - name: Yihan Wang
    email: <yronglin777@gmail.com>
toc: false
---

# Introduction

This paper introduce constraints for `std::make_from_tuple`{.cpp} to make it SFINAE friendly.

# Motivation

[@LWG3528] introduce constraints:

> ```cpp
> template<class T, class Tuple, size_t... I>
>   @@[`requires is_constructible_v<T, decltype(get<I>(declval<Tuple>()))...>`]{.add}@@
> constexpr T make-from-tuple-impl(Tuple&& t, index_sequence<I...>) {     // exposition only
>   return T(get<I>(std::forward<Tuple>(t))...);
> }
> ```

When someone write SFINAE code like the following to check whether `T` can constructed from a tuple, they may hit hard errors like "no matching function for call to `make-from-tuple-impl`".

> ```cpp
> template <class T, class Tuple, class = void>
> inline constexpr bool has_make_from_tuple = false;
>
> template <class T, class Tuple>
> inline constexpr bool has_make_from_tuple<T, Tuple,
>     T, Tuple,
>     std::void_t<decltype(std::make_from_tuple<T>(std::declval<Tuple>()))>> =
>     true;
>
> struct A {
>   int a;
> };
>
> static_assert(!has_make_from_tuple<int *, std::tuple<A *>>);
>
> ```

Even If the effects are _Equivalent to_ calling a constrained function, the constraints has not apply to `std::make_from_tuple`. 

This is somehow unclear when the constraints are not literally specified with _Constraints_ in the standard wording ([structure.specifications]{.sref}).
At least _Equivalent to_ doesn't propagate every substitution failure in immediate context. In the case of `make-from-tuple-impl`, the constraints were introduced via a requires-clause but not literal
_Constraints_. Some implementors believed the requires-clause should be treated same as _Constraints_, but this is not explicitly stated.

# Impact on the Standard

This proposal is a pure library improvement.

# Implementation Experience

I've implemented this improvement in:

- `libc++`: [\[libc++\] Implement LWG3528 (make_from_tuple can perform (the equivalent of) a C-style cast)](https://github.com/llvm/llvm-project/pull/85263).
- `microsoft/STL`: [\<tuple\>: Make std::make_from_tuple SFINAE friendly](https://github.com/microsoft/STL/pull/4528).

# Proposed Wording

> Modify section 22.4.6 [tuple.apply] as indicated:
>
> ```cpp
> template<class T, tuple-like Tuple>
>  constexpr T make_from_tuple(Tuple&& t);
> ```
>
> ::: rm
> _Mandates:_ If `tuple_size_v<remove_reference_t<Tuple>>` is 1, then `reference_constructs_from_tem`-
> `porary_v<T, decltype(get<0>(declval<Tuple>()))>` is `false`.
>
> :::
>
> ::: add
>
> Let `I` be the pack `0, 1, ..., (tuple_size_v<remove_reference_t<Tuple>> - 1)`.
>
> _Constraints:_
>
> - `is_constructible_v<T, decltype(get<I>(declval<Tuple>()))...>` is `true`.
>
> - If `tuple_size_v<remove_reference_t<Tuple>>` is 1, then `reference_constructs_from_tem`-
> `porary_v<T, decltype(get<0>(declval<Tuple>()))>` is `false`.
>
> :::
>
> _Effects:_ Given the exposition-only function template:
>
> ```cpp
>
> namespace std {
>   template<class T, tuple-like Tuple, size_t... I>
>     @@[`requires is_constructible_v<T, decltype(get<I>(declval<Tuple>()))...>`]{.rm}@@
>   constexpr T make-from-tuple-impl(Tuple&& t, index_sequence<I...>) {   // exposition only
>     return T(get<I>(std::forward<Tuple>(t))...);
>   }
> }
>
> ```
>
> _Equivalent to:_
>
> ```cpp
> return make-from-tuple-impl<T>(
>            std::forward<Tuple>(t),
>            make_index_sequence<tuple_size_v<remove_reference_t<Tuple>>>{});
> ```
> [Note 1: The type of T must be supplied as an explicit template parameter, as it cannot be deduced from the argument list. - end note]{.note}

# Acknowledgements

Thank you to Jiang An, Mark de Wever, Stephan T. Lavavej, Jonathan Wakely, Barry Revzin, Daniel Krügler,
and everyone else who contributed to the discussions, and encouraged me to write this paper.
