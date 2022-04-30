# CovarianceFunctions.jl
[![CI](https://github.com/SebastianAment/CovarianceFunctions.jl/actions/workflows/CI.yml/badge.svg)](https://github.com/SebastianAment/CovarianceFunctions.jl/actions/workflows/CI.yml)
[![codecov](https://codecov.io/gh/SebastianAment/CovarianceFunctions.jl/branch/main/graph/badge.svg?token=04NSNJC9H1)](https://codecov.io/gh/SebastianAment/CovarianceFunctions.jl)

`CovarianceFunctions.jl`'s primary goal are efficient computations with kernel matrices, also called `Gramian`'s.
To this end, the package implements a *lazy* `Gramian` type, which
can be used to solve linear algebraic problems arising in
kernel methods efficiently without running out of memory.
Further, the code automatically recognizes when certain linear algebraic structures are present and exploits them for computational efficiency.

## Basic Usage
This example shows how to construct a kernel matrix using the `gramian` function and highlights the small memory footprint of the lazy representation and the matrix-vector multiplication with `mul!`.
```julia
using CovarianceFunctions
using LinearAlgebra
k = CovarianceFunctions.MaternP(2); # Matérn kernel with ν = 2.5
d, n = 3, 16384; # generating data with large number of samples
x = [randn(d) for _ in 1:n]; # data is vector of vectors

@time K = gramian(k, x); # instantiating lazy Gramian matrix
  0.000005 seconds (1 allocation: 48 bytes)
size(K)
  (16384, 16384)

a = randn(n);
b = zero(b);
@time mul!(b, K, a); # multiplying with K allocates little memory
    0.584813 seconds (51 allocations: 4.875 KiB)
```
On the other hand, instantiating the matrix densely consumes 2GiB of memory:
```julia
@time M = Matrix(K); # instantiating the matrix costs 2GiB
    0.589848 seconds (53 allocations: 2.000 GiB, 4.13% gc time)
```
## Kernels

The package implements many popularly used covariance kernels.

### Stationary Kernels

We give a list of stationary kernels
whose implementations can be found in src/stationary.jl.

1. `ExponentiatedQuadratic` or `EQ` (also known as RBF)
2. `RationalQuadratic` or `RQ`
3. `Exponential` or `Exp`
4. `GammaExponential` or `γExp`
5. `Matern` for real valued parameters `ν`
6. `MaternP` for `ν = p+1/2` where `p` is an integer
7. `CosineKernel` or `Cos`
8. `SpectralMixture` or `SM`

### Non-Stationary Kernels
The  following non-stationary kernels can be found in src/mercer.jl.

9. `Dot` is the covariance functions of a linear function
10. `Polynomial` or `Poly`
11. `ExponentialDot`
12. `Brownian` is the covariance of Brownian motion
13. `FiniteBasis` is the covariance corresponding to a finite set of basis functions
14. `NeuralNetwork` or `NN` implements McKay's neural network kernel

### Combining Kernels
CovarianceFunctions.jl implements certain transformations and algebraic combinations of kernel functions.
For example,
```julia
using CovarianceFunctions
smooth = CovarianceFunctions.RQ(2); # Rational quadratic kernel with α = 2.
line = CovarianceFunctions.Dot(); # linear kernel
kernel = 1/2 * smooth + line^2 # combination of smooth kernel and quadratic kernel
```
assigns `kernel` a linear combination of the smooth Matérn kernel and a quadratic kernel. The resulting kernel can be evaluated like the base kernel classes:
```julia
d = 3
x, y = randn(d), randn(d)
kernel(x, y) ≈ smooth(x, y) / 2 + line(x, y)^2
  true
```

### Using Custom Kernels
It is simple to use your own custom kernel, e.g.
```julia
custom_rbf(x, y) = exp(-sum(abs2, x .- y)) # custom RBF implementation
```
To take advantage of some specialized structure-aware algorithms, it is prudent to let CovarianceFunctions.jl know about the input type, in this case
```julia
input_trait(::typeof(custom_rbf)) = IsotropicInput()
```

## Toeplitz Structure

The kernel matrix corresponding to isotropic kernel on a regular grid in one dimension exhibits a special Toeplitz structure.
CovarianceFunctions.jl *automatically* detects this structure,
which can be used for `O(n log(n))` multiplies and `O(n²)` direct solves.
```julia
using CovarianceFunctions
using LinearAlgebra
CovarianceFunctions.Exponential(); # exponential kernel
n = 16384;
x = range(-1, 1, n);

@time G = gramian(k, x);
  0.000572 seconds (3 allocations: 128.062 KiB)
typeof(G) # the algorithm automatically recognized the Toeplitz structure
  ToeplitzMatrices.SymmetricToeplitz{Float64}

a = randn(n);
b = zero(a);
@time mul!(b, G, a); # matrix-vector multiplications are very with G are fast O(n log(n))
  0.001068 seconds (57 allocations: 1.504 MiB)
```

In contrast, instantiating the matrix is much slower and memory-expensive:
```julia
@time Matrix(G);
  0.393198 seconds (2 allocations: 2.000 GiB, 0.89% gc time)
```

While the fast multiplicatios can be used in conjunction with iterative solvers,
CovarianceFunctions.jl also implements a `O(n²)` direct solver called `levinson`:
```julia
using CovarianceFunctions: levinson
@time x_fast = levinson(G, b); # same as G \ b
  0.172557 seconds (6 allocations: 384.141 KiB)
```
whereas naïvely, this would take two orders of magnitude longer for this problem:
```julia
julia> @time x_naive = Matrix(G) \ b;
 10.494046 seconds (8 allocations: 4.000 GiB, 1.34% gc time)
```
Notably, the results are equal to machine precision:
```julia
x_naive ≈ x_fast
  true
```

## Gradient Kernels
When conditioning Gaussian processes on gradient information,
it is necessary to work with `d × d` *matrix-valued* gradient kernels,
where `d` is the dimension of the input.
CovarianceFunctions.jl implements an automatic structure derivation engine with a `O(d)` complexity for a large range of kernel functions
and contains a generic fallback with the regular `O(d²)` complexity.

For example,
```julia
using CovarianceFunctions
using LinearAlgebra
k = CovarianceFunctions.MaternP(2); # Matérn kernel with ν = 2.5
g = CovarianceFunctions.GradientKernel(k);
d, n = 1024, 1024; # generating high-d data with large number of samples
x = [randn(d) for _ in 1:n]; # data is vector of vectors
@time G = gramian(g, x); # instantiating lazy gradient kernel Gramian matrix
  0.000013 seconds (1 allocation: 96 bytes)
size(G) # G is n*d by n*d
  (1048576, 1048576)
a = randn(n*d);
b = zero(b);
@time mul!(b, G, a); # multiplying with G allocates little memory
  0.394388 seconds (67 allocations: 86.516 KiB)
```
Note that the last multiplication was with a million by million matrix,
which would be impossible without CovarianceFunctions.jl's lazy and structured representation of the gradient kernel matrix.

Furthermore, CovarianceFunctions.jl can compute structured representations of more complex and composite kernels maintaining the fast `O(d)` matrix vector multiply.
The following exemplifies this with a combination of Matérn, quadratic, and neural network kernels using 1024 points in 1024 dimensions:
```julia
matern = CovarianceFunctions.MaternP(2); # matern
quad = CovarianceFunctions.Dot()^2; # second kernel component
nn = CovarianceFunctions.NN(); # neural network
kernel = matern + quad + nn;
g = CovarianceFunctions.GradientKernel(kernel);
@time G = gramian(g, x);
  0.000313 seconds (175 allocations: 56.422 KiB)
size(G) # G is n*d by n*d
  (1048576, 1048576)
@time mul!(b, G, a);
    2.649361 seconds (27.26 M allocations: 1.141 GiB, 2.68% gc time)
```

It is possible to hook a costum kernel into CovarianceFunctions.jl's automatic structure derivation engine, by specifying its input type
using the `input_trait` function.
Basic input traits amenable to specializations are `IsotropicInput`, `DotProductInput`, and  `StationaryLinearFunctionalInput`.
Further transformations and combinations of kernels are also supported, as well as efficient `O(d²)` operations with certain Hessian kernels, in constrast to the naïve `O(d⁴)` complexity.
The main files containing the implementation are src/gradient.jl, src/gradient_algebra.jl, and src/hessian.jl

## Sparsification
In high dimensions, kernel matrices associated with exponentially-decaying kernels are very likely sparse.
This is because the average distance between points grows with increasing dimension.
CovarianceFunctions.jl contains a sparsification algorithm that extends SparseArrays.jl's `sparse` function,
guaranteeing a user-defined element-wise accuracy.
```julia
using CovarianceFunctions
using SparseArrays

k = CovarianceFunctions.EQ()
d, n = 32, 16384;
X = randn(d, n);
G = gramian(k, X);

@time S = sparse(G, 1e-6); # sparsification with 1e-6 tolerance
  7.208343 seconds (407.54 k allocations: 73.094 MiB)
nnz(S) # number of non-zeros in S
  594480
nnz(S) / n^2 # over 99% of entries are sparse
  0.0022146105766296387
```
Subsequent matrix-vector multiplications with `S` are very fast:
```julia
@time mul!(b, S, a); # sparse multiply
    0.000451 seconds
@time mul!(b, G, a); # lazy dense multiply
  0.949835 seconds (61 allocations: 5.594 KiB)
```
The bottleneck in the computation of `S` is NearestNeighbors.jl's `inrange` function.
We believe that it is possible to further accelerate this computation.

## Barnes-Hut
The Barnes-Hut algorithm has its origins in accelerating gravity simulutations, but a variant of it can be applied to kernel matrices arising from more general kernels,
allowing for an approximate matrix-vector multiply in `O(n⋅log(n)⋅d)` operations.
CovarianceFunctions.jl contains the `BarnesHutFactorization` structure whose constructor can be called on a `Gramian` matrix and applies the fast transform whenever `*` or `mul!` is called.


<!-- * more covariance matrix factorizations (HODLR), or approximations (SKI, Nystrom, Random Kitchen Sinks, Fast Food)
* GPLVMs
* chebyshev interpolation
* functional eigendecomposition of kernels as pre-processing step
* `const SmoothKernels = Union{EQ, RQ} -> trylowrank(::SmoothKernels, n::Int) = n > 128` -->
