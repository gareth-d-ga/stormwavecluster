
# **Modelling the distributions of storm event statistics**
--------------------------------------------------------------------------

*Gareth Davies, Geoscience Australia 2017*

# Introduction
------------------

This document follows on from
[statistical_model_storm_timings.md](statistical_model_storm_timings.md)
in describing our statistical analysis of storm waves at Old Bar. 

It illustrates the process of fitting probability distributions to the storm event summary statistics,
which are conditional on the time of year and ENSO.

It is essential that the code
[statistical_model_storm_timings.md](statistical_model_storm_timings.md) has
alread been run, and produced an Rdata file
*'Rimages/session_storm_timings_FALSE_0.Rdata'*. **To make sure, the code below
throws an error if the latter file does not exist.**

```r
# Check that the pre-requisites exist
if(!file.exists('Rimages/session_storm_timings_FALSE_0.Rdata')){
    stop('It appears you have not yet run the code in statistical_model_storm_timings.md. It must be run before continuing')
}
```
You might wonder why the filename ends in `_FALSE_0`. Here, `FALSE`
describes where or not we perturbed the storm summary statistics before running
the fitting code. In the `FALSE` case we didn't - we're using the raw data.
However, it can be desirable to re-run the fitting code on perturbed data, to
check the impact of data discretization on the model fit. One would usually do
many such runs, and so we include a number (`_0` in this case) to distinguish
them. So for example, if the filename ended with `_TRUE_543`, you could assume
it includes a run with the perturbed data (with 543 being a unique ID, that is
otherwise meaningless).

Supposing the above did not generate any errors, and you have R installed,
along with all the packages required to run this code, and a copy of the
*stormwavecluster* git repository, then you should be able to re-run the
analysis here by simply copy-pasting the code. Alternatively, it can be run
with the `knit` command in the *knitr* package: 

```r
library(knitr)
knit('statistical_model_univariate_distributions.Rmd')
```

The basic approach followed here is to:
* **Step 1: Load the previous session**
* **Step 2: Exploratory analysis of seasonal non-stationarity in event statistics**
* **Step 3: Model the distribution of each storm summary statistic, dependent on season (and mean annual SOI for wave direction)**

Later we will model the remaining joint dependence between these variables, and
simulate synthetic storm sequences. 

# **Step 1: Load the previous session**
Here we simply re-load the session from the previous stage of the modelling.

```r
    previous_R_session_file = 'Rimages/session_storm_timings_FALSE_0.Rdata'
    load(previous_R_session_file)
```

# **Step 2: Exploratory analysis of seasonal non-stationarity in event statistics**
----------------------------------------------------------------------

**Here we plot the distribution of each storm statistic by month.** This
highlights the seasonal non-stationarity.

```r
# Get month as 1, 2, ... 12
month_num = as.numeric(format(event_statistics$time, '%m'))
par(mfrow=c(3,2))
for(i in 1:5){
    boxplot(event_statistics[,i] ~ month_num, xlab='Month', 
        ylab=names(event_statistics)[i], names=month.abb,
        col='grey')
    title(main = names(event_statistics)[i], cex.main=2)
}

rm(month_num)
```

![plot of chunk monthplot](figure/monthplot-1.png)

To model the seasonal non-stationarity mentioned above, we define a seasonal
variable periodic in time, of the form `cos(2*pi*(t - offset))` where the time
`t` is in years. The `offset` is a phase variable which can be optimised for
each storm summary statistic separately, to give the 'best' cosine seasonal pattern.

**Below we compute the offset for each storm summary statistic, and also assess
it's statistical significance using a permutation test.** The figure shows the
rank correlation between each variable and a seasonal variable, for each value
of `offset` in [-0.5, 0.5] (which represents all possible values). The shaded
region represents a 95% interval for the best correlation expected of 'random'
data (i.e. a random sample of the original data with an optimized offset).
Correlations outside the shaded interval are unlikely to occur at random, and
are intepreted as reflecting true seasonal non-stationarity. 

```r
# Store some useful statistics
stat_store = data.frame(var = rep(NA, 5), phi=rep(NA,5), cor = rep(NA, 5), 
    p = rep(NA, 5), cor_05=rep(NA, 5))
stat_store$var = names(event_statistics)[1:5]

# Test these values of the 'offset' parameter
phi_vals = seq(-0.5, 0.5, by=0.01)
par(mfrow=c(3,2))
for(i in 1:5){

    # Compute spearman correlation for all values of phi, for variable i
    corrs = phi_vals*0
    for(j in 1:length(phi_vals)){
        corrs[j] =  cor(event_statistics[,i], 
            cos(2*pi*(event_statistics$startyear - phi_vals[j])),
            method='s', use='pairwise.complete.obs')
    }

    plot(phi_vals, corrs, xlab='Offset', ylab='Spearman Correlation', 
        main=names(event_statistics)[i], cex.main=2,
        cex.lab=1.5)
    grid()
    abline(v=0, col='orange')

    # Save the 'best' result
    stat_store$phi[i] = phi_vals[which.min(corrs)]
    stat_store$cor[i] = min(corrs)

    # Function to compute the 'best' correlation of season with
    # permuted data, which by definition has no significant correlation with
    # the season. We can use this to assess the statistical significance of the
    # observed correlation between each variable and the season.
    cor_phi_function<-function(i0=i){
        # Resample the data
        d0 = sample(event_statistics[,i0], size=length(event_statistics[,i0]), 
            replace=TRUE)
        # Correlation function
        g<-function(phi){ 
            cor(d0, cos(2*pi*(event_statistics$startyear - phi)), 
                method='s', use='pairwise.complete.obs')
        }
        # Find best 'phi'
        best_phi = optimize(g, c(-0.5, 0.5), tol=1.0e-06)$minimum

        return(g(best_phi))
    }
   
    # Let's get statistical significance 
    cor_boot = replicate(5000, cor_phi_function())

    # Because our optimizer minimises, the 'strongest' correlations
    # it finds are negative. Of course if 0.5 is added to phi this is equivalent
    # to a positive correlation. 
    
    qcb = quantile(cor_boot, 0.05, type=6)
    stat_store$cor_05[i] = qcb
    stat_store$p[i] = mean(cor_boot < min(corrs))

    polygon(rbind( c(-1, -qcb), c(-1, qcb), c(1, qcb), c(1, -qcb)),
        col='brown', density=10)
}

write.table(stat_store, file='seasonal_correlation_statistics.csv', sep="  &  ",
    quote=FALSE, row.names=FALSE)

rm(phi_vals, corrs)
```

![plot of chunk seasonphase1](figure/seasonphase1-1.png)

Recall also that relationships between mean annual SOI and storm wave direction
were established earlier (
[../preprocessing/extract_storm_events.md](../preprocessing/extract_storm_events.md),
[statistical_model_storm_timings.md](statistical_model_storm_timings.md) ). We
also found relationships between mean annual SOI and the rate of storms, and
MSL, which were treated in those sections (using the non-homogeneous poisson
process model, and the STL decomposition, respectively).

Below we will make each storm summary statistic dependent on the seasonal
variable. For wave direction, the mean annual SOI value will also be treated.

# **Step 3: Model the distribution of each storm summary statistic, dependent on season (and mean annual SOI for wave direction)**

**Below we fit an extreme value mixture model to Hsig, using maximum
likelihood.** The model has a GPD upper tail, and a gamma lower tail.

```r
# Get the exmix_fit routines in their own environment
evmix_fit = new.env()
source('../../R/evmix_fit/evmix_fit.R', local=evmix_fit, chdir=TRUE)

# Fit it
hsig_mixture_fit = evmix_fit$fit_gpd_mixture(data=event_statistics$hsig, 
    data_offset=as.numeric(hsig_threshold), phiu=TRUE, bulk='gamma')
```

```
## [1] "  evmix fit NLLH: " "530.237080056746"  
## [1] "  fit_optim NLLH: " "530.237080030477"  
## [1] "  Bulk par estimate0: " "0.842400448660163"     
## [3] "1.0204368965617"        "1.27267963923683"      
## [5] "-0.219878654410625"    
## [1] "           estimate1: " "0.842402977361388"     
## [3] "1.02042857261524"       "1.27268869445894"      
## [5] "-0.219876719484767"    
## [1] "  Difference: "        "-2.52870122496862e-06" "8.32394645855494e-06" 
## [4] "-9.05522210814524e-06" "-1.93492585820465e-06"
## [1] "PASS: checked qfun and pfun are inverse functions"
```

```r
DU$qqplot3(event_statistics$hsig, hsig_mixture_fit$qfun(runif(100000)), 
    main='Hsig QQ-plot')
abline(0, 1, col='red'); grid()
```

![plot of chunk hsig_fitA](figure/hsig_fitA-1.png)

The above code leads to print-outs of the maximum likelihood parameter fits
achieved by different methods, and the differences between them (which are
only a few parts per million in this case). Because fitting extreme value
mixture models can be challenging, the code tries many different fits. 

During the fitting process, we also compute quantile and inverse quantile functions
for the fitted distribution. The code checks numerically that these really are the inverse
of each other, and will print information about whether this was found to be true (*if not,
there is a problem!*)

The quantile-quantile plot of the observed and fitted Hsig should fall close to
a straight line if the fit worked. Poor fits are suggested by strong deviations
from the 1:1 line.

Given that the above fit looks OK, **below we use Monte-Carlo-Markov-Chain
(MCMC) techniques to compute the Bayesian posterior distribution of the 4 model
parameters**. A few points about this process:
* The prior probability is uniform for each variable, with the
gamma distribution parameters ranging from zero and 100 million, the GPD
threshold parameter ranging from zero to the 50th highest data point, and the
GPD shape parameter ranging over [-1000 to 1000]. Note that for some other datasets, it
might be necessary to constrain the GPD shape parameter prior more strongly
than we do below. Here, we are trying to make our priors 'non-informative' by
allowing them to have very large ranges -- except in the case of the GPD threshold,
where we enforce an upper bound which leads to at least 50 points being used for
the upper tail fit.
* The routines create update the object `hsig_mixture_fit`, so it contains
multiple chains, i.e. 'random walks' through the posterior parameter
distribution.
* Here we run 6 separate chains, with randomly chosen starting parameters, to make it
easier to detect non-convergence (i.e. to reduce the chance that a single chain gets 'stuck'
in part of the posterior distribution). The parameter `mcmc_start_perturbation` defines the
scale for that perturbation.
* It is possible that the randomly chosen start parameters are theoretically impossoble. In this
case, the code will report that it had `Bad random start parameters`, and will generate new ones.
* We use a burn-in of 1000 (i.e. the first 1000 entries in the chain are discarded). This
can assist with convergence.
* We make a simple diagnostic plot to check the MCMC convergence.
* The code runs in parallel, using 6 cores below. The parallel framework will
nly work correctly on a shared memory linux machine.

```r
#' MCMC computations for later uncertainty characterisation
#' (hard to get convergence when phiu = FALSE)

# Prevent the threshold parameter from exceeding the highest 50th data point
hsig_u_limit = sort(event_statistics$hsig, decreasing=TRUE)[50] - hsig_threshold

# Compute the MCMC chains in parallel
hsig_mixture_fit = evmix_fit$mcmc_gpd_mixture(
    fit_env=hsig_mixture_fit, 
    par_lower_limits=c(0, 0, 0, -1000.), 
    par_upper_limits=c(1e+08, 1.0e+08, hsig_u_limit, 1000),
    mcmc_start_perturbation=c(0.4, 0.4, 2., 0.2), 
    mcmc_length=mcmc_chain_length,
    mcmc_thin=mcmc_chain_thin,
    mcmc_burnin=1000,
    mcmc_nchains=6,
    mcmc_tune=c(1,1,1,1)*2,
    mc_cores=6,
    annual_event_rate=mean(events_per_year_truncated))

# Graphical convergence check of one of the chains. 
plot(hsig_mixture_fit$mcmc_chains[[1]])
```

![plot of chunk hsigmixtureFitBayes](figure/hsigmixtureFitBayes-1.png)

**Below, we investigate the parameter estimates for each chain.** If all the
changes have converged, the quantiles of each parameter estimate should be
essentially the same (although if the underlying posterior distribution is
unbounded, then of course the min/max will not converge, although non-extreme
quantiles will). We also look at the 1/100 year event Hsig implied by each
chain, and make a return level plot.

```r
# Look at mcmc parameter estimates in each chain
lapply(hsig_mixture_fit$mcmc_chains, f<-function(x) summary(as.matrix(x)))
```

```
## [[1]]
##       var1             var2            var3              var4        
##  Min.   :0.6363   Min.   :0.766   Min.   :0.01096   Min.   :-0.4563  
##  1st Qu.:0.8155   1st Qu.:0.963   1st Qu.:1.08507   1st Qu.:-0.2530  
##  Median :0.8451   Median :1.014   Median :1.33749   Median :-0.2065  
##  Mean   :0.8454   Mean   :1.023   Mean   :1.32929   Mean   :-0.2033  
##  3rd Qu.:0.8752   3rd Qu.:1.071   3rd Qu.:1.60360   3rd Qu.:-0.1559  
##  Max.   :1.0636   Max.   :1.892   Max.   :2.17548   Max.   : 0.2241  
## 
## [[2]]
##       var1             var2             var3              var4        
##  Min.   :0.6349   Min.   :0.7414   Min.   :0.01888   Min.   :-0.4656  
##  1st Qu.:0.8155   1st Qu.:0.9629   1st Qu.:1.08671   1st Qu.:-0.2527  
##  Median :0.8455   Median :1.0135   Median :1.33692   Median :-0.2055  
##  Mean   :0.8453   Mean   :1.0242   Mean   :1.32622   Mean   :-0.2026  
##  3rd Qu.:0.8752   3rd Qu.:1.0717   3rd Qu.:1.60154   3rd Qu.:-0.1544  
##  Max.   :1.0451   Max.   :2.0512   Max.   :2.17544   Max.   : 0.1777  
## 
## [[3]]
##       var1             var2             var3              var4        
##  Min.   :0.6181   Min.   :0.7039   Min.   :0.03059   Min.   :-0.4485  
##  1st Qu.:0.8151   1st Qu.:0.9623   1st Qu.:1.08084   1st Qu.:-0.2524  
##  Median :0.8453   Median :1.0136   Median :1.33596   Median :-0.2053  
##  Mean   :0.8450   Mean   :1.0268   Mean   :1.32326   Mean   :-0.2022  
##  3rd Qu.:0.8757   3rd Qu.:1.0724   3rd Qu.:1.60005   3rd Qu.:-0.1548  
##  Max.   :1.0616   Max.   :2.2506   Max.   :2.17549   Max.   : 0.2301  
## 
## [[4]]
##       var1             var2             var3              var4        
##  Min.   :0.6469   Min.   :0.7713   Min.   :0.05713   Min.   :-0.4609  
##  1st Qu.:0.8156   1st Qu.:0.9632   1st Qu.:1.07948   1st Qu.:-0.2530  
##  Median :0.8456   Median :1.0139   Median :1.33580   Median :-0.2060  
##  Mean   :0.8455   Mean   :1.0236   Mean   :1.32533   Mean   :-0.2026  
##  3rd Qu.:0.8751   3rd Qu.:1.0716   3rd Qu.:1.59774   3rd Qu.:-0.1553  
##  Max.   :1.0307   Max.   :1.8757   Max.   :2.17550   Max.   : 0.2425  
## 
## [[5]]
##       var1             var2             var3              var4        
##  Min.   :0.6493   Min.   :0.7352   Min.   :0.02053   Min.   :-0.4378  
##  1st Qu.:0.8152   1st Qu.:0.9619   1st Qu.:1.07711   1st Qu.:-0.2533  
##  Median :0.8451   Median :1.0139   Median :1.33677   Median :-0.2056  
##  Mean   :0.8455   Mean   :1.0237   Mean   :1.32399   Mean   :-0.2029  
##  3rd Qu.:0.8755   3rd Qu.:1.0720   3rd Qu.:1.60027   3rd Qu.:-0.1549  
##  Max.   :1.0196   Max.   :1.7660   Max.   :2.17543   Max.   : 0.2823  
## 
## [[6]]
##       var1             var2             var3                var4        
##  Min.   :0.6434   Min.   :0.7362   Min.   :0.0006754   Min.   :-0.4561  
##  1st Qu.:0.8154   1st Qu.:0.9623   1st Qu.:1.0805607   1st Qu.:-0.2525  
##  Median :0.8451   Median :1.0142   Median :1.3349925   Median :-0.2057  
##  Mean   :0.8452   Mean   :1.0246   Mean   :1.3238637   Mean   :-0.2024  
##  3rd Qu.:0.8752   3rd Qu.:1.0727   3rd Qu.:1.6015236   3rd Qu.:-0.1543  
##  Max.   :1.0470   Max.   :1.8652   Max.   :2.1754875   Max.   : 0.3020
```

```r
# Look at ari 100 estimates
lapply(hsig_mixture_fit$ari_100_chains, 
    f<-function(x) quantile(x, p=c(0.025, 0.5, 0.975)))
```

```
## [[1]]
##     2.5%      50%    97.5% 
## 7.063619 7.539136 8.861741 
## 
## [[2]]
##     2.5%      50%    97.5% 
## 7.062581 7.543480 8.860358 
## 
## [[3]]
##     2.5%      50%    97.5% 
## 7.066485 7.541778 8.866045 
## 
## [[4]]
##     2.5%      50%    97.5% 
## 7.065698 7.539373 8.870109 
## 
## [[5]]
##     2.5%      50%    97.5% 
## 7.065144 7.539393 8.862385 
## 
## [[6]]
##     2.5%      50%    97.5% 
## 7.064374 7.543140 8.867450
```

```r
# Look at model prediction of the maximum observed value
# (supposing we observed the system for the same length of time as the data covers)
lapply(hsig_mixture_fit$ari_max_data_chains, 
    f<-function(x) quantile(x, p=c(0.025, 0.5, 0.975)))
```

```
## [[1]]
##     2.5%      50%    97.5% 
## 6.811203 7.187007 8.108811 
## 
## [[2]]
##     2.5%      50%    97.5% 
## 6.810462 7.189698 8.105421 
## 
## [[3]]
##     2.5%      50%    97.5% 
## 6.815542 7.189163 8.104913 
## 
## [[4]]
##     2.5%      50%    97.5% 
## 6.813876 7.188006 8.110017 
## 
## [[5]]
##     2.5%      50%    97.5% 
## 6.812937 7.187356 8.100880 
## 
## [[6]]
##     2.5%      50%    97.5% 
## 6.813079 7.190309 8.110748
```

```r
# If the chains are well behaved, we can combine all 
summary(hsig_mixture_fit$combined_chains)
```

```
##        V1               V2               V3                  V4         
##  Min.   :0.6181   Min.   :0.7039   Min.   :0.0006754   Min.   :-0.4656  
##  1st Qu.:0.8154   1st Qu.:0.9626   1st Qu.:1.0814071   1st Qu.:-0.2528  
##  Median :0.8453   Median :1.0138   Median :1.3363184   Median :-0.2058  
##  Mean   :0.8453   Mean   :1.0243   Mean   :1.3253241   Mean   :-0.2027  
##  3rd Qu.:0.8753   3rd Qu.:1.0719   3rd Qu.:1.6009353   3rd Qu.:-0.1549  
##  Max.   :1.0636   Max.   :2.2506   Max.   :2.1754981   Max.   : 0.3020
```

```r
# If the chains are well behaved then we might want a merged 1/100 hsig
quantile(hsig_mixture_fit$combined_ari100, c(0.025, 0.5, 0.975))
```

```
##     2.5%      50%    97.5% 
## 7.064642 7.541089 8.865505
```

```r
# This is an alternative credible interval -- the 'highest posterior density' interval.
HPDinterval(as.mcmc(hsig_mixture_fit$combined_ari100))
```

```
##         lower    upper
## var1 6.975574 8.593023
## attr(,"Probability")
## [1] 0.95
```

```r
evmix_fit$mcmc_rl_plot(hsig_mixture_fit)
```

![plot of chunk hsigmixtureFitBayesB](figure/hsigmixtureFitBayesB-1.png)