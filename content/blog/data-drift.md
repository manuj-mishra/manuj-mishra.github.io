---
title: "Monitoring input data drift using PCA"
date: 2022-09-09
draft: false
math: true
description: "Using principal component analysis autoencoders to track changes in input data distribution"
comments: false
tags: ['project', 'machine learning']
---

Machine learning models are built on the assumption that training data is sufficiently representative of test data. However, for many use cases, the distribution of test data may change over time to the extent that this assumption no longer holds. This is called input data drift. Detecting input data drift is a crucial first step towards mitigating its effects, and we can make use of autoencoder-style architectures to do this. In this post I'll discuss a PCA-based method.

### Brief aside: What is PCA?
PCA, principal component analysis, is a transformation which takes a dataset and projects it into a vector space whose basis is the first $n$ principal components (PCs) of that dataset. By projecting the data into this new subspace, we get an encoding of the data that preserves the most salient variations. The $1^{st}$ PC is a unit vector in the direction of maximal variance. The $i^{th}$ PC is a vector orthonormal to the previous $i-1$ PCs in the direction of maximal variance when the effect of the first $i-1$ PCs is removed from the data. This is also equivalent to minimizing the reconstruction error. In the animation below, we see that the sum of the red lines is minimal when the projections of the points onto the thick black line (i.e. the red dots) is maximally spread out. This black line is the direction of the first principal component.

![PCA calculation](/img/data-drift/pca.gif "Visualisation of first principal component calculation[1]")

### What is an autoencoder?
An autoencoder is a neural network designed to learn an efficient latent representation of some data. It consists of two parts. The encoder is trained to extract a low-dimensional representation of the data, ignoring noise. The decoder is trained to regenerate something close to the original data from this representation. By looking at the distribution of the latent representations of lots of training data, the thinking goes, we can get a good understanding of the "essence" of the data. Then, if a new piece of data is "far away" from this distribution in latent space, we can infer that any model trained on the previous dataset may not perform so well on this new data. To measure how "far away" this new data is, we can use a distance metric on the latent space or we can measure reconstruction error between the original data and the output of the decoder.

![Autoencoder](/img/data-drift/autoencoder.jpg "Autoencoder architecture[2]")

### PCA-based data drift monitoring
This brings us back onto PCA. PCA is a very simple linear encoding which we can use as a rudimentary "autoencoder". We find the top $n$ PCs of the training data and track the reconstruction error as each piece of data is projected into the subspace spanned by those PCs. This is our measure of data drift. If the quantity of test data is high we may want to only calculate data drift for some subset (e.g. every $100^{th}$ datapoint or at intervals of 5 seconds). If it comes in batches, we may wish to calculate the average data drift of each batch. Regardless of implementation, we can plot this as a time series and establish bounds on the data drift based on the variance seen in batches of training data. We can then create a downstream system that alerts us when data drift has exceeded our pre-set bounds. Hence, we have detected data drift.

![Monitoring data drift](/img/data-drift/monitoring.png "Monitoring data drift[3]")

## References

[1] https://stats.stackexchange.com/questions/2691/making-sense-of-principal-component-analysis-eigenvectors-eigenvalues

[2] https://blog.keras.io/building-autoencoders-in-keras.html

[3] https://towardsdatascience.com/data-drift-explainability-interpretable-shift-detection-with-nannyml-83421319d05f
