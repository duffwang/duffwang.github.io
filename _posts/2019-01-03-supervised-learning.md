---
layout: post
title: Supervised Learning
subtitle: Deep Dive into Supervised Machine Learning
tags: [machine learning, supervised learning, data science, analysis, python]
image: /img/1.png
---

In this blog post, I do an in-depth dive into five common supervised learning algorithms: Decision Trees, AdaBoost, k nearest neighbors, support vector machines (SVMs), and neural networks.

I compare two publically available datasets with different characteristics to demonstrate the advantages and disadvantages of each supervised learning algorithm.

### Code

All code used for this post, including data cleaning and plot generation, can be found on my GitHub [here](https://github.com/duffwang/SupervisedLearning). This blog post will concentrate on analysis rather than coding.

### Datasets:

The first dataset is the [Phishing Websites Data Set](https://archive.ics.uci.edu/ml/datasets/Phishing+Websites) which can be found on the UCI Machine Learning Repository. 

It is an interesting dataset due to phishing website detection being a non-trival real-world problem, with phishing website authors in an arms race to escape notice from fraud detection algorithms. It was used by researchers successfully, so it should be high quality enough for our algorithms to classify well too, and with its large number of features it may illustrate differences between different algorithms. 

Due to the large number of data points in the Phishing dataset (over 10k), I randomly select 6k rows to keep our runtimes manageable. The data is balanced, with our subset having a **3327 : 2673 true/false count** (6000 total data points).

The second dataset I use is the [Wine Quality Data Set](https://archive.ics.uci.edu/ml/datasets/wine+quality) also found on the UCI Machine Learning Repository. 
	
It is interesting because a common debate about wine quality is whether it can be predicted by quantifiable attributes or is mostly subjective; there may be a natural limit to how well supervised learning can do as a result. 

I one hot encode this multi-class problem into a binary one conditioned on whether wine quality, as measured by the scale used in the dataset, is greater than 6. Our data is slightly imbalanced, with a **2604 : 1314 true/false count** (3918 data points).

### Methodology

I split each dataset into a training dataset (75%) and testing dataset (25%). I will apply the supervised learning algorithms on the training set then the test set. To pre-prune the models, I will use SciPy’s GridSearch with cross validation to do a brute force search for the best possible hyperparameters. I use a validation curve to manually validate the chosen parameter.  

Using these model parameters, I generate learning curves with varying input sample size, and generate a confusion matrix. In the final section, I compare each model’s timing, accuracy, and F1 score on the training and test data, highlighting advantages and disadvantages to each model.

### Decision Trees

For decision trees, I wish to prune the trees by merging leaf nodes according to certain criteria. These hyperparameters include criterion for choosing the best attribute at any given branch (gini or entropy), the max depth of our tree, and finally the minimum samples split (how many data points must be in the node in order to split it, otherwise it becomes a leaf). 

Our brute force GridSearchCV give us Branch Criterion, Max Depth, Min Samples Split = (Entropy, 15, 2) for Phishing Data and (Gini, 15, 75) for Wine Data.

We validate the max depth hyperparameter by using SciPy’s validation curve function with cross validation set to 5, meaning I divide the training data into five parts, train on four while testing on the last one and repeat for all 5 possible testing sets/training sets. This is done for each depth level with the resultant errors plotted. The vertical line indicates the optimal parameter from our grid search.

<img src="/img/s1.png" width="700px"/>
 
Our hyperparameters chosen seem decent looking at the validation curves. Error reaches a minimum for the cross validation set at the chosen max depth, while further increasing max depth does not decrease cross validation error, indicating I would be overfitting in those ranges. A constant error is expected at high max depths since the tree would be large enough to cover all data points in our ~3-4k training sets (with the tree able to encompass 2^(max_depth) nodes).

<img src="/img/s2.png" width="700px"/>

Using our chosen hyperparameters, I generate learning curves for each sample. For the Phishing data, I see the error is zero at sample size of 1 (as expected, since it is trivial to fit a decision tree to one point) and increases as sample size increases. Cross validation error starts decreasing around sample size 2000, but does not decrease further with greater sample size. The gap between training and cross validation is not high, being only ~5-10% off, indicating low variance. Bias is also fairly low since the magnitude of errors for both training and cross validation is low. 

In contrast, our wine dataset shows high variance; there is a large gap between the two curves indicates our model fits the training data better than the cross validation set. Bias is very low, with no error in all sample sizes tried. This is not unusual for decision trees, as it means our tree is large enough to encompass all data points without merged leaves. 

<img src="/img/s3.png" width="700px"/>

Overall, I see from the confusion matrices that the DT model does well on the phishing dataset with an F1 score of 0.953, but not as well on the wine dataset with an F1 score of 0.822. I chose to use an F1 score due to the wine dataset being slightly imbalanced, as the F1 score combines precision and recall to minimize how accuracy is misleading when applied to imbalanced datasets. Comparison and analysis of F1 scores and timing is presented in the last section of this report.

### AdaBoost 

For Adaboost I choose an ensemble of weak DT Learners that aggressively pre-prune (entropy, max depth 5, min samples split 100) with GridSearchCV giving us hyperparameters of (estimators = 50, learning rate = 0.5) and (200, 0.3) for the two datasets respectively. Once again we’ll analyze these parameters with a validation curve. 

<img src="/img/s4.png" width="700px"/>
 
We see from the validation curve that in both cases, cross validation error drops off as learning rate increases, and our chosen learning rates are in the beginning of the regime where cross validation error stops improving, which is where I want to be to avoid overfitting. 

<img src="/img/s5.png" width="700px"/>
 
Our learning curve for the phishing dataset again shows low variance and low bias. I see that cross validation error has not yet leveled off, indicating a larger sample size could improve the model. 
For the wine dataset training error is zero (as expected given Decision Tree results) and the cross validation error similarly has not levelled off. Bias is low and variance is high, similar to Decision Tree.

<img src="/img/s6.png" width="700px"/>

With F1 scores of 0.971 and 0.85 respectively, the Adaboost represents a significant improvement over a single Decision Tree for both datasets. This makes sense as an emsemble learner is more resistant to overfitting and produces better results generally than a single learner.

### k Nearest Neighbors (KNN)

KNN is a lazy instance based algorithm. Our grid search optimization gives us (n_neighbors = 15, weights = distance) and (50, distance) respectively.
	
<img src="/img/s7.png" width="700px"/>
 
For the phishing dataset, I see our choice of n_neighbors = 15 is at the minimum of cross validation error, making it a good choice. For the wine dataset, cross validation does not have as clear a drop as n_neighbor varies. It’s likely our wine dataset is suffering from the curse of dimensionality, as many of the wine attributes may not actually contribute to wine quality due to its subjective nature, which the KNN model is ill-equiped to handle.
	
<img src="/img/s8.png" width="700px"/>
 
For both datasets, I see cross validation scores decrease as sample size increases, suggesting a larger sample size would further improve the model in both cases. Given a large attribute space this makes sense as the data is sparsely populated in n-dimensional space, where n is the number of attributes. 

<img src="/img/s9.png" width="700px"/>
 
Both models performed reasonably well relative to the Decision Tree but were slightly worse than the Adaboost model, with F1 scores of 0.948 and 0.845 respectively. 

### Support Vector Machines (SVM)

The Support Vector Machine model classifies data by separating them in n-dimensional feature space with a hyperplane, using the kernel trick to make it computationally feasible. I use the default rbf kernel for which the main hyperparameters are C, the penalty parameter, and gamma, the kernel coefficient. Our GridSearchCV gives (C = 10, gamma = 0.1)  and (100, 0.04).

<img src="/img/s10.png" width="700px"/>

We validate our chosen hyperparameters with a validation curve. In both cases, the value of C is at the inflection point at which the cross validation error begins to level off, indicating I am neither underfitting nor overfitting.

<img src="/img/s11.png" width="700px"/>
 
In both cases the learning curve shows cross validation errors decreasing as sample size increases. Similar to previous algorithms, the phishing dataset shows low bias and variance, while the wine dataset shows high variance and very low bias. This is expected for the wine dataset due to our high value for C = 100. That means a high amount of regularization occurs while training the kernel, which tends to lead to high variance and low bias. With a larger sample size variance would decrease, which agrees which our learning curve.

<img src="/img/s12.png" width="700px"/>

SVM performed well for the phishing dataset with a F1 score of 0.961. However, for the wine dataset, SVM performed worse than all previous algorithms, with an F1 score of 0.798. The kernel is likely not extracting  information well from the wine dataset, which is fairly small and has attributes that may not have a lot of inter-correlated information (where support vector machines shine).

### Neural Networks

Feed-forward neural networks are powerful tools that can approximate any other continuous function, but come at the expense of long training times and difficult interpretability. I use a multi-layer perceptron network where I vary the hidden layer size and the maximum number of iterations. GridSearchCV gives (hidden layer = (105,10,2), max iterations = 1700) and ((105,12,2) and 2000).
	
<img src="/img/s13.png" width="700px"/>
   
We see that for the phishing dataset, cross validation error quickly falls as I increase max iterations but then levels off. For the wine dataset, cross validation error does not stabliize well, and cross-validation errors are very similar to training errors, suggesting the neural network is not converging on a good solution, possibly due to the low number of training samples.

<img src="/img/s14.png" width="700px"/>
 
For the phishing dataset, bias is low and variance is fairly low, with error decreasing with more samples. The wine dataset shows some interesting behavior: bias is high while  variance is low, indicating the neural network failed to find good neural parameters to fit the training data.
 
<img src="/img/s15.png" width="700px"/>

The confusion matrix shows the neural net performs fairly well on the Phishing dataset (F1 score of 0.968) but not particularly well on the Wine dataset (F1 score of 0.815). The poor performance on the wine dataset may in part be explained by the low number of data points. I saw that cross validation did not stabilize well in the validation curve, so the model is not able to converge on optimal perceptron parameters leading to increased model variance.

### Model Comparisons

The table below shows training and testing wall times for each model on each dataset, along with accuracy and F1 score. While accuracy was the main metric used in all learning/validation curves, I also chose to look at F1 scores here due to the wine dataset being slightly imbalanced. F1 scores combine precision and recall which is less biased by an imbalanced dataset. The F1 scores agree with the accuracy results, so I am not too concerned about this effect, as the wine dataset imbalance is not great (if our model always returned True, accuracy would be 66%).

<img src="/img/s16.png" width="800px"/>

**How do the models compare with training / testing speed and why?**

We first note that decision trees are extremely fast to build and to test. They have the lowest testing times of any model for both datasets, and are the second fastest to construct after KNN. KNN training time is hard to beat as the algorithm is lazy. The fast test time of Decision Tree can be explained by the relatively simple task of traversing a tree, which only takes log(n) comparisons where n is the height of the tree. 

Our multi-layer perceptron neural network was the slowest the train, which is expected given its iterative based approach to converging on a solution. Support vector machine is relatively faster than neural net but slower than KNN or DecisionTree, as the computationally demanding task of applying the kernel to all features is more intensive than the branch comparisons in building a Decision Tree. Decision tree training time was still moderate, as the algorithm is greedy and compares all possible features to decide the correct feature to split on our criterion. SVC time was comparable to AdaBoost’s ensemble of weak learners. 

The slowest test time is dominated by KNN and AdaBoost, which makes sense due to KNN processing its closest neighbors weighted by linear distance at the time of testing, and AdaBoost needing to traverse trees for each of its many estimators.

Interestingly, the wine dataset took more time to train and test than the phishing dataset for the AdaBoost model despite having less features and datapoints. This is largely explained by the higher learning rate hyperparameter chosen for the Phishing dataset. The reason for the lower learning rate in the wine dataset is likely that it has more datapoints that do not explain wine quality well, so AdaBoost is needs to repeatedly boost these poor datapoints. Meanwhile, the Phishing data has far fewer poor datapoints so it shrinks the contribution of each classifier more quickly and terminates early when it finds a perfect fit. These reasons agree with the general trend of poor accuracy in the wine dataset.

**Which models do the best and worst overall for each dataset and why?**

As expected, AdaBoost accuracy beats out a single Decision Tree by itself. The ensemble of Decision Trees leads to better results and is more robust against overfitting, but at the expense of significantly more training time. In fact the Adaboost model shows the highest accuracy for both datasets out of all models, even with the aggressive pruning of its weak learners.

SVC is comparable in training time and accuracy for the Phishing data to AdaBoost, but performs much more poorly for the Wine data. The model is sensitive to the kernel chosen and may not be optimal for the dataset. In contrast KNN performed well on the wine dataset but more poorly on the Phishing dataset. This may be due to the high number of features in the Phishing dataset, with possibly irrelevant features skewing the KNN feature space, a known weakness of the model. Meanwhile the wine dataset has a low number of features that may correlate well with each other in feature space. 

It was not shocking to see that Neural Net was not the most accurate for either dataset. Neural networks typically require massive datasets to shine, with the trade-off of significant training times. Our datasets are too small to reap these benefits. Nevertheless our MLP Neural Network performed amongst the best of the various models for the Phishing dataset, but was poor on the wine dataset. I saw no convergence in the validation curve for the wine dataset suggesting not enough datapoints.

**How do the characteristics of the datasets affect our results?**

It’s clear the two datasets are quite different; I easily reached ~95% accuracy with nearly all models on the phishing dataset while on the wine dataset I varied from doing barely better than chance (Neural Net model) to significantly better (Adaboost) with a 10 percentage point range. This difference can largely be explained by the inherent difficulty of classifying wine quality using the provided attributes. 

Wine quality shows weak correlations with many of the attributes, with a moderate correlation of around 0.5 only to a handful of attributes. This makes it easy to overfit on datapoints with little real info. I saw in KNN that varying K did not produce significantly different results, indicating the wine dataset has many similar instances of datasets; otherwise we’d expect high bias as k increases.  Adaboost did well with the wine dataset on the otherhand as it is resistant to overfitting due to the many weak learners it uses, and does better with fewer datapoints like I have in the wine dataset.

**What amount of parameter tuning was necessary for our best results?**

We did a great deal of hyperparameter tuning, first via grid search, then using a validation curve on each to verify (only the most significant validation curve was shown due to report space constraints). For KNN, minimal tuning was needed, as different values of k showed similar results, as well as linear versus equal weighting. The most difficult model to tune was the neural networks, which had long training run times, causing our grid search to take up significant time. In the end the validation curve showed poor convergence for the wine dataset as iterations increased, indicating overfitting. Results did not improve by decreasing the hidden node layer. 

The other models (Decision Tree, Adaboost, SVM) all required a fair amount of tuning, especially the extent of pruning in Decision Tree and the number of estimators in Adaboost. The parameters did cause significant changes in cross validation accuracy, and required us to find the right place between underfitting and overfitting.


