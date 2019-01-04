---
layout: post
title:  "Swapping the Contents of n Variables"
category: C++ 
tags:
 - C++
 - folds
 - metaprogramming
---

C++11's `std::swap` is a binary function template which exchanges the contents of its two reference arguments. In C++20 `std::swap` will likely also permit execution at compile-time[^1]. In this post we'll write a version which can swap the contents of an arbitrary number of arguments using a C++17 *fold-expression*.

A candidate implementation for the non-array overload of `std::swap` is shown below. A simple `noexcept` specifier is also required, but we can introduce that later.

```cpp
template <typename T>
constexpr void swap(T &t1, T &t2)
{
  T tmp = std::move(t1);
  t1 = std::move(t2);
  t2 = std::move(tmp);
}
```

A version of `swap` which takes *three* arguments can help to identify a recursive pattern. In `swap3`, below, a temporary variable `tmp` is declared, and initialised, by a *move assignment* to the first argument. This is followed by a move assignment, of each function (template) argument, to the argument which follows it. The update of the *last* argument is irregular, in that its move assignment is to the temporary variable; still holding the contents of the *first* argument. Nevertheless, a simple pattern of move assignments is observed; one for each argument.

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

A general version can use variadic templates and a C++17 fold expression. The operator provided to the fold must be a binary function. Each of `swap3`'s move assignments involves *two* of `swap3`'s function (template) arguments; so a fold *can* be considered. The combining operation of a C++17 fold expression must though be an *operator*.

Of course there is no standard binary operator overload which will do anything like the move assignment we require. A simple class template container can be defined with a suitable binary operator; let's treat ourselves to `operator+`.

```cpp
template <typename T>
struct wrap {
  constexpr wrap &operator+(wrap &&w) { x = std::move(w.x); return w; }
  T &x;
};
```

The operator's definition comprises a move assignment, and a return statement. The move assignment is reassuringly similar to those in `swap` or `swap3`, with the additional consideration to work on the `x` member of the two `wrap` objects involved.

The operator's return statement will be required by the fold expression; just as `((0-1)-2)`[^2] "returns" the result of `(0-1)`; as the minuend for the remaining subtraction expression.

[^1]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0879r0.html
[^2]: Addition and subtraction in C++ have left-to-right associativity; so `(0-1-2)` will be parsed as `((0-1)-2)`; evaluating to `(-3)`. A left fold expression can achieve the same result: `[](auto ...xs){ return (... - xs); }(0,1,2)`.
