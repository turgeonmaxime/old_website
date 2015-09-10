---
layout: post
title: "Test case: Optimising PCEV"
tags: [Optimisation, R, microbenchmark, PCEV]
permalink: optimisation-test-case
comments: false
---







```r
# Compute PCEV and its p-value 
Wilks.lambda <- function(Y, x) {
  N <- dim(Y)[1]
  p <- dim(Y)[2] 
  bar.Y <- as.vector(apply(Y, 2, mean))
  # Estimte the two variance components
  fit <- lm(Y~x)
  Y.fit <- fit$fitted
  Vr <- t((Y - Y.fit)) %*% Y
  Vg <- (t(Y.fit) %*% Y - N * bar.Y %*% t(bar.Y))
  res <- Y-Y.fit
  # We need to take the square root of Vr
  temp <- eigen(Vr,symmetric=T)
  Ur <- temp$vectors
  diagD <- temp$values 
  value <- 1/sqrt(diagD)
  root.Vr <- Ur %*% diag(value) %*% t(Ur)
  m <- root.Vr %*% Vg %*% root.Vr
  # PCEV and Wilks are eigen-components of m
  temp1 <- eigen(m,symmetric=T)
  PCEV <- root.Vr %*% temp1$vectors
  d <- temp1$values
  # Wilks is an F-test
  wilks.lambda <- ((N-p-1)/p) * d[1]
  df1 <- p
  df2 <- N-p-1
  p.value <- pf(wilks.lambda, df1, df2, lower.tail = FALSE)
  
  return(list("environment" = Vr, 
              "genetic" = Vg, 
              "PCEV"=PCEV, 
              "root.Vr"=root.Vr,
              "values"=d, 
              "p.value"=p.value))
}
```


```r
set.seed(12345)
Y <- matrix(rnorm(100*20), nrow=100)
X <- rnorm(100)

library(proftools)

Rprof(tmp <- tempfile())
res <- replicate(n = 1000, Wilks.lambda(Y, X), simplify = FALSE)
Rprof()
proftable(tmp)
```

```
##  PctTime
##  14.95  
##   6.19  
##   4.64  
##   4.64  
##   4.64  
##   4.64  
##   3.61  
##   2.58  
##   2.58  
##   2.06  
##  Call                                                                
##  eigen                                                               
##  as.vector > apply                                                   
##                                                                      
##  %*%                                                                 
##  lm > model.frame.default > .External2 > na.omit > na.omit.data.frame
##  lm > lm.fit                                                         
##  lm > model.frame.default                                            
##  as.vector > apply                                                   
##  lm > model.matrix > model.matrix.default > .External2               
##  lm > .getXlevels > paste > deparse > .deparseOpts > pmatch > c      
## 
## Parent Call: <Anonymous> > process_file > withCallingHandlers > process_group > process_group.block > call_block > block_exec > in_dir > <Anonymous> > evaluate_call > handle > try > tryCatch > tryCatchList > tryCatchOne > doTryCatch > withCallingHandlers > withVisible > eval > eval > replicate > sapply > lapply > FUN > Wilks.lambda > ...
## 
## Total Time: 3.88 seconds
## Percent of run time represented: 50.5 %
```

```r
plotProfileCallGraph(readProfileData(tmp),
                     score = "total")
```

![plot of chunk unnamed-chunk-4](/figure/source/2015-09-11-optimisation-test-case/unnamed-chunk-4-1.png) 


```r
# Compute PCEV and its p-value - Take 2
Wilks.lambda2 <- function(Y, x) {
  N <- dim(Y)[1]
  p <- dim(Y)[2] 
  bar.Y <- as.vector(apply(Y, 2, mean))
  # Estimte the two variance components
  fit <- lm.fit(cbind(rep_len(1, N), x), Y)
  Y.fit <- fit$fitted.values
  Vr <- t((Y - Y.fit)) %*% Y
  Vg <- (t(Y.fit) %*% Y - N * bar.Y %*% t(bar.Y))
  res <- Y-Y.fit
  # We need to take the square root of Vr
  temp <- eigen(Vr,symmetric=T)
  Ur <- temp$vectors
  diagD <- temp$values 
  value <- 1/sqrt(diagD)
  root.Vr <- Ur %*% diag(value) %*% t(Ur)
  m <- root.Vr %*% Vg %*% root.Vr
  # PCEV and Wilks are eigen-components of m
  temp1 <- eigen(m,symmetric=T)
  PCEV <- root.Vr %*% temp1$vectors
  d <- temp1$values
  # Wilks is an F-test
  wilks.lambda <- ((N-p-1)/p) * d[1]
  df1 <- p
  df2 <- N-p-1
  p.value <- pf(wilks.lambda, df1, df2, lower.tail = FALSE)
  
  return(list("environment" = Vr, 
              "genetic" = Vg, 
              "PCEV"=PCEV, 
              "root.Vr"=root.Vr,
              "values"=d, 
              "p.value"=p.value))
}
```


```r
Rprof(tmp <- tempfile())
res <- replicate(n = 1000, Wilks.lambda2(Y, X), simplify = FALSE)
Rprof()
proftable(tmp)
```

```
##  PctTime Call                            
##  31.18   eigen                           
##  15.05   %*%                             
##   8.60   -                               
##   8.60   lm.fit                          
##   7.53   as.vector > apply               
##   5.38   as.vector > apply               
##   3.23   as.vector > apply > mean.default
##   3.23   t                               
##   2.15   as.vector > apply > match.fun   
##   2.15   eigen > rev                     
## 
## Parent Call: <Anonymous> > process_file > withCallingHandlers > process_group > process_group.block > call_block > block_exec > in_dir > <Anonymous> > evaluate_call > handle > try > tryCatch > tryCatchList > tryCatchOne > doTryCatch > withCallingHandlers > withVisible > eval > eval > replicate > sapply > lapply > FUN > Wilks.lambda2 > ...
## 
## Total Time: 1.86 seconds
## Percent of run time represented: 87.1 %
```

```r
plotProfileCallGraph(readProfileData(tmp),
                     score = "total")
```

![plot of chunk unnamed-chunk-6](/figure/source/2015-09-11-optimisation-test-case/unnamed-chunk-6-1.png) 



```r
# Compute PCEV and its p-value - Take 3
Wilks.lambda3 <- function(Y, x) {
  N <- dim(Y)[1]
  p <- dim(Y)[2] 
  bar.Y <- colMeans(Y)
  # Estimte the two variance components
  fit <- lm.fit(cbind(rep_len(1, N), x), Y)
  Y.fit <- fit$fitted.values
  res <- Y - Y.fit
  Vr <- crossprod(res, Y)
  Vg <- crossprod(Y.fit, Y) - N * tcrossprod(bar.Y)
  # We need to take the square root of Vr
  temp <- eigen(Vr,symmetric=T)
  Ur <- temp$vectors
  diagD <- temp$values 
  value <- 1/sqrt(diagD)
  root.Vr <- tcrossprod(Ur %*% diag(value), Ur)
  m <- root.Vr %*% Vg %*% root.Vr
  # PCEV and Wilks are eigen-components of m
  temp1 <- eigen(m,symmetric=T)
  PCEV <- root.Vr %*% temp1$vectors
  d <- temp1$values
  # Wilks is an F-test
  wilks.lambda <- ((N-p-1)/p) * d[1]
  df1 <- p
  df2 <- N-p-1
  p.value <- pf(wilks.lambda, df1, df2, lower.tail = FALSE)
  
  return(list("environment" = Vr, 
              "genetic" = Vg, 
              "PCEV"=PCEV, 
              "root.Vr"=root.Vr,
              "values"=d, 
              "p.value"=p.value))
}
```


```r
Rprof(tmp <- tempfile())
res <- replicate(n = 1000, Wilks.lambda3(Y, X), simplify = FALSE)
Rprof()
proftable(tmp)
```

```
##  PctTime Call                                    
##  32.79   FUN > Wilks.lambda3 > eigen             
##  21.31   FUN > Wilks.lambda3 > crossprod         
##  16.39   FUN > Wilks.lambda3 > lm.fit            
##   4.92   FUN > Wilks.lambda3 > %*%               
##   3.28   FUN > Wilks.lambda3                     
##   3.28   FUN > Wilks.lambda3 > -                 
##   3.28   FUN > Wilks.lambda3 > lm.fit > structure
##   1.64                                           
##   1.64   FUN > Wilks.lambda3 > *                 
##   1.64   FUN > Wilks.lambda3 > /                 
## 
## Parent Call: <Anonymous> > process_file > withCallingHandlers > process_group > process_group.block > call_block > block_exec > in_dir > <Anonymous> > evaluate_call > handle > try > tryCatch > tryCatchList > tryCatchOne > doTryCatch > withCallingHandlers > withVisible > eval > eval > replicate > sapply > lapply > ...
## 
## Total Time: 1.22 seconds
## Percent of run time represented: 90.2 %
```

```r
plotProfileCallGraph(readProfileData(tmp),
                     score = "total")
```

![plot of chunk unnamed-chunk-8](/figure/source/2015-09-11-optimisation-test-case/unnamed-chunk-8-1.png) 


```r
# Compute PCEV and its p-value - Take 4
Wilks.lambda4 <- function(Y, x) {
  N <- dim(Y)[1]
  p <- dim(Y)[2] 
  bar.Y <- colMeans(Y)
  # Estimte the two variance components
  fit <- lm.fit(cbind(rep_len(1, N), x), Y)
  Y.fit <- fit$fitted.values
  res <- Y - Y.fit
  Vr <- crossprod(res, Y)
  Vg <- crossprod(Y.fit, Y) - N * tcrossprod(bar.Y)
  # We need to take the square root of Vr
  temp <- .Internal(La_rs(Vr, FALSE))
  Ur <- temp$vectors
  diagD <- temp$values 
  value <- 1/sqrt(diagD)
  root.Vr <- tcrossprod(Ur %*% diag(value), Ur)
  m <- root.Vr %*% Vg %*% root.Vr
  # PCEV and Wilks are eigen-components of m
  temp1 <- .Internal(La_rs(m, FALSE))
  PCEV <- root.Vr %*% temp1$vectors
  d <- temp1$values
  # Wilks is an F-test
  wilks.lambda <- ((N-p-1)/p) * d[1]
  df1 <- p
  df2 <- N-p-1
  p.value <- pf(wilks.lambda, df1, df2, lower.tail = FALSE)
  
  return(list("environment" = Vr, 
              "genetic" = Vg, 
              "PCEV"=PCEV, 
              "root.Vr"=root.Vr,
              "values"=d, 
              "p.value"=p.value))
}
```


```r
Rprof(tmp <- tempfile())
res <- replicate(n = 1000, Wilks.lambda4(Y, X), simplify = FALSE)
Rprof()
proftable(tmp)
```

```
##  PctTime Call                    
##  28.26   lm.fit                  
##  26.09                           
##  13.04   crossprod               
##   6.52   %*%                     
##   6.52   lm.fit > nrow > cbind   
##   4.35   colMeans > is.data.frame
##   4.35   tcrossprod              
##   2.17   -                       
##   2.17   dim                     
##   2.17   lm.fit > colnames       
## 
## Parent Call: <Anonymous> > process_file > withCallingHandlers > process_group > process_group.block > call_block > block_exec > in_dir > <Anonymous> > evaluate_call > handle > try > tryCatch > tryCatchList > tryCatchOne > doTryCatch > withCallingHandlers > withVisible > eval > eval > replicate > sapply > lapply > FUN > Wilks.lambda4 > ...
## 
## Total Time: 0.92 seconds
## Percent of run time represented: 95.7 %
```

```r
plotProfileCallGraph(readProfileData(tmp),
                     score = "total")
```

![plot of chunk unnamed-chunk-10](/figure/source/2015-09-11-optimisation-test-case/unnamed-chunk-10-1.png) 


```r
# Compute PCEV and its p-value - Final take
Wilks.lambda5 <- function(Y, x) {
  N <- dim(Y)[1]
  p <- dim(Y)[2] 
  bar.Y <- .Internal(colMeans(Y, N, p, FALSE))
  # Estimte the two variance components
  qr <- .Fortran(.F_dqrdc2, qr = cbind(rep_len(1, nrow(Y)), x), N, N, 2L, 
                 as.double(1e-07), rank = integer(1L), qraux = double(2L), 
                 pivot = as.integer(seq_len(2L)), double(2L * 2L))[c(1, 6, 7, 8)]
  Y.fit <- .Fortran(.F_dqrxb, as.double(qr$qr), N, qr$rank, as.double(qr$qraux), 
                    Y, p, xb = Y)$xb
  res <- Y - Y.fit
  Vr <- crossprod(res, Y)
  Vg <- crossprod(Y.fit, Y) - N * tcrossprod(bar.Y)
  # We need to take the square root of Vr
  temp <- .Internal(La_rs(Vr, FALSE))
  Ur <- temp$vectors
  diagD <- temp$values 
  value <- 1/sqrt(diagD)
  root.Vr <- tcrossprod(Ur %*% .Internal(diag(value, p, p)), Ur)
  m <- root.Vr %*% Vg %*% root.Vr
  # PCEV and Wilks are eigen-components of m
  temp1 <- .Internal(La_rs(m, FALSE))
  # We only need the first eigenvector
  PCEV <- root.Vr %*% temp1$vectors[,1]
  d <- temp1$values
  # Wilks is an F-test
  wilks.lambda <- ((N-p-1)/p) * d[1]
  df1 <- p
  df2 <- N-p-1
  p.value <- .Call(stats:::C_pf, wilks.lambda, df1, df2, FALSE, FALSE)
  
  return(list("environment" = Vr, 
              "genetic" = Vg, 
              "PCEV"=PCEV, 
              "root.Vr"=root.Vr,
              "values"=d, 
              "p.value"=p.value))
}
```


```r
Rprof(tmp <- tempfile())
res <- replicate(n = 1000, Wilks.lambda5(Y, X), simplify = FALSE)
Rprof()
proftable(tmp)
```

```
##  PctTime Call                                  
##  48.72                                         
##  28.21   crossprod                             
##   5.13   %*%                                   
##   5.13   .Fortran                              
##   5.13   tcrossprod                            
##   2.56   *                                     
##   2.56   ::: > get > asNamespace > getNamespace
##   2.56   as.double                             
## 
## Parent Call: <Anonymous> > process_file > withCallingHandlers > process_group > process_group.block > call_block > block_exec > in_dir > <Anonymous> > evaluate_call > handle > try > tryCatch > tryCatchList > tryCatchOne > doTryCatch > withCallingHandlers > withVisible > eval > eval > replicate > sapply > lapply > FUN > Wilks.lambda5 > ...
## 
## Total Time: 0.78 seconds
## Percent of run time represented: 100 %
```

```r
plotProfileCallGraph(readProfileData(tmp),
                     score = "total")
```

![plot of chunk unnamed-chunk-12](/figure/source/2015-09-11-optimisation-test-case/unnamed-chunk-12-1.png) 


```r
compare <- microbenchmark(Wilks.lambda(Y, X), 
                          Wilks.lambda2(Y, X), 
                          Wilks.lambda3(Y, X), 
                          Wilks.lambda4(Y, X), 
                          Wilks.lambda5(Y, X), times = 1000)
compare
```

```
## Unit: microseconds
##                 expr      min       lq      mean   median        uq
##   Wilks.lambda(Y, X) 3042.485 3287.092 3807.8829 3434.483 3781.7235
##  Wilks.lambda2(Y, X) 1239.003 1343.631 1581.3450 1410.628 1514.4000
##  Wilks.lambda3(Y, X)  828.473  888.342 1021.0741  928.254 1002.0925
##  Wilks.lambda4(Y, X)  707.025  764.327  898.3997  793.407  854.1310
##  Wilks.lambda5(Y, X)  588.996  638.032  735.6587  656.278  689.3485
##        max neval
##  24207.021  1000
##  14978.077  1000
##   5694.394  1000
##  10282.638  1000
##  22235.907  1000
```

```r
autoplot(compare)
```

![plot of chunk unnamed-chunk-13](/figure/source/2015-09-11-optimisation-test-case/unnamed-chunk-13-1.png) 
