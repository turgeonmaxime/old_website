---
title: "Correlations"
author: "Maxime Turgeon"
date: "May 2, 2016"
output: html_document
---


{% highlight r %}
n=1000; p=500

rho1 <- 0.1
rho2 <- 0.2

Sigma1 <- matrix(rho1, nrow = p, ncol = p)
Sigma2 <- matrix(rho2, nrow = p, ncol = p)
diag(Sigma1) <- diag(Sigma2) <- 1

library(MASS)
X1 <- mvrnorm(n, mu=rep_len(0, p), Sigma = Sigma1)
X2 <- mvrnorm(n, mu=rep_len(0, p), Sigma = Sigma2)
matCor1 <- cor(X1)
matCor2 <- cor(X2)

library(data.table)
DT <- data.table(cor=matCor1[!diag(p)], pop=rho1)
DT <- rbind(DT, data.frame(cor=matCor2[!diag(p)], pop=rho2))
DT$pop <- factor(DT$pop)

library(ggplot2)
ggplot(DT, aes(cor, color = pop, fill = pop)) + geom_density(alpha=0.1)
{% endhighlight %}

![plot of chunk unnamed-chunk-1](/figure/source/2016-05-24-correlation/unnamed-chunk-1-1.png)



{% highlight r %}
DT[,list("2.5%"=quantile(cor, probs = c(0.025)),
         "97.5%"=quantile(cor, probs = c(0.975))), by=pop]
{% endhighlight %}



{% highlight text %}
##    pop       2.5%     97.5%
## 1: 0.1 0.03828486 0.1598763
## 2: 0.2 0.13858484 0.2546273
{% endhighlight %}
