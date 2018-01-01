Introduction
============

This is a short blog post of a series around the importance of writing good code for machine learning and data science projects.

A few months back I answered a question on Quora around [suggestions to give to younger data scientists](https://www.quora.com/As-a-data-scientist-what-tips-would-you-have-for-a-younger-version-of-yourself/answer/Oscar-Cassetti%5D). One of the key point I made was the importance of good coding practices when doing data science and machine learning.

More often I find myself in situations where machine learning and data science projects suffer setback because of lack of scalability, general performance issues, lack of testing which presents to reuse the code or bring the models to production.

It seems that for some unknown reasons data scientists have forgotten that software engineering matters. This series aim to bring to the light some these topics.

Writing efficient code for the framework you are using
=================================================

In the past year I kept track of how often I had to fix both my code or someone else code due to performance issues. One thing that came clear is that the first source of poor performance is the lack of understanding of the actual framework or programming language one is using as well as not doing enough bench marking. The second reason is that when addressing performance issues the code becomes less extensible and even less maintainable. Essentially it seems quite hard to find a good balance between writing code that is easy to maintain as well performant. But this does not have to be the case. Here I am going to handcraft a problem similar to something I have seen in real production environment.

Imagine we need to write a simulation engine for a non trivial random walk where we have three random walkers which can move forward according to a given probability distribution. We have a further constraint that is their journey could be randomly shorten according to a different distribution, that is after a random walker decides to move forward according to this distribution his journey can be reduced by a certain factor which is sample from a different distribution. This abstract model can be applied to various problems such as predicting the number faulty parts or the usage of resources. For this reason we want to be able to design the engine in such a way that we can dynamically change the two key behaviors that is how walkers move forward and how much their journey is reduced at each step.

I am going to show you an example written in R to highlight the issue I mentioned at the beginning, that is, the lack of understanding of the framework can lead to bad performances.

One way to approach the problem is to define a generic function `runSim` which takes 3 parameters:

1.  The number of steps to simulate `n`
2.  A function called `forwardDist` return a list of 3 elements describing how the walker will move forward
3.  A function called `reductionDist` return a list of 3 elements describe how much is the walk is reduced.

The function `runSim` will return a matrix *n* × 3 containing how much the walker will move at each step.

As stated before our initial requirement is to have a generic enough engine where different `forwardDist` and `reductionDist` function can be applied.

To focus on the key problem I am skipping some all parameter validation which we should be doing within `runSim`.

Assuming one has some experience with R an initial version of `runSim` would look like this:

``` r
runSim <- function(n, forwardDist, reductionDist) {
  simVal <- as.double( replicate(n, Map('*', forwardDist(), reductionDist())))

  matrix(simVal, ncol = 3, byrow = TRUE)
}
```

Here we made some key design assumption: 1. We assume that `forwardDist` and `reductionDist` will return a list of 3 values and the ordering of two list is the same, that is we can multiply item by item the two list. 2. We design `forwarDist` and `reductionDist` to return one value for each walker 3. We use the R `replicate` to repeat the simulation *n* times and the `Map` function is used to multiply the two list.

The design meet our requirement that is to allow different `forwardDist` and `reductionDist`.

For example to start off and to test everything is well we could implement constant `forwardDist` and `reductionDist` function.

Below some code:

``` r
constantForwardDist <- function(){
  list(c1=10, c2=30, c3=25)
}

constantReductionDist <- function(){
  list(c1=0.9, c2=0.7, c3=0.1)
}
```

This will allow us to run a sanity check around `runSim`, here I am using `RUnit` as framework for unit testing.

``` r
library(RUnit)

expectedConstantWalk <-  matrix(rep(c(9, 21, 2.5), 10), ncol = 3, byrow = TRUE)
actualConstantWalk <- runSim(10, constantForwardDist, constantReductionDist)

expectedConstantWalk
```

    ##       [,1] [,2] [,3]
    ##  [1,]    9   21  2.5
    ##  [2,]    9   21  2.5
    ##  [3,]    9   21  2.5
    ##  [4,]    9   21  2.5
    ##  [5,]    9   21  2.5
    ##  [6,]    9   21  2.5
    ##  [7,]    9   21  2.5
    ##  [8,]    9   21  2.5
    ##  [9,]    9   21  2.5
    ## [10,]    9   21  2.5

``` r
actualConstantWalk
```

    ##       [,1] [,2] [,3]
    ##  [1,]    9   21  2.5
    ##  [2,]    9   21  2.5
    ##  [3,]    9   21  2.5
    ##  [4,]    9   21  2.5
    ##  [5,]    9   21  2.5
    ##  [6,]    9   21  2.5
    ##  [7,]    9   21  2.5
    ##  [8,]    9   21  2.5
    ##  [9,]    9   21  2.5
    ## [10,]    9   21  2.5

``` r
checkEquals(expectedConstantWalk, actualConstantWalk)
```

    ## [1] TRUE

So we might quite happy with that and we might decide to move on quickly and without doing any benchmarks or implementing more complex `forwardDist`, and `reductionDist` to pass to our simulation engine.

Everything seems to work fine so we tart to extend our project by defining non constant functions e.g. we define a function `normalForwardDist` where the forward movement are sampled according to a normal distribution and each walker has different parameters for the distributions

{% include image.html img="/assets/images/ml-ds-best-practices/unnamed-chunk-4-1.png" caption="Execution comparisons" %}


``` r
normalForwardDist <- function(){
  list(
       c1=rnorm(mean=10, sd=0.5, n=1), 
       c2=rnorm(mean=30, sd=0.5, n=1), 
       c3=rnorm(mean=25, sd=0.5, n=1)
      )
}


runSim(10, normalForwardDist, constantReductionDist)
```

    ##           [,1]     [,2]     [,3]
    ##  [1,] 8.796548 20.70411 2.525997
    ##  [2,] 9.129752 20.92412 2.492297
    ##  [3,] 8.143187 20.88580 2.537238
    ##  [4,] 8.922990 21.15933 2.427785
    ##  [5,] 9.516422 20.32564 2.416143
    ##  [6,] 8.903545 21.33803 2.472923
    ##  [7,] 9.283903 20.80704 2.558777
    ##  [8,] 8.721221 21.30778 2.561612
    ##  [9,] 8.948482 21.45228 2.490752
    ## [10,] 8.927771 21.44081 2.540795

All looks good, we can customize the logic inside our input function and we do not really need to touch the engine. However if we start plugging large number of $n 10^5 $ we notice things get quite slow around 2 seconds to generate the walk.

``` r
system.time(runSim(1e5, normalForwardDist, constantReductionDist))
```

    ##    user  system elapsed 
    ##   2.381   0.007   2.396

``` r
set.seed(42)
normalFowrardWalk42 <- runSim(1, normalForwardDist, constantReductionDist)
```

We want to do better because we are going to run `runSim` with large *n* and potentially with more complex functions.

The first thing we notice is that in `runSim` we are calling `forwardDist` and `reductionDist` *n* times, most people learn that R is must faster at dealing with vectorized operations. We also notice that for `rnorm` and all random generating function could easily and faster return *n* variables rather than 1 at the time.

``` r
library(rbenchmark)
benchmark( repeatN=rep(rnorm(10, 0.5), 1e6),
           returnN=rnorm(10, 0.5, 1e6)
          )
```

    ##      test replications elapsed relative user.self sys.self user.child
    ## 1 repeatN          100   1.526       NA      1.52        0          0
    ## 2 returnN          100   0.000       NA      0.00        0          0
    ##   sys.child
    ## 1         0
    ## 2         0

So we can rewrite an version of `runSim` to take advantage of vector-generating functions.

``` r
runSim <- function(n, forwardDist, reductionDist) {
  simVal <- as.double(unlist(Map('*', forwardDist(n), reductionDist(n))))
  matrix(simVal, ncol = 3)
}
```

Note that we need to make a few change inside the function including passing *n* to `forwardDist` and `reductionDist`. This by doing so we have changed the *signature* of `forwardDist` and `reductionDist` and therefore we need to rewrite them again

We start to rewrite our constant ones after all we want to test that this *refactoring* process is not going to impacting the actual result.

``` r
constantForwardDist <- function(n){
  list(c1=rep(10,n), c2=rep(30,n), c3=rep(25,n))
}

constantReductionDist <- function(n){
  list(c1=rep(0.9,n), c2=rep(0.7,n), c3=rep(0.1,n))
}
```

We run our regression test to make sure we did not break anything

``` r
checkEquals(expectedConstantWalk, runSim(10, constantForwardDist, constantReductionDist)) 
```

    ## [1] TRUE

Now we can proceed rewriting `normalForwardDist` as

``` r
normalForwardDist <- function(n){
  list(
       c1=rnorm(mean=10, sd=0.5, n=n), 
       c2=rnorm(mean=30, sd=0.5, n=n), 
       c3=rnorm(mean=25, sd=0.5, n=n)
      )
}
```

as sanity check we can reset the random seed and compare that the new version of `normalForwardDist` returns the same result which we have store `normalFowrardWalk42` for the first step of the walk.

``` r
set.seed(42)
acutalForwardNormal <- runSim(1, normalForwardDist, constantReductionDist)
checkEquals( normalFowrardWalk42, acutalForwardNormal)
```

    ## [1] TRUE

``` r
# We store this for further tests
set.seed(42)
normalFowrardWalk42e2 <- runSim(100, normalForwardDist, constantReductionDist)
```

Further steps of the walk will be different as the in the first approach we generate 1 random variate at the time for each walker whereas in the second approach we generate *n* variate for each walker at one time.

Now the question is how faster is this approach compared the previous one, i.e. what improvement did we make?

{% include image.html img="/assets/images/ml-ds-best-practices/unnamed-chunk-15-1.png" caption="Execution comparisons" %}

We made a substation improvement and for large *n* the new version is 10x faster.

We could stop here but out of curiosity we want to understand what if we write a specific simulation engine for the problem `normalForwardDist` if this would be substantially more performant. We want to answer how much performance do we need to trade off in exchange of flexibility ?

We could write `runSim` to be a specific engine for the *normal* distribution in this way:

``` r
runSim <- function(n) {
  
  simVal <- c( rnorm(mean=10, sd=0.5, n=n) * 0.9, 
               rnorm(mean=30, sd=0.5, n=n) * 0.7, 
               rnorm(mean=25, sd=0.5, n=n)* 0.1  
            )
  
  matrix(simVal, ncol = 3)
}
```

This implementation of `runSim` is missing all the flexibility of our previous version.

As we did before we first test we are getting the same result, after all we need to make sure we are comparing apples with apples. This approach should return the same result as our second version even for *n* &gt; 1.

``` r
set.seed(42)
actualNormalWalk <- runSim(100)

checkEquals(normalFowrardWalk42e2, actualNormalWalk)
```

    ## [1] TRUE

The next question is how much faster is this new version (version 3) compared to version 2.
{% include image.html img="/assets/images/ml-ds-best-practices/unnamed-chunk-19-1.png" caption="Execution comparisons" %}


The previous graph is not very comforting the trade off between performance and flexibility is close to 10x for large values of *n*.

This where I see most people stop and write extremely custom code that is hardly re-usable and quite hard to test.

Do we really have to stop here?

Well turns out that we spend some time understanding how the framework we are using works and we start profiling the code we could try to make so important changes.

For example we can rewrite `forwardDist` and `reductionDist` in a slightly different way. Instead of returning a numeric value for each walker we can return a function.

``` r
constantForwardDist <- function(){
  list(
       c1=function(n){rep(10,n)}, 
       c2=function(n){rep(30,n)}, 
       c3=function(n){rep(25,n)}
      )
}

constantReductionDist <- function(){
  list(
       c1=function(n){rep(0.9,n)}, 
       c2=function(n){rep(0.7,n)}, 
       c3=function(n){rep(0.1,n)}
      )
}


normalForwardDist <- function(){
  list(
       c1=function(n){rnorm(mean=10, sd=0.5, n=n)}, 
       c2=function(n){rnorm(mean=30, sd=0.5, n=n)}, 
       c3=function(n){rnorm(mean=25, sd=0.5, n=n)}
       )
}
```

The differences with this approach is that when we call the function instead of a list of size 3 containing 3 vectors of size *n* we get a list of size 3 containing 3 functions

``` r
normalForwardDist()
```

    ## $c1
    ## function (n) 
    ## {
    ##     rnorm(mean = 10, sd = 0.5, n = n)
    ## }
    ## <environment: 0x55c615af85a0>
    ## 
    ## $c2
    ## function (n) 
    ## {
    ##     rnorm(mean = 30, sd = 0.5, n = n)
    ## }
    ## <environment: 0x55c615af85a0>
    ## 
    ## $c3
    ## function (n) 
    ## {
    ##     rnorm(mean = 25, sd = 0.5, n = n)
    ## }
    ## <environment: 0x55c615af85a0>

We can also start to think about re-writing `runSim` to work with and arbitrary number of walkers and for `good measure` we also match the `forwardDist` and `reductionDist` for each walker meaning that we don't need to force ordering in the `forwardDist` and `reductionDist` functions.

``` r
runSim <- function(n, forwardDist, reductionDist, walkers=c("c1", "c2", "c3")) {
  
  simValMatix <- matrix(NA, ncol = length(walkers), nrow = n)
  
  for(i in seq_along(walkers)){
    walker <- walkers[i]
    walkerForwardDist <- forwardDist()[[walker]]
    walkerReductionDist <- reductionDist()[[walker]]
    simValMatix[, i] <- walkerForwardDist(n) * walkerReductionDist(n)
  }
  simValMatix
}
```

As per usual we run our test to make sure we are still doing a good job, first against the constant functions

``` r
actualConstantWalk <- runSim(10, constantForwardDist, constantReductionDist, walkers = c("c1", "c2", "c3"))
checkEquals(expectedConstantWalk, actualConstantWalk)
```

    ## [1] TRUE

and the against the `normalForwardDist`

``` r
set.seed(42)
actualNormalWalk <- runSim(100, normalForwardDist, constantReductionDist, walkers = c("c1", "c2", "c3"))
checkEquals(normalFowrardWalk42e2, actualNormalWalk)
```

    ## [1] TRUE

We are happy that the refactoring is not impacting the result of the simulation but are we getting better speed

We are happy that the refactoring is not impacting the result of the simulation but are we getting better speed

{% include image.html img="/assets/images/ml-ds-best-practices/unnamed-chunk-27-1.png" caption="Execution comparisons" %}
{% include image.html img="/assets/images/ml-ds-best-practices/unnamed-chunk-27-2.png" caption="Execution comparisons" %}



As you can see from the graph above the overhead of more flexible system is pretty much negligible.

The two key conclusions from this short article are:

1.  A good understanding of framework will help you optimize.
2.  You must have unit test in order to be able to start any refactoring.
3.  Quite often you do not need to trade off flexibility to performance.
