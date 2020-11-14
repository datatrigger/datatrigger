---
title: "Gradient tree boosting in the cloud"
summary: "A cloud computing experiment with two slightly different implementations of gradient boosted trees : LightGBM and XGBoost. Let us evaluate how these two algorithms do on a moderately large dataset, regarding both accuracy and speed."
date: 2020-11-13
tags: ["python", "xgboost", "lightgbm", "gradient boosted trees", "cloud computing", "machine learning", "superconductors", "paperspace"]
draft: true
---

### Introduction

The topic of the present post originates from the [superconductivity dataset](http://archive.ics.uci.edu/ml/datasets/Superconductivty+Data) --- available on the UCI --- and the associated [paper [1]](https://arxiv.org/abs/1803.10260).  available on the [UCI Machine Learning Repository](http://archive.ics.uci.edu/ml/index.php).

### References

[[1]](https://arxiv.org/abs/1803.10260)) HAMIDIEH, Kam : *A Data-Driven Statistical Model for Predicting the Critical Temperature of a Superconductor*, University of Pennsylvania, Wharton, Statistics Department, 2018.  