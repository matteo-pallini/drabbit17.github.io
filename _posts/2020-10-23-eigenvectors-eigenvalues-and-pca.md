---
layout: posts
comments: true
author_profile: true
title: Few intuitions on eigenvectors, eigenvalues and their use in PCA 
excerpt: an overview of what eigenvectors and eigenvalues are and why they make PCA work
date: 2020-10-23
---

When reading about PCA I was quite surprised about the use of eigenvectors and eigenvalues. I remembered them from Uni, although I didn't quite remember what they were meant to be used for. Neither I did remember what they were meant to represent. This blog post tries to give an intuition on what eigenvectors end eigenvalues are and why they make PCA work.

In this post I am mostly interested in outlining few intuitions regarding the “why it works”, rather than repeating an easy-to-find mathematical proof. Given that, I will skip anything that I think is already well explained somewhere else and simply refer to that resource. The value of this post is mostly in combining good resources together in a single cohesive and hopefully complete picture.

## Eigenvectors and Eigenvalues

In a nutshell when a linear transformation is applied to a vector and this remains on its own span, then that vector is an eigenvector of the transformation, and by how much it is squashed/stretched (and possibly flipped) depends on its eigenvalue. On the other hand, when the same transformation is applied to any other non-eigenvector, this will be somehow rotated and therefore moved outside of its span.

Just a refresher, the span of a list of vectors is simply the set of all their possible linear combinations. When looking at a single vector, its span, being the set of all its linear combinations is simply given by all the points resulting from combining its values with any possible scalar value, which, in 2-d space, is effectively the line passing through the vector coordinates and the origin. There is a [great video about spans](https://www.youtube.com/watch?v=k7RM-ot2NWY) made by 3blue1brown.

Going back to eigen-stuff, the description given above is usually presented in formal notation through the following equation (for simplicity I replaced the linear transformation with a matrix):

$$Av=\lambda v$$

Where $$A$$ is a matrix (ie linear transformation), $$v$$, one of its eigenvector, which once multiplied by $$A$$ gets transformed by a factor of $$\lambda$$ (which is a scalar). The scalar multiplication with $$\lambda$$ is effectively shifting $$v$$ on its span.

Thinking visually of the eigenvectors as a vector that remains on its own span and it’s shifted by its eigenvalue really helped me to start grasping the intuition behind them. Still, a voice in the back of my mind kept on saying “sure, cool, who cares?”. Although, few examples made me it easier to realize why I may have cared.

If anything mentioned above is not very clear, surely it will after having watched the great video on [eigenvectors/eigenvalues video by 3blue1brown](https://www.youtube.com/watch?v=PFDu9oVAE-g).

## Few examples of eigenvalues and eigenvectors applications

#### When the transformation is a rotation ([Source](https://youtu.be/PFDu9oVAE-g?t=241))
Given that as we said when the transformation is applied the eigenvector is not moved away from its own span, this vector is effectively the axis of rotation of the transformation itself. So, If you are in a 3D space, it’s probably easier to think of a rotation as the combination of axes of rotation and the angle by which it rotates, rather than in terms of the matrix used to apply the transformation.

#### When we need to compute the power of a matrix ([Source](https://youtu.be/PFDu9oVAE-g?t=782))
Computing the power of a non-diagonal matrix is a fairly painful process. However, if that matrix has enough eigenvectors to span the whole space you can change the coordinate system of the transformation so that the eigenvectors are the basis. In this new coordinate system, the transformation is expressed through a diagonal matrix having the eigenvalues on its diagonal. It’s now much easier to compute whatever power of the diagonal matrix and then move the final result back to the original coordinate system once done.

#### When transforming a linear transformation into a set of independent actions makes it easier to reason about it ([Source](https://math.stackexchange.com/questions/23312/what-is-the-importance-of-eigenvalues-eigenvectors))
The link in the title provides a good example of this scenario. The components of a system of interdependent differential equations can be decoupled through expressing them as a matrix and finding its eigenvectors and eigenvalues. In general, eigenvectors and eigenvalues may be useful any time that looking at a transformation as a set of independent actions on different directions may help.

## At last! why using them in PCA
PCA’s goal is reducing the dimensionality of the data while losing as little information as possible in the process. This goal is achieved by finding the projection of the data in which the variance is maximized. Often in the original data, the information/variance available is spread over the different dimensions/features, some of which may also be highly correlated. While through PCA we obtain a projection of the original data in which, often, most of the information/variance is available in a small number of dimensions. Also, all the dimensions of the projected data are independent.

Surprisingly, this projection is also the one that minimizes the error to reconstruct the original data starting from the projection. The error is the distance between the original points and their projection. A brilliant intuition for why the two results concur is given in [Amoeba’s answer here](https://stats.stackexchange.com/questions/2691/making-sense-of-principal-component-analysis-eigenvectors-eigenvalues). Fundamentally, if we think of a scatter of points in a two-dimensional plane, whatever projection you pick (think of any possible line passing through the center of the scatter) the distance between its center, which is also the center of the scatter, and each single data point is fixed. Through the Pythagorean theorem, we can see that this distance for each data point is given by the sum of the squared distance between the center and the projected point (effectively variance) and the squared distance between the projected point and the original point (effectively squared residual). So, by picking a different projection, through rotating the line on its center, you increase one of the two components by reducing the other by the same amount, so maximizing the variance, the error gets minimized.

Now, let’s focus on how the eigenvectors and the eigenvalues make it possible to find this projection. Again, I am going to focus more on the intuition, rather than the proper mathematical explanation, which can be found [here](https://stats.stackexchange.com/questions/10251/what-is-the-objective-function-of-pca/10256#10256) (or in many other textbooks :-) ). So, as mentioned we want to find a projection of the centered data matrix $$X$$ that in which the variance is maximized. 

So the best components of this projection should maximize $$w^TSw$$, where $$w$$ is a vector of unit length and $$S$$ is the variance of the matrix $$X$$ (I haven’t spent too much time on this, but the unit length of $$w$$ should be needed to keep the problem well defined and avoid that the maximum of $$w^TSw$$ is infinite).

Being $$S$$ symmetric positive matrix we know that it can be transformed into a diagonal matrix if we change its basis to the eigenbasis.

$$Q^{-1}SQ =\Lambda$$

So, we have:

$$w^TSw=w^TQ\Lambda Q^{-1}w = v^T\Lambda v$$

Where $$\Lambda$$ is a diagonal matrix having $$S$$ eigenvalues on its axis. Moreover it is also the case that $$\lambda_1>=\lambda_2>=...\lambda_d$$ for each $$i$$ where $$d$$ is the number of dimensions. Given that the $$\Lambda$$ matrix is diagonal the $$v^T\Lambda v$$ is equal to $$\sum_{i=0}^d \lambda_i v_i^2$$. Also, being $$Q$$ an orthogonal matrix and having $$w$$ unit length, also the length of $$v$$ is 1. Given that the lambdas are ordered in decreasing order and that $$v$$ has to be unit length, then the first eigenvalue available maximizes $$\sum_{i=0}^d \lambda_i v_i^2$$ and consequently the variance of the projection. So, we want to have w equal to $$[1, 0, …, 0]$$. Therefore, of all the possible projections the one resulting from the first eigenvector is the one maximizing the variance. 

Again, the derivation here is quite rushed, you can find a more thorough one on [Cardinal’s answer here](https://stats.stackexchange.com/questions/10251/what-is-the-objective-function-of-pca/10256#10256). Depending on how many dimensions we want to have in the projected matrix we keep on picking the next unused eigenvalues/eigenvectors and used them to compute their projected vectors.

## Few interesting resources
While reading about the topic I found particularly useful the following resources, I have already mentioned some of them in the post:
- As already mentioned few times [3blue1brown explanation of eigenvectors and eigenvalues is majestic] (https://www.youtube.com/watch?v=PFDu9oVAE-g)
- [Intuition on how the variance is maximized and the error is minimized at the same time](https://stats.stackexchange.com/questions/2691/making-sense-of-principal-component-analysis-eigenvectors-eigenvalues)
- [Proper math of how eigenvectors/eigenvalues are used in PCA](https://stats.stackexchange.com/questions/10251/what-is-the-objective-function-of-pca/10256#10256)
- [Less mathy and intuition focused explanation of the previous point](https://stats.stackexchange.com/questions/217995/what-is-an-intuitive-explanation-for-how-pca-turns-from-a-geometric-problem-wit)
- [Another nice explanation](http://www.stat.cmu.edu/~cshalizi/uADA/12/lectures/ch18.pdf) of several points and also [Another good resource](https://joellaity.com/2018/10/18/pca.html)
