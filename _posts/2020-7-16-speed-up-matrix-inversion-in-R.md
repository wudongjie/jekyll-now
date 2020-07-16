---
layout: post
title:  "Speed up matrix inversion in R"
---


Recently, I have been working on a project which requires translating Matlab code to R code. This project uses a fairly large dataset and needs to recursively compute the inversion of a matrix (around 1000 by 1000) more than 500 times. In Matlab, it spends less then 5 minutes to run the whole computation; while in R, my initial translation ran more than an hour.

In R, there are several ways to compute the matrix inversion.

1. `solve()` function

Solve function is initially designed to solve equations. For example, for `a %*% x = b`, we can use `solve(a, b)` to compute `x`. However, when we only provide one parameter, `solve(a)` returns the inverse of `a`.

2. The Choleski decomposition (`chol2inv()`)

Another way to compute matrix inversion is to use the choleski decomposition, i.e. `cho2inv(chol(x))`. `chol()` computes the Choleski factorization of a real **symmetric positive-definite square matrix**. Therefore, we shall first make sure that the matrix `x` is a symmetric positive-definite matrix. If the matrix is not symmetric and positive-definite, an error will occur. 

3. `pd.solve()`

Similar to `solve()`, `pd.solve()` also computes the matrix inversion or solve an equation. The difference is that `pd.solve()` requires a matrix to be positive definite. This function is provided by `mnormt` package.

4. `arma::inv()` from `RcppArmadillo` package

C++ is known for its performance. One could embed C++ code in an R program by using the `Rcpp` package. `RcppArmadillo` is a linear algebra library allowing users to use C++ to boost computation in R.

In this package, we can use `arma::inv()` to compute matrix inversion. Additionally, if the matrix is positive definite, we can use `arma::inv_sympd()`. Below is the implementation of matrix inversion using `arma::inv()`.

```
  Rcpp::cppFunction("arma::mat armaInv(const arma::mat & x) { return arma::inv(x); }", depends="RcppArmadillo")
  y = armaInv(x)
```

5. `solve()` and `chol2inv()` in `Matrix` package.

`Matrix` is a package of classes and methods dealing with Matrix computations. These classes and methods are known as the `S4`. Different from the conventional data type in R, `Matrix` defines different classes for different matrices. For example, the standard dense numeric matrix is defined as `dgeMatrix` class. If a matrix is symmetric, it can be transformed into a symmetric class `symmetricMatrix` using `forceSymmetric` method. 

Thanks to the rich data type `Matrix` provides, when using `Matrix::solve()`, the function will first check the data type of the given matrix, if the matrix is a sysmetric and positive definite matrix, the function will use the Choleski decomposition to speed up the matrix inversion.

### Test 
I randomly generated a 1000 by 1000 matrix and ran the inversion 100 times for each method introduced above. The following is the benchmark test result.

| test| replications| elapsed| relative| user.self| sys.self|
|----|----|----|----|----|----|
| solve_s3| 100| 49.45| 4.732| 48.97| 0.45|
| chol_s3| 100| 25.53| 2.443| 25.19| 0.34|
| inverse_cpp_pd_s3| 100| 26.50| 2.536| 25.76| 0.62|
| inverse_cpp_s3| 100| 26.59| 2.544| 25.91| 0.60|
| Matrix::chol2inv| 100| 10.45| 1.000| 10.31| 0.14|
| Matrix::solve_s4| 100| 16.00| 1.531| 15.94| 0.06|

Clearly, the common `solve()` is the slowest (49.45s for 100 rounds of matrix inversion). If the matrix is symmetric positive-definite, using `chol2inv()` could reduce a half of the runtime. Using the `RcppArmadillo` can also achieve similar reduction. Surprisingly, the inversion using the `Matrix` package performs even much better than the `RcppArmadillo` packages. If the matrix is symmetric and positive definite, using `Matrix::chol2inv()` improves the performance by 4.7 times comparing to the `solve()`! 

However, even use the outperformer `Matrix::chol2inv()`, my program is still around 10 times slower than the matlab. After doing some research, I found [Microsoft R Open](https://mran.microsoft.com/documents/rro/installation). 

Just by using this alternative version, I achieve the 10-fold performance! Here is the benchmark test using Microsoft R Open.

| test| replications| elapsed| relative| user.self| sys.self |
|------------------|-------------|--------|---------|----------|-----------|
| solve_s3 | 100| 3.42| 3.758| 39.63| 0.50 |
| chol_s3 | 100| 1.67| 1.835| 18.63| 0.42 |
| Matrix::chol2inv | 100| 0.91| 1.000| 10.58| 0.21 |
| Matrix::solve_s4 | 100| 1.30| 1.429| 8.42| 0.17 |

`Matrix::chol2inv()` is still the fastest. The runtime is only 0.91s comparing to 49.45s of `solve()` in the previous version. I achieved a more than 50 times boost of performance!