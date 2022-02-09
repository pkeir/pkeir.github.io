---
layout: post
title:  "Folding a Global Linear Id from CUDA Dimensions and Indices"
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
global linear index, using a left fold over a list of extent/index pairs
corresponding to each dimension of a rectilinear structure; such as a
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
zero-based `x`, `y`, and `z` data members of the `uint3` built-in constants:
`blockIdx` and `threadIdx`.  Each of these data members will be less than its
corresponding kernel launch parameter. The kernel launch parameters used in
the triple-chevron syntax are also available within the kernel using two
further built-ins, named `gridDim` and `blockDim`.

CUDA's `blockIdx` and `threadIdx` variables are analogous to the
indices of a set of six nested loops. In the serial C++ code below,
the innermost loop body, in a cache friendly manner, increments
each element of `a6`; a 6D array. Here the array extents are
equal to each loop's iteration count. Nested loop indexing is often
as simple when the array and loop extents are proportional.

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
threadIdx.x`.  When used as an array index, this is also conducive to memory
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
referenced by `p` has space for as many `float` values as there are
threads in the CUDA grid, then all its elements will be incremented, and memory
accesses are duly coalesced.  Again, it is easy to (perhaps accidentally)
calculate a *different* set of unique linear ids, which also (as with
`threadIdx.x * gridDim.x + blockIdx.x`) leads to non-coalesced accesses to
global memory. An example of such a common pattern is shown in the code below:

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

Memory access in the kernel above is not coalesced. Why? Consider two threads
with consecutive thread IDs[^3]; say in a warp. Such a pair will often have a
different `threadIdx.y`. The row/column system of
`get_global_id_2x2_noncoalesced` above though would calculate indices for such
pairs as if they were from separate rows of a larger 2D structure with a width
of `gridDim.x * blockDim.x`. At the point where `row` and `col` are
initialised, we essentially concatenate all of the blocks of a grid together;
and in doing so, lose valuable index information.

It is a little surprising that NVIDIA don't provide an API for obtaining such
global linear ids themselves. The Khronos OpenCL API includes
`get_global_linear_id`; and Khronos SYCL too has the
`get_global_linear_id` member function of `nd_item`.

# Folding

Rather than advance, to consider how a scenario involving a
*three* dimensional grid of *three* dimensional blocks (3x3) might play out,
we can perhaps already agree that there is cause to pause
when calculating a linear index from a multidimensional array index set.
Instead, let's look at a potential solution to the problem *in
general*: how a fold can calculate a linear index from the common input
parameters, a collection of extent/index pairs.

```haskell
index_left_fold :: [(Integer, Integer)] -> (Integer, Integer)
index_left_fold = foldl (\(e1,i1) (e2,i2) -> (e1*e2,i1*e2+i2)) (1,0)
```

The `index_left_fold` Haskell function above returns a tuple. Its first
component holds the calculated total size of the array; and its second
component holds the calculated linear index. (The definition can use `foldr` as
a drop in replacement for `foldl` if desired.) Let's consider an example
application: `index_left_fold [(4,1),(8,6)]`. In a C++ context, this could
relate to a 4x8 array type (i.e. `T[4][8]`); if we gave it a pirate name like
`arr` we would index it with syntax such as: `arr[1][6]`. As expected, the tuple
returned is `(32,14)`. Reassuringly, the result of `index_left_fold
[(4,1),(8,7)]` is `(32,15)`; with the linear index of `15` one greater than the
earlier `14`, after incrementing the fastest moving index.

The two components of the fold's combining function have an intuitive aspect.
The second part, `i1*e2+i2`, which ultimately produces the linear index, is
reminiscent of the common `i*w+j` expression that might similarly
calculate a linear index in a traditional doubly nested loop. Meanwhile, the
first part, `e1*e2`, accumulates the array size as the product of its extents.
At each step of the fold, the value provided by the `e2` parameter holds the
product of the array dimensions so far; aligning with the idea that a
multidimensional array is also an array of arrays. So too the fold's initial
value of `(1,0)` provides the multiplicative unit (i.e. 1) for the product
calculation; and zero can be thought of as an offset into a non-existent
zeroeth dimension.

# Implementation in C++

When considering a C++ version we note that there a few options to invoke a
suitable folding mechanism. The `std::accumulate` function template facilitates
a versatile left fold[^4], but the version included with standard C++ libraries
will not run via nvcc on the GPU. Note also that the `constexpr`
attribute of `std::accumulate` is irrelevant here as the CUDA built-ins,
including `threadIdx`, are all *runtime* values.  `std::accumulate` is also
absent from [NVIDIA's libcu++](https://nvidia.github.io/libcudacxx).

```cuda
constexpr __forceinline__ __device__ __host__
unsigned index_left_fold(const unsigned extent, const unsigned index) {
  return index;
}

template <typename T, typename ...Ts>
constexpr __forceinline__ __device__ __host__
unsigned index_left_fold(const T e1, const T i1, const T e2, const T i2, Ts ...xs) {
  return index_left_fold(e1*e2, i1*e2+i2, xs...);
}
```

The CUDA C++ definition of `index_left_fold` shown above instead uses variadic
templates. The recursive overload processes the first *two* arguments at each
step. This leads to the final accumulated extent being discarded in the
recursive base case at the top. This is already usable for C-style arrays; with
the same indexing applied to the 4x8 array from earlier, the result of
`index_left_fold(4,1, 8,6)` is `14`, and the result of `index_left_fold(4,1,
8,7)` is `15`.

When working within a CUDA kernel, the indices and extents are already
provided, by the built-in constants: `gridDim`, `blockDim`, `blockIdx` and
`threadIdx`; and CUDA's three dimensional grid of three dimensional blocks
(3x3) is isomorphic to a 6D array. The fastest moving indices correspond to the
thread indices within each block; while the outermost extents correspond to
the number of blocks within each grid dimension. Thankfully the specific
mapping of indices to extents for each of the 6 dimensions need be defined only 
once for common configurations, as shown below:

```cuda
template <unsigned dims>
__forceinline__ __device__ unsigned cuda_global_linear_id();

template <>
__forceinline__ __device__ unsigned cuda_global_linear_id<1>() {
  return index_left_fold(gridDim.x,  blockIdx.x,
                         blockDim.x, threadIdx.x);
}

template <>
__forceinline__ __device__ unsigned cuda_global_linear_id<2>() {
  return index_left_fold(gridDim.y,  blockIdx.y,
                         gridDim.x,  blockIdx.x,
                         blockDim.y, threadIdx.y,
                         blockDim.x, threadIdx.x);
}

template <>
__forceinline__ __device__ unsigned cuda_global_linear_id<3>() {
  return index_left_fold(gridDim.z,  blockIdx.z,
                         gridDim.y,  blockIdx.y,
                         gridDim.x,  blockIdx.x,
                         blockDim.z, threadIdx.z,
                         blockDim.y, threadIdx.y,
                         blockDim.x, threadIdx.x);
}
```

When applied to the example task from earlier, of incrementing each element of
a suitably sized array from within a CUDA kernel, with coalesced reads and
writes, the 3 common varieties can now appear as shown below.  Note that
`cuda_global_linear_id<3>()` also works for all 3 of these kernels as CUDA implictly
sets unspecified dimension extents to `1`; and corresponding indices to `0`.

```cuda
__global__ void kernel1x1(unsigned *p) { p[cuda_global_linear_id<1>()]++; }
__global__ void kernel2x2(unsigned *p) { p[cuda_global_linear_id<2>()]++; }
__global__ void kernel3x3(unsigned *p) { p[cuda_global_linear_id<3>()]++; }
```

# Both Linear Index and Size

The `index_left_fold` variadic function template introduced above is inspired
by simplicity; involving no additional user or library types. Nevertheless, it
seems wasteful for the recursive base case to discard the final `extent` value;
which at this point contains the calculated total size of the array. Returning
the total size and linear index as a *pair* of values can drive the design of
an alternative implementation.

While the extent/index pair to be returned *could* be stored in an `std::pair`
or `std::array` instance, a custom type built on two `unsigned int` members
is sufficient; and allows for a combining operator to be defined,
so permitting this implementation to use a concise C++17 fold expression.
The `ei` class is shown below. Note how the `operator+` definition
resembles that of `index_left_fold`.

```cuda
struct ei
{
  unsigned extent, index;

  __forceinline__ __device__ __host__
  constexpr ei operator+(ei x) { return {extent * x.extent, index * x.extent + x.index}; }

  __forceinline__ __device__ __host__
  constexpr bool operator==(const ei& x) const {
    return extent==x.extent && index==x.index;
  }
};
```

The variadic function template to calculate the
linear index and array size *together* can then be defined in short order:

```cuda
template <typename ...Ts>
__forceinline__ __device__ __host__
constexpr ei size_index_left_fold(Ts ...xs) { return (ei{1,0} + ... + xs); }

static_assert(ei{48,47}==size_index_left_fold(ei{2,1}, ei{4,3}, ei{6,5}));
```

# Unfolded

How would the CUDA linear index expressions appear if they were typed in full,
by-hand rather than by a fold? The three `unsigned int` variables in the code
below are initialised by expressions which are equivalent to the fold
expressions obtained by expanding `cuda_global_linear_id` for the 1x1, 2x2 and
3x3 configurations respectively.

```cpp
unsigned i1 = blockIdx.x*blockDim.x+threadIdx.x;
unsigned i2 = ((blockIdx.y*gridDim.x+blockIdx.x)*blockDim.y+threadIdx.y)*blockDim.x+threadIdx.x;
unsigned i3 = ((((blockIdx.z*gridDim.y+blockIdx.y)*gridDim.x+blockIdx.x)*blockDim.z+threadIdx.z)*blockDim.y+threadIdx.y)*blockDim.x+threadIdx.x;
```

We can then compare these with the expressions used by the
[CUDA Thread Indexing
Cheatsheet](https://cs.calvin.edu/courses/cs/374/CUDA/CUDA-Thread-Indexing-Cheatsheet.pdf).
For the 1x1 case, there is no difference: both use
`blockIdx.x*blockDim.x+threadIdx.x`. For the 2x2 case, the Cheatsheet's
`getGlobalIdx_2D_2D` uses `4` multiplications and `3` additions; whereas
`cuda_global_linear_id<2>` uses fewer: `3` multiplications, and `3` additions.
`getGlobalIdx_2D_2D` also uses the `blockDim.x` constant twice, while
`cuda_global_linear_id<2>` uses each of its terms only once. For the 3x3 case,
differences increase. With `getGlobalIdx_3D_3D`, `9` multiplications and `5`
additions are used; while `cuda_global_linear_id<3>` uses only `5`
multiplications and `5` additions; each one less than the number of array
dimensions involved (i.e. 6 here)[^5].  Again, no term repetitions occur in
`cuda_global_linear_id<3>`; while `getGlobalIdx_3D_3D` uses `blockDim.x` three
times, `blockDim.y` twice, and `gridDim.x` twice also.

One may nevertheless fear that the recursion, or paired accumulation of extent
and index in the inlined call to a `cuda_global_linear_id` template
instantiation, may lead to the compiler producing a suboptimal representation
of the unfolded expressions discussed above. Comparing the PTX output by `nvcc`
for two minimal test programs, with one using `cuda_global_linear_id`, and the
other using the unfolded expressions, shows the pair to be equivalent: only the
order of the PTX operations, and the registers used differ. Furthermore, with a
similar pair of cubin files, `nvdisasm` reveals that this distinction is
trivial, with both versions producing identical assembly code.

# Conclusion

The project goal was to create a fold operation to produce a linear index from
a collection of array extent/index pairs, applicable to CUDA threads.  If you
are in need of a global linear id for a CUDA thread, please try the
`cuda_global_linear_id` function template. For more general array applications,
the underlying `index_left_fold` and its associate `size_index_left_fold` are
also available.  The specialisations of `cuda_global_linear_id` appear optimal
for three common CUDA grid configurations (1x1, 2x2, and 3x3), and handle all
other configurations well[^6]. The GitHub repository for the code introduced
above can be found at
[https://github.com/pkeir/cuda-linear-indexing](https://github.com/pkeir/cuda-linear-indexing).


[^1]: CUDA could not have the luxury here of incrementing a newly-added index variable to the outer scope. With CUDA, each kernel instance executes in parallel.

[^2]: Coalesced memory access occurs when the threads of a CUDA hardware warp, access consecutive memory addresses during the same clock cycle.

[^3]: CUDA thread IDs are unique only within a block. For more information on thread IDs, see Section 2.1. Kernels and Section 2.2 Thread Hierarchy from the CUDA Programming Guide (version 11.4).

[^4]: [https://github.com/pkeir/accumulate](https://github.com/pkeir/accumulate)

[^5]: The regularity of the fold operation described by `index_left_fold` means that with an array of dimension `N`, there will be `N-1` multiplication operations, and `N-1` addition operations; i.e. the complexity is O(n).

[^6]: Any `dim3` components left unspecified during a CUDA kernel launch default to `1`. It would not be difficult to create a set of function template specialisations which omit such unary dimension extents from the corresponding call to `index_left_fold`; so omitting suboptimal multiplications by `1`. The challenge is in providing this via a user-friendly API.
