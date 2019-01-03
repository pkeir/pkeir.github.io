---
layout: post
title:  "Swapping the Contents of n Variables"
category: C++ 
tags:
 - C++
 - folds
 - metaprogramming
---

C++11's `std::swap` is a binary function template which exchanges the contents of its two reference arguments. In C++20 `std::swap` will likely also permit execution at compile-time[^1]. A candidate implementation for the non-array overload of `std::swap` is shown below. A simple `noexcept` specifier is also required, but we can discuss that later.

```cpp
template <typename T>
constexpr void swap(T &t1, T &t2)
{
  T tmp = std::move(t1);
  t1 = std::move(t2);
  t2 = std::move(tmp);
}
```

How would we swap the contents of an arbitrary number of arguments? A version which takes *three* arguments can help to identify a repeating pattern. In `swap3` below, a temporary variable `tmp` is declared and initialised by a move assignment from the first argument. This is followed by a move assignment, of each argument, to the argument which follows it. The *last* argument is distinct in its move assignment to the temporary variable; which still holds the contents of the *first* argument.

```cpp
template <typename T>
constexpr void swap3(T &t1, T &t2, T &t3)
{
  T tmp = std::move(t1);
  t1 = std::move(t2);
  t2 = std::move(t3);
  t3 = std::move(tmp);
}
```

A version defined using variadic templates could aim to use a C++17 *fold expression*. The operator provided to the fold must be a binary function. Each of `swap3`'s move assignments involves *two* of `swap3`'s function (template) arguments; so a fold *can* be considered. The combining operation of a C++17 fold expression must though be an *operator*.

Of course there is no standard binary operator overload which will do anything like the move assignment we require. A simple class template container can be defined with a suitable binary operator; let's treat ourselves to `operator+`.

```cpp
template <typename T>
struct wrap {
  constexpr wrap &operator+(wrap &&w) { x = std::move(w.x); return w; }
  T &x;
};
```

[^1]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0879r0.html
