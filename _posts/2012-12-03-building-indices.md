---
layout: post
title:  "Building, say, indices<6,4,2,0,-2,-4>"
category: C++ 
tags:
 - C++
 - templates
 - metaprogramming
---

A simple indexing class of variadic `std::size_t` template parameters is often used to provide a structured method to select multiple elements from a C++11 tuple. In this post I present an alternative to this interface, commonly used when building an object of this type, wherein a finite series of indices is now generated according to a numeric range and signed stride; akin to Fortran array section syntax. Let's look at the common solution first.

A tuple may be created either using the `std::tuple` constructor, or `std::make_tuple` function template. Both expect the same variadic series of arguments. In addition, the constructor requires the type of each argument explicitly. Hence, the more concise `std::make_tuple`, is used throughout the examples. In the example below, the last element of `t1` will be an `int`.

```cpp
auto t1 = std::make_tuple('z',false,42,"cat",7);
auto t2 = std::tuple<char,bool,int,const char *,long>('z',false,42,"ktn",7);
```

Selection of a single tuple element is achieved using the standard tuple function template: `std::get`. The sole template parameter of `std::get`, a `std::size_t`, represents an index; and must be a constant expression. Using the `t1` tuple from the code above, `std::get<4>(t1)` evaluates to an `int`; with a value of `7`.

As a *variadic* function template, `std::make_tuple` can of course be used to create a new tuple from the elements of another. In the code below, `t3`, having type `tuple<char,int,int>`, is formed from copies of the 1st, 3rd, and 5th elements from the earlier tuple, `t1`.

```cpp
auto t3 = std::make_tuple(std::get<0>(t1),std::get<2>(t1),std::get<4>(t1));
```

In the code above, we hard coded the selection using three index values: `0`, `2`, and `4`. How could we instead write a function template, *select*, that accepts, *at least*, a tuple argument, and returns another tuple formed from an arbitrary set of its elements? The conventional solution introduces the following simple variadic class template:

```cpp
template <std::size_t ...Is>
struct indices {};
```

An object of type `indices` might then be created with, for example: `indices<>`; `indices<0,2,4>`; or `indices<1,1,2,3,5,8>`. Our `select` function may then be defined, as shown in the code below.

```cpp
template <typename ...Ts, std::size_t ...Is>
auto
select(std::tuple<Ts...> t, indices<Is...>) ->
  decltype(std::make_tuple( std::get<Is...>(t)... )) {
  return   std::make_tuple( std::get<Is...>(t)... );
}
```

Calling `select` with the earlier tuple variable `t1` and a `indices<0,2,4>` variable, again results in a tuple with type `tuple<char,int,int>`.

There are a few C++11 things to notice in this code. The `Is` template
parameter pack is not expanded "directly" in the function body, due to the
placement of the ellipsis. So, while `indices<Is...>`, say, would expand to
`indices<0,2,4>`, in the `select` function above, the actual expansion becomes
`std::get<0>(t), std::get<2>(t), std::get<4>(t)`; three arguments for the
variadic `make_tuple` function. Such ellipses will expand all parameter packs
in the *pattern* to their left. Expanding multiple parameter packs of differing
lengths, with a single ellipsis, will cause a compilation error.

Also worth noting: the use of C++11's trailing return type, and `decltype` specifier is, here, optional; the `select` function *can* be typed just as effectively by the more ornate code below. Often, however, the former typing technique is preferable as it has a simple, mechanical, syntax-based application; and so is *generally* applicable. With short functions the concise form can also provide a readable symmetry. With luck, perhaps by C++17, or maybe even [C++14](https://www.ibm.com/developerworks/mydeveloperworks/blogs/5894415f-be62-4bc0-81c5-3956e82276f3/entry/the_view_from_c_standard_meeting_oct_2012196?lang=en), we can do away with it altogether; after all, the `auto` keyword can bind to untyped lambda expressions.

```cpp
template <typename ...Ts, std::size_t ...Is>
std::tuple<
  typename std::decay<
    typename std::tuple_element<Is,std::tuple<Ts...>>::type
  >::type...
>
select(std::tuple<Ts...> t, indices<Is...>) {
  return std::make_tuple( std::get<Is...>(t)... );
}
```

Finally, as only the template arguments of its type are used, the second
function parameter is not bound to a name.

All well and good, but how would I modify *every* element of an
arbitrary-length tuple? A common solution, seen
[here](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2006/n2080.pdf#page=16),
[here](http://gitorious.org/redistd/redistd/blobs/d1ca735e11662b69a57c283e1c8fc22aeecef6de/include/redi/index_tuple.h),
and
[here](http://opensource.apple.com/source/clang/clang-318.0.45/src/tools/clang/test/CXX/temp/temp.decls/temp.variadic/metafunctions.cpp),
is to use `make_indices`, a variadic class template with a `typedef` member,
`type`, which equates to an instantiation of the `indices` class template. The
`std::size_t` template parameters of the `indices` type are instantiated as a
zero-based finite arithmetic progression, with length equal to the number of
template arguments given to `make_indices`. For example,
`make_indices<short,int>::type`, would evaluate to `indices<0,1>`. The code
below demonstrates a simple application of `make_indices` within a function
template, `id`, which "does nothing"; well, it returns a tuple comprised of the
same elements as the input. With additional parameters, `make_indices` can
easily be used to create tuple versions of *map* and *zipWith*.

```cpp
template <typename ...Ts>
std::tuple<Ts...> id(std::tuple<Ts...> t) {
  return select(t,typename make_indices<Ts...>::type());
}
```

Although useful, the `make_indices` class template has a number of weaknesses:

* The template parameters of `indices` are fixed as `std::size_t` only;
* The first index created is always `0`;
* The common difference between each index is fixed to `1`;
* The common difference is a positive value only;
* `make_indices` can only be applied to type template parameter lists, and not non-type template parameters.

The first point can be addressed by a variadic, generic, index container:

```cpp
template <typename T, T...>
struct indicesT {};
```

With tuples, the argument given to the relevant index function, `std::get`, will commonly have type `std::size_t`. The following *type alias template* allows the more straightforward `std::size_t` specialisation of `indicesT` to be used; e.g. `indices<0,1,2>` instead of `indicesT<std::size_t,0,1,2>`; also, the earlier definition of `select` can remain unchanged.

```cpp
template <std::size_t ...Is>
using indices = indicesT<std::size_t, Is...>;
```

The traditional `make_indices` class template outlined earlier only allows us to specify the *extent* of the generated, zero-based, indices. Using the `mk_index_range` alias template, the same may be achieved using an integral range. For example, like `make_indices<int,bool,char>::type`, the type expression `mk_index_range<0,2>` will evaluate to `indices<0,1,2>`; or `[0,2]` in interval notation.

The `mk_index_range` alias template can also use a non-zero based *start*
value; for example `mk_index_range<8,10>` will evaluate to `indices<8,9,10>`. A
third, optional, template parameter of `mk_index_range` allows the
specification of a *stride*. So, `mk_index_range<1,9,2>` will evaluate to
`indices<1,3,5,7,9>`; and `mk_index_range<9,1,-2>` to `indices<9,7,5,3,1>`.

The indices produced by `mk_index_range` will always have type `std::size_t`;
that is, the same type as the template parameter of the `<tuple>` function `std::get`.
To produce *signed* indices, say of type `int`, a second alias template is
provided: `mk_index_rangei`. Behind the scenes, something like
`mk_index_rangei<3,-3,-1>`, which evaluates to `indicesT<3,2,1,0,-1,-2,-3>`, is defined as `MkIndices<int,int,3,-3,-1>`,
using a more general alias template, `MkIndices`.

A fair question is, how many indices are produced by an arbitrary index range? For example, what is produced by `mk_index_range<1,9,2>` as opposed to `mk_index_range<1,10,2>`. How about `mk_index_range<1,0,1>`? The answer comes from the language of the new century: Fortran. With the *loop-control* of the DO construct defined by the grammar production below,

```
do-variable = expr1, expr2 [,expr3]
```

then the number of iterations is defined by the following equation:

```
niters = max((expr2 − expr1 + expr3)/expr3, 0)
```

The C++ code for this is a `constexpr` function which can be used within the template arguments of the `mk_index_range` implementation.

We're now in a position to have some fun. The following basic function, `tuple_tail` is used within the definition of the tuple overload of the insertion operator `<<`. The `tuple_tail` function returns a tuple comprised of all elements of the input tuple argument, *minus* the first element:

```cpp
template <typename T, typename ...Ts>
tuple<Ts...>
tuple_tail(tuple<T,Ts...> t) {
  return select(t, mk_index_range<1,sizeof...(Ts)>());
}
```

The following function, `tuple_reverse`, unsurprisingly returns a tuple constructed from all the elements of the input tuple argument, in reverse order:

```cpp
template <typename ...Ts>
tuple<Ts...>
tuple_reverse(tuple<Ts...> t) {
  return select(t, mk_index_range<sizeof...(Ts)-1,0,-1>());
}
```

The `mk_index_range` function template is now used throughout an updated version of the compile-time FFT code described in the [previous post](https://pkeir.github.io/blog/2012/09/02/compile-time-fft). The `map`, `zipWith` and `iota` functions there now all use `mk_index_range`; they're also similar to each other; and are interesting enough. The following function, `condenseN`, is though more compelling: returning a tuple comprised of every *n*th element of the input tuple. It is both integral to the FFT implementation; *and* uses a stride, or common difference, that isn't `1`. Incidentally, the actual instantiation is `condenseN<2>`, and alas this will crash the current version of Clang; Clang 3.2 snapshot.

```cpp
template<std::size_t N, typename ...Ts>
auto
condenseN(tuple<Ts...> t) ->
  decltype(select(t,mk_index_range<0,sizeof...(Ts)-1,N>())) {
  return   select(t,mk_index_range<0,sizeof...(Ts)-1,N>());
}
```

My personal favourite is also used by the compile-time FFT. The `std::tuple_cat` is a variadic function template which should catenate all tuples provided as arguments. The implementation below uses a helper function, `tuple_cat_helper`, which expands two parameter packs, `Is` and `Js`, within a single statement:

```cpp
template <typename ...Ts>
tuple<>
tuple_cat() { return tuple<>(); }

template <typename ...Ts>
tuple<Ts...>
tuple_cat(tuple<Ts...> t) { return t; }

template <typename Tup1, typename Tup2, std::size_t ...Is, std::size_t ...Js>
auto tuple_cat_helper(Tup1 t1, Tup2 t2, indices<Is...>, indices<Js...>) ->
  decltype(make_tuple(get<Is>(t1)...,get<Js>(t2)...)) {
  return   make_tuple(get<Is>(t1)...,get<Js>(t2)...);
}

template <typename ...Ts, typename ...Us, typename ...Tups>
auto
tuple_cat(tuple<Ts...> t1, tuple<Us...> t2, Tups ...ts) ->
  decltype(tuple_cat(
             tuple_cat_helper(
               t1,t2,
               mk_index_range<0,sizeof...(Ts)-1>(),
               mk_index_range<0,sizeof...(Us)-1>()
             ),
             ts...
           )
          )
{
  return   tuple_cat(
             tuple_cat_helper(
               t1,t2,
               mk_index_range<0,sizeof...(Ts)-1>(),
               mk_index_range<0,sizeof...(Us)-1>()
             ),
             ts...
           );
}
```

Note that the C++11 *standard* definition of `std::tuple` is not defined as `constexpr`. The non-standard `tuple` implementation provided with the FFT code is contained within the `ctup` namespace.

The code for `mk_index_range` is [here](https://github.com/pkeir/ctfft/blob/master/mk_index_range.hpp) and the `constexpr`-friendly `tuple` implementation is [here](https://github.com/pkeir/ctfft/blob/master/const_tuple.hpp).

