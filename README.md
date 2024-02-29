# rcppclock - TicToc Benchmarking with OpenMP Support for Rcpp

This R Package provides Rcpp bindings for [cppclock](https://github.com/BerriJ/cppclock), a simple tic-toc timer class for benchmarking C++ code. It's not just simple, it's blazing fast! This sleek tic-toc timer class supports overlapping timers as well as OpenMP parallelism. It boasts a microsecond-level time resolution. We did not find any overhead of the timer itself at this resolution. Results (with summary statistics) are automatically passed back to R as a data frame.


## Install

Install rcppclock from CRAN.

```
install.packages("rcppclock")
```

## The Rcpp side of things

Link it in your `DESCRIPTION` file or with `//[[Rcpp::depends(rcppclock)]]`, and load the header library into individual `.cpp` files with `#include <rcppclock.h>`. Then create an instance of the Rcpp::Clock class and use:

`.tic(std::string)` to start a new timer. `.toc(std::string)` to stop the timer.

```c++
//[[Rcpp::depends(rcppclock)]]
#include <rcppclock.h>

std::vector<int> fibonacci(std::vector<int> n)
{
  Rcpp::Clock clock; // Or Rcpp::Clock clock("my_name"); to assign a custom name
  // to the returned dataframe (default is 'times')
  clock.tic("fib_body"); // Start timer measuring the whole function
  std::vector<int> results = n;

  for (int i = 0; i < n.size(); ++i)
  {
    // Start a timer for each fibonacci number
    clock.tic("fib_" + std::to_string(n[i]));
    results[i] = fib(n[i]);
    // Stop the timer for each fibonacci number
    clock.toc("fib_" + std::to_string(n[i]));
  }
  // Stop the timer measuring the whole function
  clock.toc("fib_body");
  return (results);
}
```
Multiple timers with the same name (i.e. in a loop) will be grouped and we report the Mean and Standard Deviation for them. The results will be automatically passed to R as the `clock` instance goes out of scope. You don't need to worry about return statements.

## The R side of things

On the R end, we can now observe the `times` object that was silently passed to the global environment:

```r
[R] fibonacci(n = rep(10 * (1:4), 10))
[R] times
      Name Milliseconds    SD Count
1   fib_10        0.002 0.001    10
2   fib_20        0.048 0.011    10
3   fib_30        5.382 0.070    10
4   fib_40      658.280 1.520    10
5 fib_body     6637.259 0.000     1
```
## OpenMP Support

Since we added OpenMP support, we also have an OpenMP version of the fibonacci function:

```c++
std::vector<int> fibonacci_omp(std::vector<int> n)
{

  Rcpp::Clock clock;
  clock.tic("fib_body");
  std::vector<int> results = n;

#pragma omp parallel for
  for (int i = 0; i < n.size(); ++i)
  {
    clock.tic("fib_" + std::to_string(n[i]));
    results[i] = fib(n[i]);
    clock.toc("fib_" + std::to_string(n[i]));
  }
  clock.toc("fib_body");
  return (results);
}
```
Nothing has to be changed with respect to your `clock` object. The timings show that the OpenMP version is significantly faster (fib_body):

```r
      Name Milliseconds     SD Count
1   fib_10        0.022  0.031    10
2   fib_20        0.132  0.057    10
3   fib_30        8.728  2.583    10
4   fib_40      779.942 91.569    10
5 fib_body      908.919  0.000     1
```

## Limitations

Processes taking less than a microsecond cannot be timed.

## Acknowlegenments

This package (and the underlying [cppclock](https://github.com/BerriJ/cppclock) class) was inspired by [zdebruine](https://github.com/zdebruine)'s [RcppClock](https://github.com/zdebruine/RcppClock). I used that package a lot and wanted to add OpenMP support, alter the process of calculating summary statistics, and apply a series of other small adjustments. I hope you find it useful!