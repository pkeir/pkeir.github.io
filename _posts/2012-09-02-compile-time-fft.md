---
layout: post
title:  "A compile-time FFT in C++11"
author: "Paul Keir"
category: C++ 
tags:
 - C++
 - templates
 - metaprogramming
---

Generalised constant expressions are a core language feature introduced to C++ by the recent C++11 standard. A call to a function or method qualified with the new `constexpr` keyword can now be evaluated at compile-time. Of course there are restrictions: arguments to such a call must also be amenable to compile-time evaluation. The primary restriction is that the body of a `constexpr` function must comprise of a single `return` statement. While this initially sounds draconian, the use of recursion, and the ternary operator, can permit essentially any calculation. The language of C++11 generalised constant expressions is that of a functional language.

The code below presents a brief example. Note the call to `sqr_int` in the second template argument to the C++11 `std::array` object. Also of interest: the call to `sqr_int` will prosaically perform floating-point calculations at compile-time. Incidentally, among other restrictions, lambda expressions may not, alas, form part of a C++11 generalised constant expression.

```cpp
#include <array>
constexpr unsigned sqr_int(double d) { return d*d; }
std::array<bool,sqr_int(4.5)> garr;
```

Having been impressed by the [ctrace](https://github.com/h3r2tic/h3r2tic.github.io/blob/master/posts/ctrace/index.md) and [metatrace](https://github.com/phresnel/metatrace) compile-time ray-tracers, I thought I'd give the fast fourier transform (FFT) a whirl. Seeking an FFT implemented in a functional language, I came across the elegant Single Assignment C (SAC) version shown [here](http://www.sac-home.org/doku.php?id=cases:nasft). After failing to compile the assembled pieces with the SAC compiler, I converted it to Fortran 9x and verified the results against another Fortran implementation from [here](http://beige.ucs.indiana.edu/B673/node13.html).

The next thing I needed was a container data type to stand in for the rank-one, first-class arrays employed by the Fortran. Thoughts of using `std::array` soon evaporated as I realised that no `constexpr` element access methods exist; neither `operator[]` nor `std::array::front()` comply. Similar issues exist with `std::tuple`. Furthermore, the FFT only requires a homogeneous container. An implementation using the heterogeneous `std::tuple` would be inclined to add additional type-checking code. In response, I created the following *recursive array* container type:

```cpp
template <typename T, size_t N>
struct recarr {
  constexpr recarr(T x, recarr<T,N-1> a) : m_x(x), m_a(a) {}

  constexpr T             head()   { return m_x; }
  constexpr recarr<T,N-1> tail()   { return m_a; }

private:
  T m_x;
  recarr<T,N-1> m_a;
};

template <typename T>
struct recarr<T,0> {};
```

The common restriction on recursive data types: members having the same type as the enclosing class must be pointers, is not applicable here; the `m_a` member is always a different type; i.e. `recarr` is not the same type as `recarr`. The zero-length specialisation of `recarr` has no members.

The list type used routinely in functional languages such as Haskell is a clear inspiration in the design of `std::recarr`. A *cons* function for `std::recarr`, necessary for the construction of higher order functions such as [map](http://zvon.org/other/haskell/Outputprelude/map_f.html) and [zipWith](http://zvon.org/other/haskell/Outputprelude/zipWith_f.html), may be defined as shown below. Rather than the raw `recarr` *constructor*, the stand-alone `recarr_cons` *function* can infer its template arguments from the runtime arguments.

```cpp
template <typename T, size_t N>
constexpr
recarr<T,N+1> recarr_cons(T x, recarr<T,N> xs) {
  return recarr<T,N+1>(x,xs);
}
```

An FFT is an operation on complex numbers. The standard `std::complex` type, however, delivers another spanner to the works; it too lacks the `constexpr` methods needed for a compile-time FFT; including the constructors, and arithmetic operators. So, a new complex number class was developed, along with the necessary C++11 generalised constant expression support. Complex number formulae and definitions were drawn from [here](https://www2.clarku.edu/faculty/djoyce/complex) and [here](http://en.literateprograms.org/Complex_numbers_(C_Plus_Plus)).

GCC and Clang both have excellent support for C++11. Wouldn't it be nice to find out which is the faster compiler for the evaluation of a `constexpr` FFT? Some more challenges materialise here: Clang rejects all of the `cmath` library functions, as being non-`constexpr`. Functions required by the FFT include `sin`, `sinh`, `cos`, `cosh`, `sqrt`, `log`, `exp`, `atan`, `atan2` and `pow`. Initially I though that Clang was being a little cranky here, but this is in fact the correct behaviour; n.b. a `constexpr` function may not be overloaded with a non-`constexpr` function; and vice-versa. Standard library functions which are to be `constexpr` must consequently be specified as such by the relevant C++ standard; the functions in `cmath` are not. After all, functions comprised of a single return statement may not be the most performant! The [N3228](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2010/n3228.html) and [N3231](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2010/n3231.html) documents from the C++ Standards Committee's [November 2010 mailings](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2010/#mailing2010-11) elaborate on this theme.

So, `constexpr` versions of the mathematical functions from `cmath` necessary for the FFT were developed; with reference to the formulae and guidelines of the [Algebra Help e-Book](http://mathonweb.com/algebra_e-book.htm).

I am in fact a big fan of the `std::tuple` class. In the end I couldn't resist a second, comparative implementation of a `constexpr` FFT using C++11 tuples; and so the magic of variadic templates. Need I mention that `std::tuple` is also missing many of the `constexpr` constructors and support functions necessary for the job? The github code linked below includes a simple recursive version of the `std::tuple` type, with `constexpr` constructors; a version of `get`; a version of `make_tuple`; and, as the FFT uses [concat](http://zvon.org/other/haskell/Outputprelude/HH_o.html), a version of `tuple_cat`. For comparison with `recarr_cons` above, here is the simple, concise `tuple_cons`; in constrast to a more verbose definition for `tuple_cat` provided in the full project.

```cpp
template <typename T, typename ...Ts>
constexpr
tuple<T,Ts...> tuple_cons(T x, tuple<Ts...> xs) {
  return tuple_cat(make_tuple(x),xs);
}
```

The code is stored in a github repository [here](https://github.com/pkeir/ctfft).
