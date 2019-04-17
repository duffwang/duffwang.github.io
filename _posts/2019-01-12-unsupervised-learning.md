---
layout: post
title: In-depth Look at Unsupervised Learning with Python
---

I perform several experiments with unsupervised learning techniques to illuminate the advantages and disadvantages of each. In particular, I take a look at k-means clustering, expectaction maximization, principal component analysis, independent component analysis, randomized projections, and factor analysis. I compare two publically available datasets with different characteristics to demonstrate the advantages and disadvantages of each unsupervised learning algorithm (the same datasets as the previous blog post on supervised machine learning).

### Code

All code used for this assignment, including data cleaning and plot generation, can be found on my GitHub [here](https://github.com/duffwang/UnsupervisedLearning). This blog post will concentrate on analysis rather than coding.

### Datasets:

The first dataset is the [Phishing Websites Data Set](https://archive.ics.uci.edu/ml/datasets/Phishing+Websites) which can be found on the UCI Machine Learning Repository.

It is an interesting dataset due to phishing website detection being a non-trivial real-world problem, with phishing website authors in an arms race to escape notice from fraud detection algorithms. It was used by researchers successfully, so it should be high quality enough for our algorithms to classify well too, and with its large number of features it may illustrate differences between different algorithms. 

Due to the large number of data points in the Phishing dataset (over 10k), I randomly select 6k rows to keep our runtimes manageable. The data is balanced, with our subset having a **3327 : 2673 true/false count** (6000 total data points).

The second dataset I use is the [Wine Quality Data Set](https://archive.ics.uci.edu/ml/datasets/wine+quality) also found on the UCI Machine Learning Repository. 
	
It is interesting because a common debate about wine quality is whether it can be predicted by quantifiable attributes or is mostly subjective; there may be a natural limit to how well supervised learning can do as a result. 

I one hot encode this multi-class problem into a binary one conditioned on whether wine quality, as measured by the scale used in the dataset, is greater than 6. Our data is slightly imbalanced, with a **2604 : 1314 true/false count** (3918 data points).

### Experiment 1: Clustering Algorithms

We apply two clustering algorithms to each dataset, K-Means and MNIST. In order to determine what I must change to make the algorithms better, I must find the optimal hyperparamaters.
In K-means, random centers in feature space are initialized.  Convergence is achieved by applying cycles of expectation and maximization, where expectation associates every point with the closest mean according to Euclidean distance, and maximization updates the means to best describe the clustered points. In our experiments, we will run K-means several times and take the best result, since the results are dependent on the random initialization. 

I chose to use the elbow method in finding the best value of k. The reason for choosing this method is that it has a sound theoretical basis for choosing the optimal k value: the point at which the percentage of variance explained no longer improves by increasing the number of clusters.

In Expected Maximization, our algorithm takes a current posterior and tries to maximize the probability that any particular point is part of a cluster. I used the log likelihood metric to determine how well the data points fit the cluster (where the log likelihood is used instead of likelihood due to many convenient mathematical features, such as the ability to separate and add the log likelihood of each individual data point belonging to the cluster). This is iterated until convergence.

Instead of trying to visualize the clusters themselves, which would be messy and hard to analyze, I chose to use objective evaluation criteria: namely homogeneity and silhouette scores, which are explained later.

#### K-Means

I chose Euclidean distance because our data has already been normalized, making other metrics such as cosine similarity not advantageous. To choose a value for k, I ran k-means for k from 3 to 11.  
For the wine data (on right), we see that each additional cluster explains less variance than the last, but there is no sharp cutoff, so we cannot find an optimal k from this plot alone. 

<img src="/img/u1.png" width="800px"/>

We compare the true labels we have and look at the homogeneity (all clusters contain only members of a single class) and silhouette scores (all data points resemble members of its clusters but not others). Homogeneity increases with number of clusters as to be expected for data having only a binary label; silhouette score peaks at k = 4, which makes that the cluster size I will choose.

For the Phishing data (on left), the variance plot starts to flatten out at around k = 4-7. Homogeneity shows a major improvement at k = 4; above that, our clusters become less homogeneous and have poorer silhouette scores, indicating k = 4 is the optimal value I will choose. 

#### Expected Maximization

With expected maximization, we also much choose an optimal number of clusters that the algorithm will then maximize log-likelihood for. We look at the same performance metrics as we did for k-means: homogeneity, silhouette. Per sample average log likelihood is used to look at how effective each additional cluster is at fitting our datasets.

<img src="/img/u2.png" width="800px"/>

For the Phishing data, the variance plot flattens out at k =5. Homogeneity shows a major improvement at k = 5; above that, our clusters become less homogeneous and have poorer silhouette scores, indicating k = 5 is the optimal value I will choose. 

For the wine data, we see that variance explained by each cluster loses its slope at k = 5. That is also regime at which homogeneity scores and silhouette scores flatten out, though we don’t see an obvious peak, meaning the wine data is not well suited to being clustered. The best choice is k = 5.
There were not significant differences between k-means and expected maximization. That indicates our datasets do not have many clusters with non-Gaussian distributions that have different variances. 

### Experiment 2: Dimensionality Reduction

The methodology for these experiments was to run the various dimensional reduction techniques on the datasets, varying the number of components we wish to dimensionally reduce to. Instead of trying to visualize the basis vectors themselves in feature space, which would be difficult and hard to analyze visually, I plotted relevant metrics for each technique (variance explained by basis vector for PCA, kurtosis explained by basis vector for ICA, and noise variance for FA). 

The dimensionally reduced datasets were then used in a decision tree classifier to generate cross validation accuracy. The decision tree classifier was run multiple times and the mean CV accuracy was reported. 

#### Principal Component Analysis (PCA):

In PCA, orthogonal basis vectors are chosen in feature space according to a simple algorithm: pick the basis vector that explains the most variance at any given point. This is done with eigenvector decomposition.

<img src="/img/u3.png" width="700px"/>

For the phishing data, at n = 16 we see most of the variance explained and no more improvement in cross validation accuracy, indicating this is a well-chosen parameter (we could say the same about all parameters between n=10 to 16, but it does look like there is a small improvement in accuracy, so I go with n = 16). 

For the wine data, almost all variance is explained with 2 components. Increasing dimensions after that does not give us greater information. However, cross-validation accuracy continues to improve with number of components. This can be explained because PCA treats the variable with that explains the most variance as the most important feature, but doesn’t consider the response variable / prediction target, causing it to eliminate potentially useful features. We choose n = 10 as our optimal value, with the greatest CV accuracy.

#### Independent Component Analysis (ICA):

In ICA, the idea is that we wish to separate out different independent components that have been mixed together, for example separating out two pictures that have been blended together. To do this, we distinguish these independent components from Gaussian noise by looking at how much kurtosis (the fourth moment or ‘tailedness’ of the probability distrubutions) can be explained by each basis vector. 

<img src="/img/u4.png" width="700px"/>

For the Phishing dataset, n_components is fairly high at 23, and kurtosis explained by each component is not high. Accuracy ends up being as accurate as with PCA, but with more components, indicating the underlying data is not best explained as independent vectors in feature space. 
	
For the Wine dataset, all the kurtosis is explained by 6 components. This fits in with what we previously know about the wine dataset, namely that there is significantly more noise in the data and many of the features may not be correlated with good clusters. ICA may be a good choice for the wine dataset due to the high amount of kurtosis explained by the first few component. The chosen parameters is n = 6 where accuracy peaks and no further kurtosis is explained by adding more basis vectors.

#### Randomized Projections (RP):

In Randomized Projection, a transformation matrix transforms the N features into M features using an N x M matrix where each value is random from a Gaussian distribution. The Gaussian distribution used has zero mean and variance that is the inverse of number of components. 

<img src="/img/u5.png" width="700px"/>

The randomized projection was run multiple times and averaged out. Variation was fairly high when I re-ran the RP algorithm.

Reconstruction error from the RP algorithm is shown and discussed later, when I compare all four algorithms.

For the Phishing dataset, accuracy decreases as we reduce dimensions, consistent with our previous analysis that most components of the dataset contains information. Even with 29 components accuracy is significantly less than other dimension reduction techniques, because information is lost. We use n = 29 which has the best accuracy.

For the Wine dataset, accuracy is also less than other techniques, but there is less of trend. We choose the best accuracy at n = 6, but due to the volatility in the results, it is likely RP is not a good way to reduce the dimensionality of this dataset.Again, information is being lost, but the advantage of RP is that it may be faster to run techniques on dimensionally reduced datasets, which we will analyze in experiment 5. 

#### Factor Analysis (FA):

In factor analysis, the idea is that there are various unobservable factors that explain the observable data. The observed data is assumed to be some linear combination of these factors with some error term. Expectation maximization is then used to fit the matrix that transforms the factors to the observed data.

<img src="/img/u6.png" width="700px"/>

With the Phishing dataset, we see accuracy of the decision tree classifier peaks fairly fast, but stays at around 90%, less than other dimensional reduction techniques. Gaussian noise variance attributable to each n_component is shows no trend. This suggests latent factors can be a reasonable explanation of the observed phishing data.

With the wine dataset, accuracy drops quickly as we reduce the number of components. Noise variance is fairly bimodal between zero and one. The uneven distribution of noise variance suggests latent factors is not a good model for this data, as we are overfitting the data when noise variance is zero and underfitting when noise variance is one.

#### Reconstruction Error Comparisons:

<img src="/img/u7.png" width="700px"/>

Finally I show reconstruction errors for each model. Random Projection reconstruction error was scaled down in order to display it on a plot; it was orders of magnitude higher than the other models since every component is mixed together into new ones, making it difficult to reconstruct. The other models show a consistent decrease in reconstruction error as components goes up, as we would expect due to more information retained.

### Experiment 3: Clustering after Dimensional Reduction

Now, we re-run both clustering algorithms on our dimensionally reduced datasets to compare differences in clustering effectiveness, and analyze why using various evaluation metrics.

#### K-means:

<img src="/img/u8.png" width="700px"/>

For k-means clustering, we look at the homogeneity scores (how homogenous each cluster is) and silhouette scores (how similar each data point is to its own cluster and dissimilar to other clusters).
For the Phishing dataset, it is clear that homogeneity is higher for FA than others. This suggests overfitting as number of components gets higher. This is corroborated by the silhouette scores which remain roughly constant, and are higher than any other method. 

My conclusion is that FA is the best method for the Phishng dataset, but we should keep the number of clusters low to avoid overfitting. For ICA homogeneity remains low, which also is not great as it suggests the clusters are not clearly differentiable. Silhouette scores are also low, suggesting the data is poorly modelled by independent factors.

The wine dataset shows an interesting trend: homogeneity does not significantly rise for any of the dimensional reduction technique as n_components rises except for ICA. Since the label is binary, what is likely happening is that with ICA even a low number of clusters has already captured most of the information, and adding new clusters does not help, so using independent components is probably not a great way to dimensionally reduce due to the underlying dataset not being modelled well by ICA. 
	
With ICA, there is an upwards trend, but silhouette scores are very low, indicating independent factors is not a great model and we are overfitting as we increase the number of clusters. On the other hand, RP actually shows high silhouette scores and reasonable homogeneity scores; even though we previously showed RP sacrifices accuracy, the reasonable k-means clustering evaluation scores on RP means it is a viable way to reduce dimensions while not sacrificing too much.

#### Expected Maximization

<img src="/img/u9.png" width="700px"/>

For expected maximization, we look at silhouette and homogeneity scores, but I also chose to look at completeness scores (all data points of a label are in the same cluster) to provide more insight into the best dimensionality reduction method.

For the Phishing dataset, the clear winner is FA, which maximizes completeness, homogeneity, and silhouette scores at almost every level of n_components chosen. This aligns with the results for k-means. The best explanation is that phishing websites have common factors that result in the observable data, for example certain common phishing techniques (unobservable) that results in common traits. ICA once again performs the most poorly, as independent components doesn’t well describe our phishing dataset, where all features are likely not independent.
	
For the wine dataset, although ICA and FA appear to do well in terms of completeness and homogeneity, silhouette scores are low, indicating the expected maximization clusters is creating many clusters that are similar, which makes the clustering not useful. The best metric is likely once again RP,  which creates clusters with the best balance of the evaluation scores.

#### Remarks

**What are the differences between k-means and expected maximization on the different dimensional reduction techniques?**

My results did not show significant differences between the two clustering techniques. The best dimensional reduction technique was the same in both cases, with similar trends for number of clusters chosen. This is expected since even in our non-reduced case, we saw both clustering algorithms performing similarly. 

We would expect to see deviations between the two algorithms if our clusters in the datasets performed better when they are non-Gaussian and had different variances, since expected maximization would pick up on these patterns, while k-means does not since k-means only groups everything by simply distance to the cluster mean. Our datasets do not show these characteristics.

**Were the clusters the same when I re-ran the clustering algorithms on the reduced datasets?**

No, the clusters formed were clearly different depending on the dimensional reduction algorithm as my evaluation metrics showed. Many of the transformed datasets ended with poor clusters that had weak silhouette scores, such as ICA for both datasets/clustering algorithms. 

### Experiment 4: Neural Network after Dimensionality Reduction and with/without Clustering

I re-ran the dimensionally reduced Wine dataset through the neural network I used in the previous blog post. I performed it without clustering and with clustering (using EM and K-means) and plot components versus three-fold cross-validation.

<img src="/img/u10.png" width="700px"/>

<img src="/img/u11.png" width="700px"/>

In order to apply clustering to the dataset, the cluster labels were appended to the dimensionally reduced dataset, and then the neural network classifier was run on that newly created dataset.
Looking at neural network accuracy results after dimensionally reducing the datasets in different ways, we see there is no significant difference in neural network accuracy for FA, PCA, and RP. The cluster labels don’t actually add information that is inherently new, so it is likely the neural network is able to classify just as well without them, as the neural network hidden layer weights will implicitly consider data that are similar to each other.

The exception is ICA, which shows weaker accuracy with clustering then without. This is in line with my previous analyses which showed clustering on ICA data produces poor clusters with low silhouette scores and that are homogeneous, meaning they don’t discriminate well between truly different clusters of data, which adds more noise the neural network is forced to use and potentially overfit on.

In terms of accuracy between the dimensional reduction algorithms, we see that RP produces datasets that leads to significantly lower cross-validation accuracy than the other datasets, indicating we are losing a significant amount of information when reducing this way. The other three dimensional reduction techniques are fairly similar in terms of absolute accuracy, and all trend towards poorer accuracy as n_components is reduced in a similar way. There is a consistent steep decline at n = 4, which is consistent with our previous analysis that showed that the optimum number of dimensions for the wine dataset is always more than 4 (usually around 6-9). This likely shows that the wine dataset has a fair number of features with poor information, but all forms of dimensional reduction do sacrifice some accuracy due to the increasing slope shown in the graphs. 

<img src="/img/u12.png" width="700px"/>
 
In turn, then, we should receive gains in neural network training speed due to having less features to work on. The graph above shows neural network training speed does significantly decrease at low dimensions compared to high dimensions. One exception is RP, which shows less of a trend, possibly because the information lost even at higher dimensions that the neural network is able to converge quickly.

