# Markov Chain Mixture Models for Real-Time Direct Illumination

## Introduction

To arrive at a general technique that can, in the future, be extended more easily to indirect illumination, we do not perform NEE but use only hemispherical samples to discover lights. In the spirit of recent path guiding approaches, we use parametric mixture models to represent and sample a target distribution. Our technique combines the simplicity and GPU-friendly fixed memory footprint of a single component per pixel with the expressiveness of a mixture of components by replacing a deterministic mixture with the equilibrium distribution of a Markov chain.

## An Estimator using a Randomized Mixture Model

$$
p(\omega)=\int p(\mathbf{t})V(\omega|\mathbf{t})\text{d}\mathbf{t}
$$

where $\mathbf{t}$ represents the component parameters and $p(\mathbf{t})$ the component weight.

We use the n-sample *stochastic multiple importance sampling* (SMIS) estimator proposed in continuous multiple importance sampling:
$$
\langle L_r(\mathbf{x}_1)\rangle_{\text{SMIS}}=\sum_{i=1}^n\frac{f_r(\mathbf{x}_1)L_e(\mathbf{x}_{2,i})\cos\theta_i}{\sum_{j=1}^np(\omega_i|\mathbf{t}_j)}
$$
We approximate the full PDF through a stochastic selection of lobes. $p(\mathbf{t})$ is not present in the expression, since it cancelled out. We define $p(\mathbf{t})$ through a Markov chain operating on a state space which allows us to compute the lobe's parameters $\mathbf{t}=(\mu,\kappa)$. Contrary to Metropolis-Hastings, which gives strong guarantees about the *exact* equilibrium distribution through detailed balance conditions, we do not depend on this reasoning.

For our use-case, we store one Markov chain state per pixel in a simple screen space data structure.

### A Markov Chain Mixture Model

Our mechanism has to partially opposing goals:

-   the equilibrium distribution should faithfully represent the target distribution
-   it should adapt fast to dynamically changing content

To address the first goal we use a maximum a-posteriori (MAP) approach to compute accurate distribution parameters. This makes the reduced state space non-Markovian, since transitions are affected by past states.

To allow picking up newly appearing lights quickly, we exchange Markov chain state between neighboring pixels.

Within one frame, each pixel will independently mutate their state in a loop with $n$ iterations. Each iteration consists of three stages:

-   We look at the states of neighboring pixels and potentially use their states to overwrite ours
-   We create a new path vertex $\mathbf{x}_2$ by drawing either an independent hemispherical sample or by using the current vMF lobe of this pixel
-   Through a mutation/acceptance step, we keep a current state $S_c$ over all iterations which will eventually be written out as $S^{(i+1)}_p$

 ### Sample Decorrelation

Using a sample both to advance the Markov chain and construct the SMIS estimator leads to dependencies between a sample and subsequent techniques and samples, resulting in a biased SMIS estimator.

Since we are running multiple Markov chains in parallel - one for each pixel, by permuting Markov chain states between pixels after each step, we break any sample dependencies in the SMIS estimator.

**Temporal SMIS (TSMIS)** To make sure MCMC and SMIS are completely independent in one pixel, it is enough to simply shuffle up the MCMC state after mutation and before evaluating the SMIS estimator.

**Spatial SMIS (SSMIS)** We choose to construct a combination of $n$ one-sample SMIS estimators instead of a true $n$-sample SMIS estimator.



