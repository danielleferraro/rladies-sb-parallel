A gentle introduction to parallel processing in R
================
Danielle Ferraro
May 26, 2021

-   [Overview and setup](#overview-and-setup)
    -   [Install packages](#install-packages)
-   [1. `parallel`](#1-parallel)
    -   [`mclapply()`](#mclapply)
    -   [`parLapply()`](#parlapply)
-   [2. `foreach`](#2-foreach)
-   [3. `furrr`](#3-furrr)
-   [4. `future.apply`](#4-futureapply)

# Overview and setup

This file contains sample code snippets that demonstrate how parallel
processing is implemented in R for the R-Ladies Santa Barbara meetup on
May 26, 2021.

## Install packages

During this session, we’ll cover a few (of the many) packages and
functions for parallelizing your R code. These include the base package
`parallel`, as well as the packages `foreach` (for which we will also
need the package `doParallel`), `furrr`, and `future.apply`. If the
these packages aren’t already installed on your local computer, please
install the following packages by un-commenting and running these lines:

``` r
# install.packages("foreach")
# install.packages("doParallel")
# install.packages("furrr") 
# install.packages("future.apply")
```

We will also use the packages `dplyr` for minor data wrangling and
`tictoc` for timing how long code takes to run. Install those packages
if needed, and then load them.

``` r
# install.packages("dplyr)
# install.packages("tictoc")
library(dplyr)
library(tictoc)
```

# 1. `parallel`

The `parallel` package is a base R package that comes with your R
installation, so we just have to load it.

``` r
library(parallel)
```

One of the first things to do with `parallel` is to check how many cores
your computer has available to use. This number is easily returned with
the `detectCores()` function.

``` r
detectCores()
```

    ## [1] 8

The computer I’m working on gives me 8 cores to play with. How many
cores you use when parallel processing is up to you. If your computer
has *n* cores, some people suggest using *n-1* cores so that you have
space to run other processes in the background. But for tasks that are
particularly computationally intensive, it might make sense to shut down
other applications on your computer and use all *n* cores. And remember
that there is overhead for setting up parallel processors, so take care
when using computers or servers with large numbers of cores available.
Note that some packages/functions default to using a certain number of
cores, while others require you to specify the number you want to use.

## `mclapply()`

The simplest way to use the `parallel` package is with the function
`mclapply()`. You may already be familiar with the function `lapply()`,
which applies a function over the elements of a list. `mclapply()` is
the parallelized version of this function! **Note:** This function
relies on a method of parallelization (called forking) that does not
work on Windows operating systems. We will go through Windows-friendly
examples later.

Let’s try it out! First, let’s write a simple but purposefully slow
function for our testing purposes. This sample function and some of the
following exercises are credited to Grant McDermott and can be found in
his very helpful lecture notes on parallel programming, linked
[here](https://raw.githack.com/uo-ec607/lectures/master/12-parallel/12-parallel.html).

``` r
# Function to calculate the square of a number, then put R to sleep for two seconds
slow_square <- function(x) {
  x_squared <- x^2 
  d <- tibble(value = x, 
              value_squared = x_squared)
  Sys.sleep(2)
  return(d)
}
```

We can now test out our slow function on a few numbers using lapply, and
then bind the results into a tibble with `bind_rows()`. We’ll use
`tic()` and `toc()` to see how long this takes.

``` r
tic("Time to beat")
lapply(1:5, slow_square) %>% bind_rows()
```

    ## # A tibble: 5 x 2
    ##   value value_squared
    ##   <int>         <dbl>
    ## 1     1             1
    ## 2     2             4
    ## 3     3             9
    ## 4     4            16
    ## 5     5            25

``` r
toc()
```

    ## Time to beat: 10.21 sec elapsed

Now let’s parallelize our code with `mclapply()`:

``` r
n_cores <- detectCores() - 1
tic("mclapply")
mclapply(1:5, slow_square, mc.cores = n_cores) %>% bind_rows()
```

    ## # A tibble: 5 x 2
    ##   value value_squared
    ##   <int>         <dbl>
    ## 1     1             1
    ## 2     2             4
    ## 3     3             9
    ## 4     4            16
    ## 5     5            25

``` r
toc()
```

    ## mclapply: 2.035 sec elapsed

## `parLapply()`

If you have a Windows machine, the function you’ll use from the
`parallel` package is `parLapply` (instead of `mclapply()`). There’s a
little more setup involved. First you need to make a local cluster with
`makePSOCKcluster()`. Here, “cluster” just refers to the collection of
cores on our personal computer.

``` r
cluster <- makePSOCKcluster(n_cores)
tic("parLapply")
parLapply(cluster, 1:5, slow_square) # This will generate an error
toc()
```

An error! With `parLapply()`, we need to explicitly “export” any
packages needed into our cluster using `clusterEvalQ()`. In this case,
we need to export `tibble`. This is a result of the socket method of
parallel processing (vs. forking, which does not work on Windows).

``` r
cluster <- makePSOCKcluster(n_cores)
clusterEvalQ(cluster, library("tibble"))
```

    ## [[1]]
    ## [1] "tibble"    "stats"     "graphics"  "grDevices" "utils"     "datasets" 
    ## [7] "methods"   "base"     
    ## 
    ## [[2]]
    ## [1] "tibble"    "stats"     "graphics"  "grDevices" "utils"     "datasets" 
    ## [7] "methods"   "base"     
    ## 
    ## [[3]]
    ## [1] "tibble"    "stats"     "graphics"  "grDevices" "utils"     "datasets" 
    ## [7] "methods"   "base"     
    ## 
    ## [[4]]
    ## [1] "tibble"    "stats"     "graphics"  "grDevices" "utils"     "datasets" 
    ## [7] "methods"   "base"     
    ## 
    ## [[5]]
    ## [1] "tibble"    "stats"     "graphics"  "grDevices" "utils"     "datasets" 
    ## [7] "methods"   "base"     
    ## 
    ## [[6]]
    ## [1] "tibble"    "stats"     "graphics"  "grDevices" "utils"     "datasets" 
    ## [7] "methods"   "base"     
    ## 
    ## [[7]]
    ## [1] "tibble"    "stats"     "graphics"  "grDevices" "utils"     "datasets" 
    ## [7] "methods"   "base"

``` r
tic("parLapply")
parLapply(cluster, 1:5, slow_square) %>% bind_rows()
```

    ## # A tibble: 5 x 2
    ##   value value_squared
    ##   <int>         <dbl>
    ## 1     1             1
    ## 2     2             4
    ## 3     3             9
    ## 4     4            16
    ## 5     5            25

``` r
toc()
```

    ## parLapply: 2.052 sec elapsed

At this scale, `parLapply()` was only barely slower than `mclapply()`.

# 2. `foreach`

Another implementation of parallel processing is the package `foreach`.
It works on all operating systems, and the primary function `foreach()`
enables you to use loops rather than the apply format we used in the
previous example.

The code is formatted as follows:

`y <- foreach(...) %dopar% { ... }`

where the `%dopar%` operator indicates that the computation will be done
in parallel.

The setup is a little bit more involved in that we must first register a
“parallel backend” to link the `foreach` package to the type of parallel
processing we want to conduct. In this example, we will use the
`doParallel` library, which makes use of the base `parallel` package.
Alternatively, we could use the `doFuture` library, which would use the
`future` package as the backend. Either will work on any operating
system.

``` r
# Load packages
library(foreach)
library(doParallel)

# Register the parallel processing backend
registerDoParallel(n_cores) # If left blank, will default to half the number of cores detected by the {parallel} package
```

By default, results are returned as a list. The `.combine` argument in
`foreach()` allows you to specify a function to return a different
class, so we can name `bind_rows()` right there.

``` r
tic("foreach")
foreach(i = 1:5, .combine = bind_rows) %dopar% {slow_square(i)}
```

    ## # A tibble: 5 x 2
    ##   value value_squared
    ##   <int>         <dbl>
    ## 1     1             1
    ## 2     2             4
    ## 3     3             9
    ## 4     4            16
    ## 5     5            25

``` r
toc()
```

    ## foreach: 2.053 sec elapsed

**Note:** For debugging, it may be helpful to replace `%dopar%` with
`%do%`, which will revert the expression back to serial processing.

# 3. `furrr`

The `furrr` package is part of the `future` ecosystem, which is a newer
method of parallel programming in R that is a bit more user-friendly and
supportive of any operating system. The functions in `furrr` are meant
to be near-replacements for `purrr`, but in parallel. All your favorite
`purrr` functions have a `furrr` equivalent: `map()`, `walk()`, etc. Fun
fact: Add a progress bar with the argument `.progress = TRUE`!

``` r
library(furrr)
plan(multisession, workers = n_cores) # Tell R we want to run things in parallel. If workers is left, blank, it will default to the number of available cores. 

tic("future_map_dfr")
future_map_dfr(1:10, slow_square, .progress = TRUE)
```

    ## # A tibble: 10 x 2
    ##    value value_squared
    ##    <int>         <dbl>
    ##  1     1             1
    ##  2     2             4
    ##  3     3             9
    ##  4     4            16
    ##  5     5            25
    ##  6     6            36
    ##  7     7            49
    ##  8     8            64
    ##  9     9            81
    ## 10    10           100

``` r
toc()
```

    ## future_map_dfr: 5.208 sec elapsed

I definitely recommend reading more about the different options for
parallel processing strategies here (multisession vs. multicore). Run
`?plan` for some helpful documentation. Long story short: multicore is
typically faster, but doesn’t work on Windows and may be unstable in
IDEs such as RStudio, while multisession is slower, but will work on any
operating system and as well as within IDEs. For the purposes of this
workshop, we will use multisession.

# 4. `future.apply`

The `future.apply` package is also part of the `future` ecosystem, and
provides parallel versions of base `apply()` functions. This is similar
to what `mclapply()` did in the first example, but uses a different
backend.

``` r
library(future.apply)
# plan(multisession, workers = n_cores) # Already set up above

tic("future.apply")
future_lapply(1:5, slow_square) %>% bind_rows()
```

    ## # A tibble: 5 x 2
    ##   value value_squared
    ##   <int>         <dbl>
    ## 1     1             1
    ## 2     2             4
    ## 3     3             9
    ## 4     4            16
    ## 5     5            25

``` r
toc()
```

    ## future.apply: 2.187 sec elapsed
