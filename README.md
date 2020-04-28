### A very simple bayesian model exercise to "correct" crude prevalence based on sensitivity and specificity

This is inspired by the [COVID-19 seroprevalence study](https://www.medrxiv.org/content/10.1101/2020.04.14.20062463v1.full.pdf) conducted in Santa Clara County of California. The study tested 3330 persons (not completely random) and found 50 persons had the SARS-CoV-2 antibodies, i.e. about 1.5% prevalence. They did some reweighting and conclude that, on average, the population prevalence was 2.5 to 4.2%. Which is high and the study immediately invited multiple (just) criticisms. Many criticisms were on the calculation of confidence intervals and specificity of the test kit. One long critic can be seen [here](https://statmodeling.stat.columbia.edu/2020/04/19/fatal-flaws-in-stanford-study-of-coronavirus-prevalence/).

I was thinking of how to "correct" the prevalence based on sensitivity and specificity information. And decided to do a simple exercise on it. This work is SIMPLE, and by no means usable for this particular study (or any others). But it's fun to model, it's fun to do Bayesian, so why not try something...

First let $p_{prev}$ be the overall prevalence of SARS-CoV-2 antibodies, $p_{sens}$ be the sensitivity and $p_{spec}$ be the specificity. The study states that out of 122 samples known to be positive, 103 of them were tested positive using the kit (TP); 401 samples known to be negative, 399 of them tested negative (TN). So these will help us to model the sensitivity and specificity:

\begin{align*}
w_{TN} & \sim Bin(401,p_{spec})\\
w_{TP} & \sim Bin(122,p_{sens})
\end{align*}

Let $y_i$ be the test outcome of individual $i$, and $z_i$ be the true antibody status of this individual. $z_i$ is not known. So:

\begin{align*}
z_{i} & \sim Bernoulli(p_{prev})\\
y_{i}|z_{i}=0 & \sim Bernoulli(1-p_{spec})\\
y_{i}|z_{i}=1 & \sim Bernoulli(p_{sens})
\end{align*}

The model for $y_i$ can be simplified to:
\[
y_{i}\sim Bernoulli[(1-p_{spec})\times(1-z_{i})+p_{sens}\times z_{i}]
\]

$z_i$ is troublesome; it is a discrete latent variable, which can be handled by JAGS but not Stan. Nevertheless, we now need to estimate an extra $n$ parameters in the MCMC framework; not good if $n$ large. Good thing is we can integrate it away:
\[
y_{i}\sim Bernoulli[(1-p_{spec})\times(1-p_{prev})+p_{sens}\times p_{prev}]
\]

Here's the JAGS way of fitting it:

```r
library(R2jags)

model <- function() {
  tn ~ dbin(pspec, negs)
  tp ~ dbin(psens, poss)
  
  for (i in 1:N) {
    y[i] ~ dbern(psens * prev + (1 - pspec) * (1 - prev))
  }
  
  pspec ~ dbeta(1, 1)
  psens ~ dbeta(1, 1)
  prev ~ dbeta(1, 1)
}

N = 3330
poss = 122
negs = 401
tp = 103
tn = 399
# 50 individuals with test outcome positive the rest negatives
y = rep(c(0, 1), c(3330-50, 50)) 

data <- list(N = N, poss = poss, negs = negs, tp = tp, tn = tn, y = y)
mod <- jags(data, parameters.to.save = c("pspec", "psens", "prev"), 
            model.file = model)
```

```
## module glm loaded
```

```
## Compiling model graph
##    Resolving undeclared variables
##    Allocating nodes
## Graph information:
##    Observed stochastic nodes: 3332
##    Unobserved stochastic nodes: 3
##    Total graph size: 3344
## 
## Initializing model
```

And the result is in:

```r
mod
```

```
## Inference for Bugs model at "C:/Users/TKB/AppData/Local/Temp/RtmpiodTT0/model3b20706d3a97.txt", fit using jags,
##  3 chains, each with 2000 iterations (first 1000 discarded)
##  n.sims = 3000 iterations saved
##          mu.vect sd.vect    2.5%     25%     50%     75%   97.5%  Rhat
## prev       0.011   0.005   0.001   0.007   0.011   0.014   0.019 1.012
## psens      0.837   0.034   0.763   0.816   0.839   0.860   0.898 1.003
## pspec      0.993   0.003   0.986   0.991   0.994   0.996   0.999 1.008
## deviance 529.202   2.265 526.560 527.570 528.662 530.261 535.012 1.005
##          n.eff
## prev       300
## psens      860
## pspec      300
## deviance   570
## 
## For each parameter, n.eff is a crude measure of effective sample size,
## and Rhat is the potential scale reduction factor (at convergence, Rhat=1).
## 
## DIC info (using the rule, pD = var(deviance)/2)
## pD = 2.6 and DIC = 531.8
## DIC is an estimate of expected predictive error (lower deviance is better).
```

The unweighted prevalence has a CI ranging from 0.1% to 1.9% with a mean of 1%. This certainly has a different picture from "uncorrected" sample mean of 1.5% and CI of 1.1 to 2.0%.
