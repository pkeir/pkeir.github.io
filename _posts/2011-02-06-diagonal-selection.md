---
layout: post
title:  "Diagonal Selection"
category: Fortran
tags:
 - Fortran
 - array programming
---

A good example of the expressivity of the *forall* statement from Fortran 95 and HPF, is the selection of the elements on the diagonal of an array. We begin such a thing with the declarations of a rank 2 array *a*, and a vector *da* to hold the diagonal:

```fortran
integer :: i, a(4,2) = reshape([(i,i=1,size(a))],shape(a))
integer :: da(minval(shape(a)))
```

Note that regardless of the rank of the original array, the length of the sought diagonal will always be equal to the mininum element of the array's shape vector. In our case the shape vector of *a* is [ 4, 2 ], and so the diagonal has a length of 2. Of course this is due to our definition of the diagonal of an array. Informally, our notion of diagonal corresponds to the set of array elements, indexed by a set of equal values; and ordered by those values. So, for *a*, this will be the elements indexed by *a(1,1)* and *a(2,2)*: [ 1, 6 ]. So then, let's look at the *forall* statement to achieve this:

```fortran
forall (i=1:size(da)) da(i) = a(i,i)
```

How would this look as an array expression? As discussed in my [first blog post](https://pkeir.github.io/blog/2010/02/23/fragrant-fortran), such array expressions hold the potential to handle input of arbitrary dimensionality; unlike the *forall* statement; though certainly more verbose.

# Diagonal Selection

If an array of rank *n* is flattened, say by the Fortran *pack* intrinsic, it becomes clear that the number of elements separating each adjacent pair of *diagonal* members, is constant. Conveniently, the set of recursive equations which can express this constant may also be encoded using the *scan_product* function from the [previous blog post](https://pkeir.github.io/blog/2010/09/06/scanners).

The array *a* has shape [ 4, 2 ]. The product scan of [ 4, 2 ] is [ 4, 8 ]. We're looking for a logical mask vector to select the diagonal elements. ( True and False will also be denoted as 1 or  0 as required. ) Some examples: the earlier [ 4, 2 ] shape array will require a mask of [ **1**, 0, 0, 0, 0, **1**, *0*, *0* ]; while an array of shape [ 3, 4 ] requires a mask of [ **1**, 0, 0, 0, **1**, 0, 0, 0, **1**, *0*, *0*, *0* ]. The pattern to notice is a repeating string of False values, delimited by True values, then succeeded by False values. The repeating string of False values has a length of four in the first example, and a length of three in the second. How many False values in general? With an input array *a*, the following code will assign the answer to the *zps* (zeros_per_step) integer variable:

```fortran
zps = sum(eoshift(scan_product(shape(a)),-1))
```

This value be used to size a logical vector (*zs*) of False values as shown. A True value should be cons'd to the start of this vector, and the result repeated *mi* times, where *mi* is one less than the length of the final diagonal. The implicit do loop can handle such concatenative actions concisely, as shown in the code below:

```fortran
zs  = spread(.false.,1,zps)
mi  = minval(shape(a))-1
tzs = [(.true.,zs,j=1,mi)]
```

The final, padding, string of False values, the vector *zpd* below, is longest in arrays having a relatively large final dimension; for example, shapes such as [ 2, 10 ], or [ 4, 5, 100 ].

```fortran
nz = product(shape(a)) - (mi * (zps+1)) - 1
zpd = spread(.false.,1,nz)
```

To construct the final 1D boolean selection mask, *msk*, the *tzs* vector has both a single True value appended; and also the *zpd* vector:

```fortran
msk = [(tzs,.true.,zpd,i=1,1)]
```

To apply this mask vector, we need to flatten the input array *a* using the *pack* intrinsic. The same *pack* intrinsic can then be used to apply the mask we have constructed above:

```fortran
pack(pack(a,.true.),msk)
```

If the operation is to be encapsulated in a function, we are once again forced to choose a single, static rank for the result. The code below is then arbitrarily for an array of rank 3; again, an interface statement, could help construct a verbose generic representation. I've also used the *associate* *construct* from Fortran 2003, which operates like a non-recursive let expression. Hopefully it can aid readability; at the least one can easily print out the contents of each binding.

```fortran
function diag3(a) result(r)
  integer, intent(in) :: a(:,:,:)
  integer :: i,j,r(minval(shape(a)))

  associate (sa  => shape(a)         )
  associate (sc  => scanl_product(sa))
  associate (eo  => eoshift(sc,-1)   )
  associate (zps => sum(eo)          )
  associate (ps  => product(sa)      )
  associate (mi  => minval(sa)-1     )
  associate (nz  => ps - (mi * (zps+1))-1)
  associate (zpd => spread(.false.,1,nz))
  associate (zs  => spread(.false.,1,zps))
  associate (tzs => [(.true.,zs,j=1,mi)])
  associate (msk => [(tzs,.true.,zpd,i=1,1)])
    r = pack(pack(a,.true.),msk)
  end associate
  end associate
  end associate
  end associate
  end associate
  end associate
  end associate
  end associate
  end associate
  end associate
  end associate
end function
```
