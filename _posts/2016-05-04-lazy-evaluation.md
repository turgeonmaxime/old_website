---
title: "R and lazy evaluation"
author: "Maxime Turgeon"
date: "May 4th, 2016"
output: html_document
---

A few days ago, Luc Villandre (a colleague of mine at [McGill University](http://www.mcgill.ca/epi-biostat-occh/)) asked me if I could explain the peculiar behaviour of the function ```missing``` in ```R```. Trying to understand what happens forced me to learn a little bit more about the inner workings of ```R```, especially when it comes to lazy evaluation. I thought I would share what I learned on my blog.

<!--more-->

The function ```missing``` can be very useful when we want to define a default behaviour of a function. I'll come back later to what it actually does, but first note that it can propagate from one function call to the next:


{% highlight r %}
bar <- function(x) missing(x)
foo <- function(x) bar(x)

foo()
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}

There can even be more than two function calls:


{% highlight r %}
norf <- function(x) foo(x)
norf()
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}

The problem comes when we want to skip over some of the functions in the stack. For example, let's consider the function ```foo```, which defines the function ```snorf``` in its local environment. Using the helper function ```bar```, ```snorf``` wants to check if an argument was passed to ```foo```:


{% highlight r %}
bar <- function(x) missing(x)
foo <- function(x) {
    snorf <- function(y) bar(x)
    snorf()
}
foo()
{% endhighlight %}



{% highlight text %}
## [1] FALSE
{% endhighlight %}

Perhaps surprinsly, the value ```FALSE``` is returned. This should imply that ```foo``` wasn't missing an argument. So what's going on?

### Functions and promises

To understand what is going on, we need to understand how R deals with functions. 



{% highlight r %}
bar <- function(x) pryr::promise_info(x)
foo <- function(x) bar(x)
norf <- function(x) {
    print(foo(x))
    print(bar(x))
}

norf()
{% endhighlight %}



{% highlight text %}
## Error in loadNamespace(name): there is no package called 'pryr'
{% endhighlight %}


{% highlight r %}
# bar <- function(x) missing(x)
foo <- function(x) {
    snorf <- function(y) bar(x)
    print(environment(snorf))
    print(snorf())
}
foo()
{% endhighlight %}



{% highlight text %}
## <environment: 0x1a53630>
{% endhighlight %}



{% highlight text %}
## Error in loadNamespace(name): there is no package called 'pryr'
{% endhighlight %}
