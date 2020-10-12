---
title: "Back to basics : Scaling train and test samples."
summary: "Splitting and scaling a dataset seems easy. Well, it is admittedly not that hard, however it can be tricky. Today we will see how to properly split and scale a dataset, as this step if often necessary before any ML wizardry. Let us do this with a few R & Python packages/modules."
date: 2020-10-12
tags: ["scaling", "normalize", "standardize", "spark", "pyspark", "python", "r", "dplyr", "caret"]
draft: true
---

### The importance of scaling

Scaling data is essential before applying a lot of Machine Learning techniques. For example, distance-based methods such as K-Nearest Neighbors, Principal Component Analysis or Support-Vector Machines will artificially attribute a great importance to a given feature if its range is extremely broad. As a consequence, these kind of algorithms require scaling to work properly. Although not *absolutely* necessary, scaling and specifically normalizing helps a lot with the convergence of gradient descent [[1]](https://arxiv.org/abs/1502.03167). Hence it is generally recommended for Deep Learning. On the contrary, tree-based methods such as Random Forest or XGBoost do not require the input features to be scaled.
  
Today, we will only consider the two main ways of scaling, i.e *standardization*, a.k.a Z-score normalization and *normalization*, a.k.a Min-Max scaling. The **standardized** version of the numerical feature $X$ is: $$\tilde{X} := \frac{X - \overline{X}}{\hat{\sigma}}$$ where $\overline{X}$ is the estimated mean of $X$ and $\hat{\sigma}$ its estimated standard deviation. The point of this transformation is that $\tilde{X}$ has an estimated mean equal to 0 and an estimated standard deviation equal to 1.  
  
On the other hand, **normalization** consists in the transformation: $$\tilde{X} := \frac{X - min(X)}{max(X) - min(X)}$$ Consequently, $\tilde{X} \in [0;1]$.

### Scaling *can* be tricky

In his excellent [blog post](https://sebastianraschka.com/faq/docs/scale-training-test.html), Sebastian Raschka explains why the only one way to properly split and scale a dataset is the following one :  
1. Train-test split the data
2. Scale the train sample
3. Scale the test sample **with the training parameters**  

Any other method (scaling then splitting or scaling each sample with its own parameters for example) is wrong because it makes use of information extracted from the test sample to build the model afterwards. Sebastian gives extreme examples to illustrate this in his post. Although improper scaling may have minor consequences when working with a large dataset, it can seriously diminish the perfomance of a given model if only a few observations are available.  
  
Programmatically speaking, splitting and scaling a dataset using the method presented above can be a little bit more troublesome than just scaling a set of observations by itself.
























### References

[[1]](https://arxiv.org/abs/1502.03167) IOFFE, Sergey, SZEGEDY, Christian : *Batch Normalization: Accelerating Deep Network Training by Reducing Internal Covariate Shift*, Google Inc., 2015.  