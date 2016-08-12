Introduction
============

A Brief Welcome
---------------

If you're reading this, you probably have some interest in sequencing T cell receptors (TCR), especially paired alpha-beta (or gamma-delta) sequences of individual TCRs. And you probably have a sense that this is *hard* to do in a high-throughput, scaleable, and efficient way. The immunology community has recently been innovating new and interesting solutions that range from pushing the limits of experimental techniques to using complicated statistical or computationals models (which includes our own approach). The a dividing fragmentation of techniques that I perceive just confirms the reality that we don't have a good standard for getting paired TCR sequences. My hope is that this vignette will be clear and transparent in showing how we thought to solve this problem, not as be-all panacea for our sequencing woes but as a contribution towards the greater goal of moving the field closer to an industry standard. The main purpose of this vignette is to demonstrate how to obtain the resuls of our paper[1] and hopefully give you the tools to do simulations of your own.

Immunology background
---------------------

Under construction.

T cell receptors (TCR) are made up of two chains, an alpha chain and a beta chain. Identifying TCRs of T cell clones at a minimum requires the identification of the pair of CDR3 sequences of the alpha and beta chains.

Our approach
------------

Under construction.

The approach in `alphabetr` at its core is mathematical and statistical with a heuristic favlor.

<!-- # Quick startup guide -->
<!-- This section will get you up and running the code really quickly with minimal explanation. The next section provides more details about each function.  -->
<!-- We begin by create a T cell population with our specified number of unique beta chains, proportion of dual TCR-alpha clones, and degrees of sharing of both component chains. -->
Overview of the code
====================

We'll start by looking at an overview of how to run the `alphabetr` functions on the sequencing data and then look at the parameters used to simulate the sequencing data.

Processing the sequencing data
------------------------------

Our goals are to:

1.  Determine alpha-beta candidate pairs
2.  Discriminate between dual-alpha TCR clones and beta-sharing clones
3.  Estimate clonal frequencies

**Determining candidate pairs.** We input the sequencing data into `bagpipe()`, which determines candidate alpha-beta pairs. The output of `bagpipe()` determines pairs consisting of only one alpha and one beta.

**Discriminating dual-alpha and beta-sharing clones.** We then utilize `freq_estimate()`and then use those estimates in the `dual_top()`/`dual_tail()` functions. Although it seems backwards to estimate clonal frequencies before determining dual-alpha clones, `dual_top()`/`dual_tail()` need these frequencies to determine the dual clones[2] `dual_top()`/`dual_tail()` will then tell us which candidates that share the same beta are actually one clone express both alpha chains.

**Estimating clonal frequencies.** The final list of candidate clones, which includes both single-alpha and dual-alpha clones, is then passed into `freq_estimate()` in order to get a final list of frequen estimates and 95% frequency confidence intervals. The package includes some helper functions to amend the original list of candidate pairs with the newly determined dual-alpha clones (used below in the rest of the vignette).

<!-- ```{r, out.width = 650, fig.retina = NULL, fig.cap = "Figure 1", echo = FALSE} -->
<!-- knitr::include_graphics("~/Dropbox/_Research/TCR Diversity/alphabetr/alphabetr/vignettes/overview.png") -->
<!-- ``` -->
Simulating sequencing data
--------------------------

Simulating the experimental data involves choosing parameters for two domains:

1.  The clonal structure of the T cell population of interest
2.  The sequencing experiment

Both are described in Figure .

**Clonal structure.** We create a T cell population `create_clones()` is fully described below, so all I want to do is illustrate what *degree of sharing* means. Suppose the clones in Figure represent our population of interest. We have four unique alpha chains (a1, a2, a3, a4), one of which is shared by two clones (a2 is shared by a2b1, a2b3). In this population, 25% of the alpha chains are shared by two clones, and 75% of the alpha chains are not shared.

**Simulating the sequencing experiment.** Just a remark that there's a lot of parameters to simulate on the experimental side of things.

Running the code
================

This section of the vignette will show you how to run simulations from start to finish in order to obtain the type of results as shown in the paper. We will first create T cell populations with user-specified attributes, use functions in `alphabetr` to figure our candidates pairs, discriminate between dual-alpha and beta-sharing clones, and then determine clonal frequencies.

Installing and loading alphabetr
--------------------------------

The package is currently not on CRAN and must be installed from github. The easiet way to do this is through the `devtools` package. If you don't have `devtools` installed, run the following

``` r
install.packages("devtools")
```

With `devtools` installed, install alphabetr by running

``` r
devtools::install_github("edwardslee/alphabetr")
```

And then load the package by running

``` r
library(alphabetr)
```

Creating the clonal structure of the T cell population
------------------------------------------------------

`create_clones()` will create a T cell population with a structure specified by you. The attributes that can be changed are

1.  The number of unique beta chains
2.  The proportion of clones with dual alpha chains
3.  The degree of alpha sharing (the proportion of clones that share 1-6 alpha chains)
4.  The degree of beta sharing (the proportion of clones that share 1-5 beta chains)

In order to create a population containing \* 1000 unique beta chains \* 30% of clones with dual TCRs \* beta-sharing: 75% beta shared by one clone, 20% by two clones, 5% by three clones \* alpha-sharing: 80% alpha chains shared by one clone, 15% by two clones, and 5% by three clones

We run the following

``` r
set.seed(290)   # to make the results reproducible
TCR <- create_clones(numb_beta = 1000, dual = .3,
                     alpha_sharing = c(0.80, 0.15, 0.05),
                     beta_sharing  = c(0.75, 0.20, 0.05))
```

The function arguments are fairly straightforward: `numb_beta` is the number of unique beta chains in the population, `dual` is the proportion of clones with dual alpha chains, and `alpha_sharing` and `beta_sharing` are the vectors that represent the degree of sharing. Position `i` of the sharing vectors represent the proportion of the chains shared by `i` clones.

The output is a list containing different useful versions of the clonal structure

``` r
# Clones ordered by beta index
ordered_clones <- TCR$ordered

# Clones randomly ordered in order to assign random clonal frequencies later 
random_clones <- TCR$random

# Clones that express two different alpha chains
dual_clones <- TCR$dual
```

Each of these are 3 column matrices, where each row represents a clone, the first column is the beta chain index, and the second and third columns are the alpha indices. If the clone expresses only one alpha chain, then the 2nd and 3rd columns will be equal. For example,

``` r
# single TCR-alpha clones
random_clones[4:6, ]
#>      beta alpha1 alpha2
#> [1,]  614    159    159
#> [2,]  145    609    609
#> [3,]  867    910    910

# dual TCR-alpha clone
random_clones[13:14, ]
#>      beta alpha1 alpha2
#> [1,]  945    412    534
#> [2,]   89    989   1018
```

For the paper, the clonal structures were created with the following code

``` r
# Sharing vectors; with this, 1692 beta chains results in 2100 unique clones
share_alph <- c(.816, .085, .021, .007, .033, .005, .033)
share_beta <- c(.859, .076, .037, .019, .009)

# Creating a population of 2100 clones with specified sharing and 30% dual clones
set.seed(258)   # reproducibility for the vignette
TCR_pairings <- create_clones(numb_beta = 1692, dual = 0.3, 
                             alpha_sharing = share_alph, beta_sharing = share_beta)
TCR_clones <- TCR_pairings$random
```

We'll use `TCR_clones`, which contains the clones in random order, in the next section to simulate sequencing data.

Simulating a TCR sequencing experiment
--------------------------------------

Now that we've created the clones with their alpha and beta sequences (with each unique chain represented by an integer/index), we can simulate sequencing data. We do this by using the function `create_data()`. `create_data()` will sample the clones from a skewed distribution into the wells of 96-well plates, which is what would happen in a sequencing experiment by staining T cells with tetramer and using a sorter to sample them into plates.

A number of experimental parameters can be changed to simulate different experimental settings. These parameters and their corresponding arguments in `create_data` are

1.  `plates`: the number of 96-well plates used in the experiment
2.  `error`: the error "drop-out" rate, i.e. the rate that any sampled chain can fail to be sequenced (and thus not appear in our final data set)
3.  `skewed`: the number of clones that represent the "top proportion" of the population in frequency; this "top proportion"" is specified in the `pct_top` argument
4.  `prop_top`: the proportion of the population by frequency that is represented by the top clones (of which there are `skewed` number of them)
5.  `dist`: to developed in the future, but this option controls the shape of the frequency distribution; always set to `"linear"` for now
6.  `numb_cells`: sampling strategy that keeps track of the sample sizes of the wells; more details below

`create_data` takes a matrix of clones as an input, such as `TCR_clones`, and the order the clones appear in this matrix determines their relative frequencies. The first `skewed` number of rows of the input matrix is used as the clones representing the top proportion of the population, and `create_data` internally distributes them in a descending, linear way across the top. The other clones of the tail then have the same frequency, all adding up to the tail proportion of the population.

To illustrate this better, if `skewed = 25` and `prop_top = 0.6`, then the clones represented by the first 25 rows of `TCR_clones` make up the top 60% of the population, and the other 2075 clones make the other 40% of the population.

``` r
# The 25 clones that make up the top 60% of the population
TCR_clones[1:25, ]
#>       beta alpha1 alpha2
#>  [1,] 1026    690    983
#>  [2,]   33   1310   1260
#>  [3,]   49   1227   1224
#>  [4,] 1610   1310   1310
#>  [5,]  127     85   1116
#>  [6,] 1137    663    663
#>  [7,]  163      6      6
#>  [8,]  869    533    533
#>  [9,] 1027    295    295
#> [10,] 1465     37     37
#> [11,]  786   1311   1311
#> [12,]  862    998   1102
#> [13,]  816   1266   1266
#> [14,] 1406   1310   1310
#> [15,]   13    143    143
#> [16,] 1312   1347   1347
#> [17,] 1175    111    111
#> [18,] 1526    348    159
#> [19,]  784     65     65
#> [20,]  486   1310    806
#> [21,]  498    683    966
#> [22,] 1214    281    281
#> [23,] 1174    891    891
#> [24,]  675    343    343
#> [25,] 1534    418    418
```

The `numb_cells` argument is a 2 column matrix, where for each row, the 1st column represents the sample size in the well, and the 2nd column is the number of wells with that sample size. You must ensure that the number of wells specified by `numb_cells` is equal to `96 * number_plates`.

``` r
# 5 plates (= 480 wells), every well has a sample size of 50 cells
numb_cells <- matrix(c(50, 480), ncol = 2)

# 1 plate (= 96 wells), 48 wells with 100 cells/well, 48 wells with 200 cells/well
numb_cells <- matrix(c(100, 200, 48, 48), ncol = 2)
```

Here are the parameters used in the paper with the "mixed" sampling strategy:

``` r
# Different experimental parameters
number_plates <- 5       # five 96-well plates
err <- 0.15              # error drop out rate of 15%
number_skewed <- 25      # 25 clones representing the top proportion of population
pct_top <- 0.5           # top of population represents 50% of population
dis_behavior <- "linear" # always set to linear

# Mixed sampling strategy: 128 wells of 20 cells/well, 64 wells of 50 cells/well,
# 96 wells of 100 cells/well, 200 cells/well, 300 cells/well each
numb_cells <- matrix(c(20,  50, 100, 200, 300,
                       128, 64,  96,  96,  96), ncol = 2)

# Creating the data sets
data_tcr <- create_data(TCR = TCR_clones, plates = number_plates, error = err,
                        skewed = number_skewed, prop_top = pct_top,
                        dist = dis_behavior, numb_cells = numb_cells)

# Saving the data for alpha chains and data for beta chains
data_alph <- data_tcr$alpha
data_beta <- data_tcr$beta
```

The output of `create_data` is a list, and the `alpha` and `beta` components contain the data for alpha chains and beta chains respectively. Each is a matrix where the rows represent the columns represent the chain indices and the row represents the well. If a well contains a chain, then row representing that well has a 1 in the chain's column and 0 if the well does not contain that chain (e.g. if well \#25 contains beta 20, then `data_beta[25, 20]` is 1).

Finding candidate pairs
-----------------------

Now that we've created the data set, we can begin to apply the algorithms to determine TCRs. The default parameters should work sufficiently well for most data sets

``` r
# Normally you would want to set rep = 100
pairs <- bagpipe(alpha = data_alph, beta = data_beta, rep = 3)
```

The output of `bagpipe` is a matrix with the candidate pairs and the proportion of replicates that each candidate pair appears in.

``` r
head(pairs)
#>      beta alpha1 alpha2 prop_replicates
#> [1,]    1   1282   1282       0.6666667
#> [2,]    2    321    321       1.0000000
#> [3,]    4    874    874       1.0000000
#> [4,]    5    493    493       1.0000000
#> [5,]    6    400    400       0.6666667
#> [6,]    7   1220   1220       1.0000000
```

You can clearly see that all of the candidate pairs are single TCR clones at this point. We will attempt to determine dual TCR clones later on in the vignette.

Before moving on, you may choose to the filter the candidate pairs with a threshold of the proportion of replicates that a candidate pair must appear in. Increasing the threshold will significantly decrease the false pairing rate (i.e. the rate at which incorrect pairs are identified) while sacrificing depth of the tail and depth of dual TCR-alpha clone identification.

Our own simulations seems to indicate that a threshold of 0.3 gives a good balance of false pairing rate, tail depth, and dual depth:

``` r
# remove candidate pairs that don't appear in more than 30% of replicates
pairs <- pairs[pairs[, "prop_replicates"] > .3, ]
head(pairs)
#>      beta alpha1 alpha2 prop_replicates
#> [1,]    1   1282   1282       0.6666667
#> [2,]    2    321    321       1.0000000
#> [3,]    4    874    874       1.0000000
#> [4,]    5    493    493       1.0000000
#> [5,]    6    400    400       0.6666667
#> [6,]    7   1220   1220       1.0000000
```

Performing frequency estimation on these pairs
----------------------------------------------

With the candidate alpha/beta pairs, we perform an initial frequency estimation as though each candidate pair represents a distinct clone. This is an immediately incorrect assumption because at this point: dual-alpha clones are represented as two distinct clones that share the same beta chain (e.g. if *β*<sub>1</sub>, *α*<sub>1</sub>, *α*<sub>2</sub> is a dual clone, then we will have two candidate pairs *β*<sub>1</sub>, *α*<sub>1</sub> and *β*<sub>1</sub>, *α*<sub>2</sub> at this point). We use the estimated frequencies in order to discriminate between beta-sharing clones and dual-alpha clones.

In order to perform frequency estimation, we use `freq_estimate()`. The arguments `alpha` and `beta` require the sequencing data sets about alpha and beta chains respectively; `pair` takes the output of `bagpipe`; `error` is the experimental error dropout rate; and `cells` is the sample sizes of the wells and the number of wells with those sample sizes (in the same format as `numb_cells` in `bagpipe()`).

Using the data and output of `bagipe()` from above, we perform frequency estimation:

``` r
freq_init <- freq_estimate(alpha = data_alph, beta = data_beta, pair = pairs,
                           error = err, cells = numb_cells)
head(freq_init)
#>   beta alpha1 alpha2          MLE        CI_up       CI_low    CI_length
#> 1    1   1282   1282 0.0001400168 0.0003024176 6.532634e-05 0.0002370913
#> 2    2    321    321 0.0002265519 0.0003901819 1.100830e-04 0.0002800988
#> 3    4    874    874 0.0002265519 0.0004218406 1.252950e-04 0.0002965457
#> 4    5    493    493 0.0003079321 0.0004783278 1.570089e-04 0.0003213189
#> 5    6    400    400 0.0002265519 0.0003908085 1.102715e-04 0.0002805370
#> 6    7   1220   1220 0.0004184966 0.0006228219 2.432787e-04 0.0003795432
#>   pct_replicates
#> 1      0.6666667
#> 2      1.0000000
#> 3      1.0000000
#> 4      1.0000000
#> 5      0.6666667
#> 6      1.0000000
```

You can see that the output shows you the alpha and beta indices of the clone, the frequency point estimate (under `MLE`), the upper and lower limits of the 95% confidence interval (`CI_up` and `CI_low` respectively), the length of the confidence interval, and the number of replicates each clone was found in during `bagpipe()`.

Discriminating beta-sharing and dual TCR-alpha clones
-----------------------------------------------------

In order to disciminate between beta-sharing and dual-alpha clones, we needed the frequency estimation of the candidate pairs given by `bagpipe()`, and the estimated frequencies are (the reason for this described in our pp)

``` r
# determining duals in the top; note the use of the error rate
common_dual <- dual_top(alpha = data_alph, beta = data_beta,
                        pair = freq_init, error = err, cells = numb_cells)

# determining duals in the tail; note that this does NOT use the error rate
tail_dual <- dual_tail(alpha = data_alph, beta = data_beta,
                                freq_results = freq_init, cells = numb_cells)
```

The arguments of these functions are the same arguments used before.

We combine the output of both functions to obtain a data frame with all of the dual clones:

``` r
clones_dual <- rbind(common_dual, tail_dual)
head(clones_dual)
#>   beta alpha1 alpha2
#> 1   33   1310   1260
#> 2  127   1116     85
#> 3  862   1102    998
#> 4 1026    690    983
#> 5   18    870    795
#> 6   25    734    832
```

Final frequency estimation
--------------------------

Now that we identified dual TCR-alpha clones, all that's left is to estimate their frequencies and replace the corresponding beta-sharing candidate pairs with the dual TCR-alpha clone:

``` r
# Find the frequencies of the newly identified dual clones
freq_dual <- freq_estimate(alpha = data_alph, beta = data_beta,
                           pair = clones_dual, error = err, cells = numb_cells)

# Remove the candidate beta-sharing clones and replace with the dual clones
tcrpairs <- combine_freq_results(freq_init, freq_dual)
```

`combine_freq_results()` is a helper function that will combine the initial frequency results and the freshly calculated dual frequency results. It will find the two rows containing the candidate pairs that derived from the dual clone and replace it with the results of the dual clone. The first argument is the initial frequency results, and the second argument is the dual frequency results.

The final results of processing our TCR sequencing data is contained in our data frame `tcrpairs`

``` r
head(tcrpairs)
#>   beta alpha1 alpha2        MLE      CI_up     CI_low  CI_length
#> 1   33   1310   1260 0.04913229 0.05755395 0.04182061 0.01573334
#> 2   49   1224   1224 0.04178970 0.04883334 0.03563634 0.01319701
#> 3 1026    690    983 0.03978803 0.04653287 0.03394254 0.01259033
#> 4 1137   1310   1310 0.03934406 0.04590850 0.03363445 0.01227405
#> 5 1137    663    663 0.03889887 0.04532260 0.03326572 0.01205688
#> 6 1610   1310   1310 0.03870891 0.04517764 0.03305171 0.01212593
#>   pct_replicates
#> 1     -1.0000000
#> 2      1.0000000
#> 3     -1.0000000
#> 4      0.6666667
#> 5      1.0000000
#> 6      1.0000000
```

Note that if a clone is dual, then the `pct_replicates` column is set to -1.

Evaluation of results
---------------------

To be finished.

[1] Paper was submitted recently

[2] see the paper to understand the mathematics that's going on.