---
# Example from https://joss.readthedocs.io/en/latest/submitting.html
title: 'aorsf: An R package for supervised learning using the oblique random survival forest'
tags:
  - R
  - machine learning
  - supervised learning
  - survival
  - random forest
authors:
  - name: Byron C. Jaeger
    orcid: 0000-0001-7399-2299
    affiliation: 1 # (Note: multiple affiliations must be quoted)
  - name: Sawyer Welden
    orcid: 0000-0000-0000-0000
    affiliation: 1
  - name: Kristin Lenoir
    orcid: 0000-0000-0000-0000
    affiliation: 1
  - name: Nicholas M Pajewski
    orcid: 0000-0000-0000-0000
    affiliation: 1
affiliations:
 - name: Wake Forest School of Medicine
   index: 1
citation_author: Jaeger et. al.
date: 11 May 2022
year: 2022
bibliography: paper.bib
output: rticles::joss_article
csl: apa.csl
journal: JOSS
---

# Summary

The random forest (RF) is a supervised learning method that combines predictions from a large set of decision trees [@breiman2001random]. To de-correlate decision trees in the RF, random subsets of training data are used to grow each tree and a subset of randomly selected predictor variables is considered at each non-terminal node in the trees. In @breiman2001random, RFs grown with axis based trees and oblique trees are described. Axis-based trees split non-terminal nodes using individual predictor variables whereas oblique trees use a linear combination of variables. Although oblique RFs outperform their axis-based counterparts in prediction tasks [@menze2011oblique], they are also more computationally intensive and less easily interpreted. Thus, software packages have focused mostly on axis based RFs, with few packages supporting oblique RFs, and even fewer supporting oblique random survival forests (ORSFs; @jaeger2019oblique). 

``aorsf`` is an R package optimized for fitting, interpreting, and computing predictions with ORSFs. Extensions of core features are supported by allowing users to supply their own function to identify a linear combination of inputs when growing oblique trees. The target audience includes both __practitioners__ aiming to develop an accurate and interpretable risk prediction model (e.g., see @segar2021development) and __researchers__ who want to conduct experiments comparing different techniques for identifying linear combinations of predictor variables in a controlled environment (e.g., see @katuwal2020heterogeneous). Key features of ``aorsf`` include computational efficiency compared to existing software, extensive unit and integration testing that ensure cross-platform consistency and reproducibility, and user-friendly documentation paired with an application programming interface that facilitates proper usage of the core algorithms. Interpretation is facilitated by partial dependence and negation importance, a novel method for variable importance designed for compatibility with oblique decision trees.

# Existing software 

The `obliqueRF` R package supports classification and regression using oblique random forests but does not support survival analysis. The `ranger` and `randomForestSRC` packages support survival analysis but not oblique decision trees. The ``obliqueRSF`` R package is currently the only software available to fit ORSFs. Penalized Cox regression models are used to identify linear combinations of inputs. ``aorsf`` is a re-write of the `obliqueRSF` package, with a focus on computational efficiency and flexibility. For example, ``aorsf`` allows oblique decision trees to be grown using Newton Raphson scoring (the default), penalized Cox regression, or a user-defined function.

# Newton Raphson scoring

The default routine for creating linear combinations of predictor variables in ``aorsf`` applies Newton Raphson scoring to the partial likelihood function of the Cox regression model. ``aorsf`` uses the same approach as the `survival` package to complete this estimation procedure efficiently. Full details on the steps involved have been made available by @therneau_survival_2022. Briefly, a vector of estimated regression coefficients, $\hat{\beta}$, is updated in each step of the procedure based on its first derivative, $U(\hat{\beta})$, and second derivative, $H(\hat{\beta})$: 

$$ \hat{\beta}^{k+1} =  \hat{\beta}^{k} + U(\hat{\beta} = \hat{\beta}^{k})\, H^{-1}(\hat{\beta} = \hat{\beta}^{k})$$
While it is standard practice in statistical modeling to iterate until a convergence threshold is met, the default approach in ``aorsf`` only completes one iteration. Our decision to implement this design is based on three points. First, while completing more iterations reduces bias in the regression coefficients, it generally amounts to little or no gain in prediction accuracy for the RF due to the bias-variance trade-off. Second, computing $U$ and $H$ requires computation and exponentiation of the vector $X\hat{\beta}$, where $X$ is the matrix of predictor values, but these steps can be skipped on the first iteration if an initial value of $\hat{\beta} = 0$ is assumed, allowing for a reduction in required computation. Third, using only one iteration with a starting value of 0 for $\hat{\beta}$ ensures numerical stability by avoiding exponentiation of large numbers, which can occur in later iterations depending on the scale of variables in $X$.


# Computational efficiency

The increased efficiency of ``aorsf`` versus `obliqueRSF` results from improved memory management and using Newton Raphson scoring instead of penalized Cox regression (the default approach in `obliqueRSF`). The benchmark below shows ``aorsf`` is about 3 times faster than `obliqueRSF` when both packages use penalized regression and about 400 times faster when ``aorsf`` uses Newton Raphson scoring, suggesting that the increased efficiency of ``aorsf`` is largely attributable to its use of Newton Raphson scoring.


```r
library(microbenchmark)
library(obliqueRSF)
library(aorsf)

options(microbenchmark.unit="relative")

data_bench <- pbc_orsf
data_bench$id <- NULL

microbenchmark(
 
 obliqueRSF = ORSF(data = data_bench, 
                   verbose = FALSE,
                   compute_oob_predictions = FALSE,
                   ntree = 100),
 
 aorsf_net = orsf(data_train = data_bench,
                  formula = time + status ~ .,
                  control = orsf_control_net(),
                  oobag_pred = FALSE,
                  n_tree = 100),
 
 aorsf = orsf(data_train = data_bench,
              formula = time + status ~ .,
              oobag_pred = FALSE,
              n_tree = 100),
 
 times = 100
 
)
```

```
## Unit: relative
##        expr       min       lq     mean   median       uq      max neval cld
##  obliqueRSF 241.30487 419.0895 435.3320 440.8739 453.8081 468.1365   100   c
##   aorsf_net  72.23774 124.9815 150.5739 144.0883 179.9645 246.7632   100  b 
##       aorsf   1.00000   1.0000   1.0000   1.0000   1.0000   1.0000   100 a
```


# Acknowledgements

The development of this software was supported by the Center for Biomedical Informatics at Wake Forest School of Medicine.

# References