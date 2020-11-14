---
layout: post
title:  "Fragrant Fortran"
author: "Paul Keir"
category: Fortran
tags:
 - Fortran
 - array programming
---

Fortran is a programming language like C, with more capital letters, and less pointers. Since around 1990, the Fortran standard has also supported arrays as first class language elements. Such array features, provided by Fortran 90, Fortran 95, and Fortran 2003, are reminiscent of Kenneth Iverson's seminal APL (**A** **P**rogramming **L**anguage), and J; though less developed.

Fortran 9x includes many useful and essential procedures for manipulating arrays. For example, the function *shape* takes one array argument (of any dimensionality) and return a one-dimensional array (often referred to as a vector) which lists the extent/size of each dimension. The length of this vector is therefore equal to the dimensionality of the argument to the *shape* function.

Tens of such array functions are specified by the Fortran standard, and included in essentially every Fortran compiler. Along with the more numerous scalar functions, these are collectively referred to as the Fortran intrinsics.

Fortran array intrinsics are often defined for arrays of arbitrary dimensionality (rank). However, the end user is made to work hard to replicate such genericity. A user-defined function accepting or returning arrays must specify a fixed dimensionality. Consequently, Fortran's expression language is massively more concise than its procedure language. Hence,

```fortran
m = sum(a) / size(a)
```

will assign the mean (excepting zero-sized arrays, and other desiderata) of input array *a*, to *m*, regardless of the rank of the *a*. A function to perform a similar action on a *one-dimensional* array could look like this:

```fortran
function mean1(a) result(m)
  real, intent(in) :: a(:)
  real :: m

  m = sum(a) / size(a)
end function
```

which could be called with:

```fortran
m = mean1(a)
```

We would need another for an array of rank 2:

```fortran
function mean2(a) result(m)
  real, intent(in) :: a(:,:) ! Extra colon !
  real :: m

  m = sum(a) / size(a)
end function
```

The two versions of mean differ only by the rank in array *a*'s declaration. Also, apart from such "boilerplate" declarations, the core of our function is lexically the same as the earlier expression; `m = sum(a) / size(a)`. At least Fortran arrays have a maximum rank of seven, so we would "only" need seven versions of this *mean* function. Tragically however, if a function takes *n* array arguments, we will need to write *7^n* functions.

One ray of light comes in the form of Fortran's interface statement. The interface statement allows a user to access multiple procedures via a single name; similar to C++ overloading. For example, the following interface statement allows a call to *mean* to be instantiated appropriately, determined by the types of its arguments.

```fortran
interface mean
  module procedure mean1, mean2
end interface mean
```

It's worth mentioning that Professor Martin Erwig's [Parametric Fortran](http://web.engr.oregonstate.edu/~erwig/pf) project provides one solution to this convoluted aspect of Fortran.

Finally, note also that alongside array expressions, 1990 also eschewed ye olde brutal and capitalist "FORTRAN" in favour of the effervescent stateswoman we know and love as "Fortran" today.
