---
layout: post
title: "Tutorial: Optimising R code"
tags: [Optimisation, R, microbenchmark]
permalink: optimisation
comments: true
---



The R language is very good for statistical computations, due to its strong functional capabilities, its open source philosophy, and its extended package ecosystem. However, it can also be quite slow, because of some [design choices](http://adv-r.had.co.nz/Performance.html#language-performance) (e.g. lazy evaluation and extreme dynamic typing). 

This tutorial is mainly based on Hadley Wickam's book [Advanced R](http://adv-r.had.co.nz/).

### Before optimising...

First of all, before we start optimising our R code, we need to ask ourselves a few questions:

1. Is my code doing what I want it to do?

2. Do I really need to make my code faster?

3. Is considerable speed up even possible?

For the first point, is to useful to keep the following quote in mind:

> Premature optimisation is the root of all evil. (Donald Knuth)

When writing code, we have a specific task in mind, and this has to be our main focus. To make our lives simpler, it is important to write simple, understandable code to start with; it is considerably easier to debug simple code than complex code. Only when we are certain that our code is correct can we turn to the next point.

An R script that will be used only once does not necessarily need to be optimised, and writing your code quickly will probably be more important than writing code that *runs* quickly. 

Finally, when your code is bug-free and you do need to make it faster, you need to identify the bottle-necks, the places where your code spends the most time, and you need a way to compare the speed of multiple expressions. These two things are known as *profiling* and *benchmarking*; we will treat them both in what follows.

### Benchmarking

There are a few ways to time your code. The simplest is to use the function ```system.time()```:


{% highlight r %}
system.time(x <- runif(10^6))
{% endhighlight %}



{% highlight text %}
## utilisateur     système      écoulé 
##        0.11        0.00        0.11
{% endhighlight %}

However, this timing will depend on your OS, it will generally differ from one run to the other, and therefore it is not clear how to use it to compare two or more expressions. Nonetheless, it is probably the way to go when you want to time expressions that take a long time to run. 

For comparisons, we will use the ```microbenchmark``` package. Its ```microbenchmark``` function will run a series of expressions multiple times and return a distribution of running times. It can also be used with ```ggplot2``` to output a nice graphical display of the comparisons.


{% highlight r %}
library(microbenchmark); library(ggplot2)
# Create a dataframe with random values
data <- data.frame("column1" = runif(1000), 
                   "column2" = rnorm(1000))

compare <- microbenchmark::microbenchmark( "extract1" = {
  data[666, 1]
}, "extract2" = {
  data[[1]][666]
})

compare
{% endhighlight %}



{% highlight text %}
## Unit: microseconds
##      expr    min     lq     mean  median      uq     max neval
##  extract1 22.807 24.518 43.44227 25.3735 31.0755 786.850   100
##  extract2 11.403 12.544 16.26744 13.6850 14.8250 113.467   100
{% endhighlight %}



{% highlight r %}
ggplot2::autoplot(compare)
{% endhighlight %}

![plot of chunk unnamed-chunk-3](figure/source/2015-09-11-optimisation/unnamed-chunk-3-1.png) 

So we compared two ways of extracting an element in the data frame: first, we think of ```data``` as being matrix-like and extract based on its row and column position; or we remember that data frames are actually *lists* and extract the element by using the list methods. As we can see, the latter is about twice faster than the former. However, by looking at the units (i.e. microseconds), we see that the difference is quite minimal and unlikely to improve your code (unless you perform this operation millions of times). 

The ```microbenchmark``` function is the main tool we will use below to benchmark snippets of code and, most importantly, compare different implementations of a same idea.

### Before going into profiling...

As I mentionned above, R has the reputation of being slow. However, we have to keep in mind that most of the time, slow R code can be made faster by simply coding in a way that is more natural for R. And to understand what is natural, we need to understand a bit more about how R works.

#### R is an interpreted language

This means that the R interpreter translates our script into small chunks of pre-compiled code. The [most common implementation of R](https://www.r-project.org/) is coded using about 50\% of C code and 25\% of FORTRAN code. 

Vectorization is a way of coding in R which tries to make the best use of the pre-compiled code. The main idea is to code in such a way that our computations are done on *vectors* instead of *numbers*. We will look at two examples.

First, let's say we want to compute the sum of all pairs of positive integers from 1 to 100. We could do this using a double loop, or in a *vectorized* way, using the ```outer``` function:


{% highlight r %}
numbers <- 1:100

compare <- microbenchmark::microbenchmark("2loops" = {
  for(i in numbers) {
    for(j in numbers) {
      i + j
    }
  }
}, "vect" = {
  outer(numbers, numbers, `+`)
})

compare
{% endhighlight %}



{% highlight text %}
## Unit: microseconds
##    expr      min       lq      mean   median       uq      max neval
##  2loops 4746.187 5239.678 6007.7809 5920.475 6747.807 8389.074   100
##    vect   96.931  117.457  157.7749  137.984  146.821 1327.381   100
{% endhighlight %}



{% highlight r %}
ggplot2::autoplot(compare)
{% endhighlight %}

![plot of chunk unnamed-chunk-4](figure/source/2015-09-11-optimisation/unnamed-chunk-4-1.png) 

There is a 40-fold difference between the two expressions!

As a second example, let's compute the mean across all rows of a matrix. We will do it using three different approaches: a for loop, using the ```apply``` function, and using the ```rowMeans``` function:


{% highlight r %}
mat <- matrix(rnorm(20*100), nrow=100, ncol=20)

compare <- microbenchmark::microbenchmark("loop" = {
  for(i in 1:nrow(mat)) {
    mean(mat[i,])
  }
}, "apply" = {
  apply(mat, 1, mean)
}, "rowMeans" = {
  rowMeans(mat)
})

compare
{% endhighlight %}



{% highlight text %}
## Unit: microseconds
##      expr      min       lq       mean   median        uq      max
##      loop 1024.616 1081.349 1176.36350 1128.674 1174.8580 2069.187
##     apply 1132.950 1219.047 1357.58404 1276.636 1354.7505 3634.334
##  rowMeans   19.386   21.097   30.55048   27.369   37.3465   80.396
##  neval
##    100
##    100
##    100
{% endhighlight %}



{% highlight r %}
ggplot2::autoplot(compare)
{% endhighlight %}

![plot of chunk unnamed-chunk-5](figure/source/2015-09-11-optimisation/unnamed-chunk-5-1.png) 

Again, we see close to a 40-fold difference. 

What vectorization does is essentially moving the for loop from R to C. Moreover, the function ```apply``` is simply a wrapper for a loop, and this is why it is usually as fast as a loop (in this example, it is actually *slower* than a loop, because we are not recording the results of the loop but apply is). 

#### R is a dynamic language

Unlike C/C++, objects in R can change quite a lot during a computation: data frames can become matrices, we can flatten lists and create atomic vectors, the class of an object can be modified multiple times, etc. This flexibility, however, usually comes with a computational cost. For example, most functions you can find in packages will perform "input checking" to control the behaviour. Understanding R coercion rules can sometimes lead to faster (and more predictable) code.

For example, the function ```sapply``` will apply a function to each element of a list and try to simplify the input to an array. However, if simplification is not possible, it will output a list **without any warning**. For this reason, it is preferable to use ```vapply```, which takes one more argument: an example of desired output.


{% highlight r %}
foo <- list(c(1,2,3,4), c(1,1,2,3,4))

sapply(foo, Filter, f = function(t) t == 1)
{% endhighlight %}



{% highlight text %}
## [[1]]
## [1] 1
## 
## [[2]]
## [1] 1 1
{% endhighlight %}



{% highlight r %}
vapply(foo, Filter, f = function(t) t == 1, 1L)
{% endhighlight %}



{% highlight text %}
## Error in vapply(foo, Filter, f = function(t) t == 1, 1L): les valeurs doivent être de type 'integer',
##  mais FUN(X[[1]]) est de type 'double'
{% endhighlight %}

Because we are telling ```vapply``` what type of output we expect, this can actually lead to faster code:


{% highlight r %}
fits <- lapply(1:100, function(t) {
  lm(rnorm(100) ~ rnorm(100, 2))
})

compare <- microbenchmark("sapply" = {
  sapply(fits, coef)
}, "vapply" = {
  vapply(fits, coef, c(1.0, 1.0))
})

compare
{% endhighlight %}



{% highlight text %}
## Unit: microseconds
##    expr     min       lq     mean   median       uq      max neval
##  sapply 858.693 898.3205 946.2955 933.3865 967.8825 1243.565   100
##  vapply 778.868 807.6615 881.2092 835.3150 868.3855 2592.614   100
{% endhighlight %}



{% highlight r %}
autoplot(compare)
{% endhighlight %}

![plot of chunk unnamed-chunk-7](figure/source/2015-09-11-optimisation/unnamed-chunk-7-1.png) 

For this example, the efficiency gain is minimal. As another example, imagine you want to make sure that when you select columns of a matrix, you still get a matrix and not an atomic vector (when subsetting, R will by default coerce a matrix with one row or one column to a vector). You can coerce the result to a matrix, or use the (little known) ```drop``` argument of the function ```[```:


{% highlight r %}
mat <- matrix(rnorm(20*100), nrow=100, ncol=20)

compare <- microbenchmark("coerce" = {
  as.matrix(mat[,1])
}, "drop" = {
  mat[,1,drop=FALSE]
})

compare
{% endhighlight %}



{% highlight text %}
## Unit: microseconds
##    expr    min     lq     mean median     uq    max neval
##  coerce 14.824 15.395 17.07142 15.395 15.965 82.106   100
##    drop  1.710  2.281  2.70282  2.851  2.851  5.132   100
{% endhighlight %}



{% highlight r %}
autoplot(compare)
{% endhighlight %}

![plot of chunk unnamed-chunk-8](figure/source/2015-09-11-optimisation/unnamed-chunk-8-1.png) 

There is an 8-fold difference between the two methods. We would still need to check if the two methods give the same result:


{% highlight r %}
identical(as.matrix(mat[,1]), mat[,1,drop=FALSE])
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}

#### Memory allocation

Somewhat related to the dynamism of R is the fact that memory can be frequently re-allocated during your computations, and this can lead to decrease in performance. For example, it is much better to pre-allocate a vector for the results of a computation than use a for loop to continually grow the results:


{% highlight r %}
compare <- microbenchmark("preallocate" = {
  results <- rep(NA, 10000)
  for(i in 1:length(results)) {
    results[i] <- runif(1)
  }
}, "growing" = {
  results = NULL
  for(i in 1:10000) {
    results <- c(results, runif(1))
  }
})

compare
{% endhighlight %}



{% highlight text %}
## Unit: milliseconds
##         expr       min        lq      mean    median       uq
##  preallocate  44.61895  45.46196  52.20404  49.03471  53.4787
##      growing 270.64611 277.21688 302.97290 296.37924 313.9291
##       max neval
##  137.9821   100
##  440.0834   100
{% endhighlight %}



{% highlight r %}
autoplot(compare)
{% endhighlight %}

![plot of chunk unnamed-chunk-10]('/'figure/source/2015-09-11-optimisation/unnamed-chunk-10-1.png) 

Of course, this is a silly example, because we could simply use the vectorised form ```runif(10000)```.

For other examples of what to do and what not to do, I recommend the book [R Inferno](http://www.burns-stat.com/pages/Tutor/R_inferno.pdf). For example, memory allocation is discussed in Circle 2 and vectorisation, in Circle 3.

### Profiling your code

### Data.table
[Data.table pass-by-reference](http://stackoverflow.com/questions/10225098/understanding-exactly-when-a-data-table-is-a-reference-to-vs-a-copy-of-another)

#### Bytecode compilation

[JIT compiler in R](http://www.r-bloggers.com/speed-up-your-r-code-using-a-just-in-time-jit-compiler/)
