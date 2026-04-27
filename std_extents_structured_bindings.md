---
title: "Structured bindings for `std::extents`"
document: P2906R1
date: today
audience:
  - Library Evolution Working Group
author:
  - name: Bernhard Manfred Gruber
    email: <bernhardmgruber@gmail.com>
  - name: Yihan Wang
    email: <yihwang@nvidia.com>
toc: true
toc-depth: 2
---

# Changelog

## R1

- Since [@P1061R10] has been accepted, update the motivation of this proposal.
- Update the Goldbot implementation.
- Complete the proposal wording.

## R0

Initial revision

# Motivation and Scope

[@P0009R18] proposed `std::mdspan`, which was approved for C++23.
It comes with the utility class template `std::extents` to describe the integral extents of a multidimensional index space.
Practically, `std::extents` models an array of integrals, where some of the values can be specified at compile-time.
However, `std::extents` behaves very little like an array.
A notable missing feature are structured bindings, which would come in handy if the extents of the individual dimensions need to be extracted:

::: cmptable

### Before
```cpp
std::mdspan<double,
    std::extents<I, I1, I2, I3>, L, A> mdspan;
const auto& e = mdspan.extents();
for (I z = 0; z < e.extent(2); z++)
    for (I y = 0; y < e.extent(1); y++)
        for (I x = 0; x < e.extent(0); x++)
            mdspan[z, y, x] = 42.0;

const auto total =
    e.extent(0) * e.extent(1) * e.extent(2);
```

### After
```cpp
std::mdspan<double,
    std::extents<I, I1, I2, I3>, L, A> mdspan;
const auto& [depth, height, width] = mdspan.extents();
for (I z = 0; z < depth; z++)
    for (I y = 0; y < height; y++)
        for (I x = 0; x < width; x++)
            mdspan[z, y, x] = 42.0;

const auto total =
    width * height * depth;
```

:::

Comparing before and after, the usability gain with structured bindings alone is marginal,
but it allows us to use descriptive names for the extents to improve readability.

The proposed feature becomes even more powerful in combination with [@P1061R10] (adopted into C++26),
which allows structured bindings to introduce a pack:

```cpp
std::mdspan<double, std::extents<I, Is...>, L, A> mdspan;
const auto& [...es] = mdspan.extents();
for (const auto [...is] : std::views::cartesian_product(std::views::iota(0, es)...))
    mdspan[is...] = 42.0;

const auto total = (es * ...);
```

In this example, we trade readability for generality.
Destructuring the extents into a pack allows us to expand the extents again into a series of `iota` views,
which we can turn into the index space for `mdspan` using a `cartesian_product`.
Notice that the implementation is rank agnostic,
and for `std::layout_right` (`std::mdspan`'s default) iterates contiguously through memory.
Under the proposed design (see [](#handling-static-extents)),
the pack `es` is heterogeneous: each element is either an `IndexType` or a `std::constant_wrapper`,
so when every extent is static the fold `(es * ...)` itself yields a `std::constant_wrapper`
and `total` is a compile-time constant.

# Impact On the Standard

This is a pure library extension.
Destructuring `std::extents` in the current specification [@N4944] is ill-formed,
because `std::extents` stores its runtime extents in a private non-static data member, which is inaccessible to structured bindings.

# Design Decisions

## Handling static extents {#handling-static-extents}

When destructuring `std::extents` we have to decide how to expose static extents to the user.
This paper proposes to **retain the compile-time nature of static extents** by exposing them as `std::constant_wrapper` instances ([@P2781R9]),
while dynamic extents are exposed as plain values of `IndexType`.

This design has two important properties:

- It preserves information the user already gave to the compiler. Demoting static extents to runtime values would be lossy,
  with no way for user code to recover the compile-time nature afterwards.
- `std::constant_wrapper` has a non-explicit conversion to its `value_type` and overloads the usual arithmetic and comparison operators,
  so static extents transparently degrade to runtime values whenever used as such.
  As a result, the structured-binding examples in [](#motivation-and-scope) work unchanged regardless of whether each extent is static or dynamic.

The cost is that destructured extents are heterogeneously typed: some bindings are values of `IndexType`, others are `constant_wrapper` specializations.
In the rare cases where uniform typing is required, users can apply an explicit cast or use `std::extents::extent(I)` directly.
The alternative of always demoting static extents to runtime values is described in [](#alternative-considered-demote-static-extents-to-runtime-values);
it is simpler to specify but discards information the compiler already has.

## Modification of extents

Modifications of the values stored inside a `std::extents` should not be allowed,
since it is neither possible in case of a static extent
nor does it follow the design of `std::extents::extent(rank_type) -> index_type`, which returns by value.

# Proposed implementation

The proposed implementation uses the tuple interface and queries the extents type whether a specific extent is static or dynamic.
Depending on this, `get` returns either the runtime extent via `std::extents::extent(I)`, or a `std::constant_wrapper` whose value is the static extent
cast to `IndexType`:

```cpp
namespace std {
    template <size_t I, typename IndexType, size_t... Extents>
    constexpr auto get(const extents<IndexType, Extents...>& e) noexcept {
        if constexpr (extents<IndexType, Extents...>::static_extent(I) == dynamic_extent)
            return e.extent(I);
        else
            return constant_wrapper<
                static_cast<IndexType>(
                    extents<IndexType, Extents...>::static_extent(I))>{};
    }
}

template <typename IndexType, std::size_t... Extents>
struct std::tuple_size<std::extents<IndexType, Extents...>>
    : std::integral_constant<std::size_t, sizeof...(Extents)> {};

template <std::size_t I, typename IndexType, std::size_t... Extents>
struct std::tuple_element<I, std::extents<IndexType, Extents...>> {
    using type = decltype(std::get<I>(std::declval<std::extents<IndexType, Extents...>>()));
};
```

An example of such an implementation using the Kokkos reference implementation of `std::mdspan` on Godbolt is provided here:
[https://godbolt.org/z/5srxdEEna](https://godbolt.org/z/5srxdEEna).

# Alternative considered: demote static extents to runtime values {#alternative-considered-demote-static-extents-to-runtime-values}

A simpler alternative is to delegate `get` directly to `std::extents::extent(rank_type)`,
which yields a uniform `IndexType` for every binding regardless of whether the corresponding extent is static or dynamic:

```cpp
namespace std {
    template <size_t I, typename IndexType, size_t... Extents>
    constexpr IndexType get(const extents<IndexType, Extents...>& e) noexcept {
        return e.extent(I);
    }
}

template <typename IndexType, std::size_t... Extents>
struct std::tuple_size<std::extents<IndexType, Extents...>>
    : std::integral_constant<std::size_t, sizeof...(Extents)> {};

template <std::size_t I, typename IndexType, std::size_t... Extents>
struct std::tuple_element<I, std::extents<IndexType, Extents...>> {
    using type = IndexType;
};
```

This alternative is simpler to specify, but it discards the compile-time information that the user encoded in the static extents,
with no way for user code to recover it.
An example using the Kokkos reference implementation of `std::mdspan` on Godbolt is provided here:
[https://godbolt.org/z/o76nr4aa9](https://godbolt.org/z/o76nr4aa9).

# Polls

The authors would like to seek guidance on:

- Whether structured bindings for `std::extents` are perceived as useful and whether to continue with this proposal.
- Whether LEWG agrees with retaining static extents as `std::constant_wrapper` instances (the proposed design),
  rather than the alternative of demoting them to runtime values.

# Wording

## Header `<mdspan>` synopsis [mdspan.syn]{.sref}

Modify the header `<mdspan>` synopsis in [mdspan.syn]{.sref} as follows:

```cpp
 namespace std {
   // ...

   // [mdspan.extents.dims], alias template dims
   template<size_t Rank, class IndexType = size_t>
     using dims = $see below$;

  @@[`// [mdspan.extents.tuple], tuple interface`]{.add}@@
  @@[`template<class T> struct tuple_size;`]{.add}@@

  @@[`template<size_t I, class T> struct tuple_element;`]{.add}@@

  @@[`template<class IndexType, size_t... Extents>`]{.add}@@
    @@[`struct tuple_size<extents<IndexType, Extents...>>;`]{.add}@@

  @@[`template<size_t I, class IndexType, size_t... Extents>`]{.add}@@
    @@[`struct tuple_element<I, extents<IndexType, Extents...>>;`]{.add}@@

  @@[`template<size_t I, class IndexType, size_t... Extents>`]{.add}@@
    @@[`constexpr tuple_element_t<I, extents<IndexType, Extents...>>`]{.add}@@
      @@[`get(const extents<IndexType, Extents...>& e) noexcept;`]{.add}@@

   // ...
 }
```

## Tuple interface [mdspan.extents.tuple]

Insert a new subclause at the end of [mdspan.extents]{.sref},
immediately after [mdspan.extents.dims]{.sref}:

::: add

**23.7.3.3.8 Tuple interface [mdspan.extents.tuple]**

```cpp
template<class IndexType, size_t... Extents>
struct tuple_size<extents<IndexType, Extents...>>
  : integral_constant<size_t, sizeof...(Extents)> { };

template<size_t I, class IndexType, size_t... Extents>
struct tuple_element<I, extents<IndexType, Extents...>> {
  using type = conditional_t<
    extents<IndexType, Extents...>::static_extent(I) == dynamic_extent,
    IndexType,
    constant_wrapper<
      static_cast<IndexType>(
        extents<IndexType, Extents...>::static_extent(I))>>;
};
```

[1]{.pnum} *Mandates*: `I < sizeof...(Extents)` is `true`.

```cpp
template<size_t I, class IndexType, size_t... Extents>
constexpr tuple_element_t<I, extents<IndexType, Extents...>>
  get(const extents<IndexType, Extents...>& e) noexcept;
```

[2]{.pnum} *Mandates*: `I < sizeof...(Extents)` is `true`.

[3]{.pnum} *Returns*:

  - `e.extent(I)`, if `extents<IndexType, Extents...>::static_extent(I) == dynamic_extent`;
  - otherwise, `constant_wrapper<static_cast<IndexType>(extents<IndexType, Extents...>::static_extent(I))>{}`.

:::

## Feature-test macro

Modify the header `<version>` synopsis in [version.syn]{.sref} by adding the following macro definition,
with the value selected by the editor to reflect the date of adoption of this paper:

```diff
+ #define __cpp_lib_extents_structured_bindings 20XXXXL // also in <mdspan>
```

# Acknowledgements

I would like to thank Mark Hoemmen, Christian Trott, Wenming Wang, and June Yang for reviewing and encouraging us to write this proposal.

# References {.unnumbered}

::: {#refs}
:::
