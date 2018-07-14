---
layout: page
title: Software
---

### R packages

 - ```pcev```: PCEV is a dimension-reduction technique, similar to Principal components Analysis (PCA), which seeks to maximize the proportion of variance (in the response vector) being explained by a set of covariates. The R package implements two estimation methods: the classical approach and a block approach, proposed by Turgeon et al. (submitted), which is suitable for high-dimensional response vectors. The package also performs inference using both analytic and permutation tests. A stable version is available on [CRAN](https://cran.r-project.org/package=pcev), and the development version is on [Github](https://github.com/GreenwoodLab/pcev). For more information, have a look at the [vignette](https://cran.r-project.org/web/packages/pcev/vignettes/pcev.pdf)!

 - ```casebase```: A package whose main purpose is to fit smooth-in-time parametric hazard functions using case-base sampling (Hanley & Miettinen, 2009). This approach enables the user to fit a wide class of parametric hazard functions and, as a result, to get smooth estimates of absolute risks. The [website](http://sahirbhatnagar.com/casebase/) has several vignettes explaining how the package can be used.

 - ```multiKernel```: This package implements multivariate prediction using kernel-machine regression. It is still very much in development on [Github](https://github.com/turgeonmaxime/multiKernel).
 
 - ```rootWishart```: Functions for hypothesis testing in single and double Wishart settings, based on Roy's largest root. This test statistic is especially useful in multivariate analysis. The computations are based on results by [Chiani (2014)](http://dx.doi.org/10.1016/j.jmva.2014.04.002) and [Chiani (2016)](http://dx.doi.org/10.1016/j.jmva.2015.10.007). They use the fact that the CDF is related to the Pfaffian of a matrix that can be computed in a finite number of iterations. This package takes advantage of the Boost and Eigen C++ libraries to perform multi-precision linear algebra. A stable version is available on [CRAN](https://cran.r-project.org/package=rootWishart), and the development version is on [Github](https://github.com/turgeonmaxime/rootWishart).
