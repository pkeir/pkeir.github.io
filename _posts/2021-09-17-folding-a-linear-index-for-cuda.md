---
layout: post
title:  "Folding a Linear Index from CUDA Dimensions and Indices"
author: "Paul Keir"
category: C++
tags:
 - C++
 - gpgpu
 - cuda
 - folding
 - constexpr
---

In this post I'll introduce a C++ variadic function template to calculate a
global linear id, using a left fold over a list of extent/index pairs
corresponding to each dimension of a rectilinear structure, such as a
multidimensional array.  The fold is first derived in Haskell, before the C++
version, applicable to arbitrary dimension counts, is applied to the familiar
`gridDim`, `blockDim`, `blockIdx` and `threadIdx` builtins of a CUDA grid.

CUDA kernels are executed by numerous threads. The number and configuration
of these is referred to as a CUDA grid, and set at *runtime* using
non-C++ triple
chevron syntax. For example with a kernel launch such as
`my_kernel<<<dim3{7,6,5},dim3{4,3,2}>>>()`, a
total of 5040 (i.e. `7*6*5*4*3*2`) threads would be
launched; organised as a collection of 210 CUDA thread blocks,
each containing 24 (i.e. `4*3*2`) threads.

Each thread in a CUDA grid is uniquely identified by the `unsigned int`
zero-based `x`, `y`, and `z` data members of the `dim3` built-in constants:
`blockIdx` and `threadIdx`.  Each of these data members will be less than its
corresponding kernel launch parameter. The kernel launch parameters used in
the triple-chevron syntax are also available within the kernel using two
further built-ins, named `gridDim` and `blockDim`.

CUDA's `blockIdx` and `threadIdx` variables are analogous to the
indices of a set of six nested loops. In the serial C++ code below,
the innermost loop body, in a cache friendly manner, increments
each element of `a6`; a 6D array. Here the array extents are
equal to each loop's iteration count. Indexing in general remains
comparably simple while each pair are proportionate.

```cpp
int main(int argc, char *argv[])
{
  struct dim3 { unsigned x = 1, y = 1, z = 1; };
  float a6[5][6][7][2][3][4]{};
  float a1[5*6*7*2*3*4]{};

  const dim3 gridDim{7,6,5}, blockDim{4,3,2};

  for (unsigned i6 = 0; i6 <  gridDim.z; ++i6)
  for (unsigned i5 = 0; i5 <  gridDim.y; ++i5)
  for (unsigned i4 = 0; i4 <  gridDim.x; ++i4)
  for (unsigned i3 = 0; i3 < blockDim.z; ++i3)
  for (unsigned i2 = 0; i2 < blockDim.y; ++i2)
  for (unsigned i1 = 0; i1 < blockDim.x; ++i1)
  {
    const dim3 threadIdx{i1,i2,i3}, blockIdx{i4,i5,i6};
    a6[blockIdx.z][blockIdx.y][blockIdx.x][threadIdx.z][threadIdx.y][threadIdx.x]++;
  }

  return 0;
}
```

Of course large stack-based arrays are rarely used. Consequently, a common
requirement in CUDA codes is the calculation of a global linear (flat) id to
use as an offset from the base address of dynamically allocated global arrays.
In the code above, such a linear id would be useful if it was instead the
`a1` array which should have its elements incremented[^1].

When required, global linear ids can be calculated from the four built-in
constants mentioned earlier. For a one dimensional grid of one dimensional
blocks (1x1), the calculation is simple: `blockIdx.x * blockDim.x +
threadIdx.x`.  When used as an array index, this is conducive to memory
coalescence[^2]. The consecutive, increasing thread IDs[^3] of each warp's
threads, correspond to the similarly consecutive, increasing values calculated
from the thread indices in this common expression.  In contrast, while another
expression, `threadIdx.x * gridDim.x + blockIdx.x`, will also return unique
linear ids; it does *not* produce consecutive, increasing values, from threads
with consecutive, increasing thread IDs.

```cuda
__device__ unsigned getGlobalIdx_2D_2D()
{
  unsigned blockId = blockIdx.x + blockIdx.y * gridDim.x;
  unsigned threadId = blockId * (blockDim.x * blockDim.y) + (threadIdx.y * blockDim.x) + threadIdx.x;
  return threadId;
}

__global__ void kernel2x2(float *p)
{
  p[getGlobalIdx_2D_2D()]++;
}
```

When calculating a global linear id from a *two* dimensional grid of *two*
dimensional blocks (2x2), I often find myself reaching for `getGlobalIdx_2D_2D`
from Martin Peniak's [CUDA Thread Indexing
Cheatsheet](https://cs.calvin.edu/courses/cs/374/CUDA/CUDA-Thread-Indexing-Cheatsheet.pdf);
courtesy of Calvin University.  In the code above, assuming the global array
referenced by `p` has space for at least as many `float` values as there are
threads in the CUDA grid, then all its elements will be incremented, and memory
accesses duly coalesced.  Again, it is easy to (perhaps accidentally) calculate
a *different* set of unique linear ids, which also (as with `threadIdx.x *
gridDim.x + blockIdx.x`) leads to non-coalesced accesses to global memory. An
example of this common pattern is shown in the code below:

```cuda
__device__ unsigned get_global_id_2x2_noncoalesced()
{
  unsigned row = blockIdx.y * blockDim.y + threadIdx.y;
  unsigned col = blockIdx.x * blockDim.x + threadIdx.x;
  unsigned width = gridDim.x * blockDim.x;
  return width * row + col;
}

__global__ void kernel2x2_noncoalesced(float *p)
{
  p[get_global_id_2x2_noncoalesced()]++;
}
```

Memory access in the kernel above is not coalesced. Why? Two threads with
consecutive thread IDs[^3], say in a warp, might still have a different
`threadIdx.y`. The row/column system of `get_global_id_2x2_noncoalesced`
above though would calculate indices for such pairs as if they were from
separate rows of a larger 2D structure with a width of `gridDim.x *
blockDim.x`. At the point where `row` and `col` are initialised, we essentially
concatenate all of the blocks of a grid together; and in doing so, we lose
valuable index information.

It is a little surprising that NVIDIA do not provide an API for obtaining such
global linear ids themselves. Then again, while the Khronos OpenCL API includes
`get_global_linear_id`, and Khronos SYCL too has the
`get_global_linear_id` member function of `nd_item`, both are
effectively specified using the non-coalescing row/column approach just
discussed.

# Folding

Rather than move on to consider how a scenario involving a
*three* dimensional grid of *three* dimensional blocks (3x3) might play out,
we can perhaps already agree that there is cause to pause
when calculating a linear index from a multidimensional array index set.
Instead, let's look at a potential solution to the problem *in
general*: how a fold can calculate a linear index from the common input
parameters, an array of extent/index pairs.

```haskell
index_left_fold :: (Foldable t, Num a) => p -> t (a, a) -> (a, a)
index_left_fold is = foldl (\(e1,i1) (e2,i2) -> (e1*e2,i1*e2+i2)) (1,0)
```

The `index_left_fold` Haskell function above returns a tuple, the first
component of which holds the calculated total size of the array, while the
second component holds the topical linear index. (The definition can use
`foldr` as a drop in replacement for `foldl` if desired.) Let's consider
the application: `index_left_fold [(4,1),(8,6)]`. In a C++ context, this could
relate to a 4x8 array (i.e. `T[4][8]`); if we named it `arr` we would
index it with syntax like: `arr[1][6]`. As expected, the tuple returned
is `(32,14)`.

When considering a C++ version we note that there a few options to invoke a
suitable folding mechanism. The `std::accumulate` function template is a [^4],

alas, the versions included with standard C++ libraries will not run through
CUDA on the GPU; the function is also absent from NVIDIA's libcu. Regardless of
nvcc's `--expt-relaxed-constexpr` flag, the `constexpr` attribute of
`std::accumulate` is no help here as crucial parameters such as `threadIdx` are
not usable within constant expressions.

above uses a tuple of integers


SYCL has 2D buffers/accessors

It is
certainly
possible already to device a linear indexing scheme 



So too, the 

is due to the performance advantage gained through memory coalescence.
ensures that 
which applies when
threads with 
is that memory accesses will 

CUDA Programming Guide

Section 2.2 Thread Hierarchy

Section 4.1. SIMT Architecture
"...each warp contains threads of consecutive, increasing thread IDs with the first warp containing thread 0."

 will also 
As the smallest unit of execution, a
*warp* involves the simultaneous execution of 

A two dimensional grid of
two dimensional blocks requires a bit more care:
`blockIdx.x

, with `threadIdx` and
`blockIdx` a unique pairing within a CUDA grid.

This is also needed in other array calculations.

are analogous to the extents
of a common rectinlinear array.

It is often useful to flatten

, each of the thousands of threads involved
has a unique identifier. Like the 









[^1]: CUDA could not have the luxury here of incrementing a newly-added index variable to the outer scope. With CUDA, each kernel instance executes in parallel.

[^2]: Coalesced memory access occurs when the threads of a CUDA hardware warp, access consecutive memory addresses during the same clock cycle.

[^3]: CUDA thread IDs are unique only within a block. For more information on thread IDs, see Section 2.1. Kernels and Section 2.2 Thread Hierarchy from the CUDA Programming Guide (version 11.4).

[^4]: https://github.com/pkeir/accumulate

