---
title: "Shiny Central Limit Theorem"
summary: "The central limit theorem is one of the greatest hits in the history of statistics. I wrote a little Shiny app to visualize it and to illustrate its infamous \"counterexample\", Cauchy distribution: https://datatrigger.shinyapps.io/CLT_Visualization/."
date: 2020-11-28
tags: ["r", "shiny", "data visualization", "central limit theorem"]
draft: false
---

### The Galton Board

As an introduction to this post, let me introduce one of the geekiest objects I own: a [Galton Board](https://en.wikipedia.org/wiki/Bean_machine), also known as a *bean machine* or a *quincunx*.

![Vincent's Galton Board](/res/shiny_clt/galton_board_1.resized.jpg)

It is named after the polymath of genius [Francis Galton](https://en.wikipedia.org/wiki/Francis_Galton), well-known for coming with the statistical term *regression*. I bought it after watching [Vsauce's video](https://youtu.be/UCmPmkHqHXk) on the subject. It is a great illustration of the central limit theorem because it shows how a binomial distribution --- as the sum of Bernoulli variables --- approximates a normal distribution when the number of trials $n$ is large enough.

![Vincent's Galton Board](/res/shiny_clt/galton_board_2.resized.jpg)

### The central limit theorem

The CLT comes in many tastes and flavors and it can be found in about a zillion math books. However, writing this article without explicitly mentioning at least its "standard" version is inconceivable ! Let $(X_i)_{i \in \mathbb{N}}$ be a sequence of independent and identically distributed random variables. Suppose that the associated distribution admits finite expected value $\mu$ and variance $\sigma²$. Then the sequence:

$$(\frac{\overline{X_n} - \mu}{\frac{\sigma}{\sqrt{n}}})_{n \in \mathbb{N}}$$

converges in distribution to the standard normal distribution $\mathcal{N}(0,1)$, where

$$\overline{X_n} = \frac{1}{n} \sum\limits_{i=1}^{n} X_i \ , \ n \in \mathbb{N}$$  

### Visualizing the CLT

The [*Shiny CLT*](https://datatrigger.shinyapps.io/CLT_Visualization/) app is currently deployed on shinyapps.io. It computes samples, then the mean of these samples, for the following distributions:

* Continous distributions: **Cauchy**, exponential, log-normal and uniform
* Discrete distributions : binomial and Poisson

The **Cauchy** distribution does not satisfy the CLT hypotheses as its mean and variance are undefined, consequently to the divergence of the associated integrals. The density shown in the Shiny app has its location parameter $x_0$ set to 0 and its scale $\gamma$ set to 1. We can see how close it is to the density of a standard normal distribution, except for the heavier tails. The parameters of the other distributions are the default parameters of the random variate generation functions available in the R package ```stats```.
  
For each of the studied distributions, the app displays a histogram of the computed means. It lets the user define the *size* $n$ of the samples generated to compute those means, as well as the *number of samples*. The *number of bins* for the histograms is also adjustable. It corresponds to the parameter *bins* in ggplot2's ```geom_histogram()```.  
  
When the size of each sample is equal to 1, then the computed mean of a given sample is just the value of its unique observation. In this case, we obtain histograms of the standardized original distributions:  

![original distributions](/res/shiny_clt/original_distributions.png)

We can see that as we increase the sample size, the histograms of the means gets more and more close to the density of a standard normal distribution. That is the beauty of the central limit theorem shining live:  

![CLT convergence](/res/shiny_clt/convergence_100_bins.png)

The Cauchy distribution magnificently stands out as a "counterexample" of the CLT. As usual, the source code of the Shiny app is available on [GitHub](https://github.com/datatrigger/shiny_apps).
