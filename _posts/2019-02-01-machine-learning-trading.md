---
layout: post
title: Machine Learning and Markets
subtitle: Full Tutorial on Trading using an Ensemble Learner in Python
tags: [machine learning, finance, python, supervised learning, random forest]
image: /img/3.png
---

In this blog post, I go over some basic technical indicators to use on market data, show how to implement an ensemble learner in Python to predict future prices, and explain why it's nearly impossible to beat the market.

### Dataset

For this blog post, I am going to work with publically available financial data for JPM (JP Morgan) from the time period **1/1/08 to 12/31/11**. Specifally I am going to use end of day prices for each trading day. This data can be found from Yahoo Finance or any other freely available financial data provider. For your convenience I have placed the dataset [here](/data/JPM.csv).

I divide the dataset in half. I will use **1/1/08-12/31/09** as my training set (in-sample), and **1/1/10-12/31/11** as my testing set (out-of-sample). If I didn't do this, any strategy I use will overfit the data used, making it useless on additional data.

### Technical Indicators

Technical analysis is a strategy where we use patterns in past prices to try and predict future prices. At the end of this post, I'll go over some reasons why this is near impossible to do in reality. But for educational purposes, I will begin by looking at a few basic technical indicators and see how effective they are in on the in-sample and out-of-sample datasets.

To begin, I load required Python packages and read in the end of day stock data using the *pandas* DataFrame structure, subsetted for the date period of interest.

```python
import math
import pandas as pd
import matplotlib.pyplot as plt
import numpy as np
import random

df = pd.DataFrame(index=pd.date_range("January 1, 2008", "Dec 31, 2011"))
df_temp = pd.read_csv('JPM.csv', index_col='Date', parse_dates=True, usecols=['Date', colname], na_values=['nan'])
df_temp = df_temp.rename(columns={colname: 'JPM'})
df = df.join(df_temp)  		   	  			    		  		  		    	 		 		   		 		  
data = pd.DataFrame(data=df)
```

#### Simple Moving Average (SMA)

The simple moving average is the average adjusted close price for the last x days, or as many days as we can if there are less than x days of previous price history. In our case, we look at the last 10 days (we call this SMA10). The actual indicator is this simple moving average divided by the price on any given day.

The basis of this indicator is that when SMA10/price rises above 1, it means the current price has fallen below the average, so it seems cheap. Likewise, when the ratio falls below 1, the stock seems expensive. If the price tends to mean revert, then we can make profit by buying when the ratio rises above 1 and selling when it falls below 1.

```python
def sma_indicator(prices_sym, lookback = 10):
    moving_sum = 0
    sma = []
    bb_value = []
    for i in range(0, len(prices_sym)):
        moving_sum += prices_sym[i]
        if i >= lookback:
            moving_sum -= prices_sym[i - lookback]
            sma_cur = moving_sum / lookback
            std_dev = (prices_sym[(i - lookback + 1):i + 1]).std()
        else:
            sma_cur = moving_sum / (i + 1)
            std_dev = (prices_sym[0:i + 1]).std()
        if i == 0:
            bb_value.append(0)
        else:
            bb_value.append(abs((prices_sym[i] - sma_cur) / (2 * std_dev)))
        sma.append(sma_cur)


    norm_factor = prices_sym[0]
    prices_sym =  prices_sym / norm_factor
    sma = sma / norm_factor
    indicator1 = sma / prices_sym

    return indicator1, sma, sma + (bb_value / norm_factor), sma - (bb_value / norm_factor), prices_sym
```

I run this indicator on the first two months of our training set to illustrate what it looks like. This is done by calculating the indicator using the sma_indicator function I wrote above, then graphing it using the *matlibplot* library.

```python
#Simple moving average with 10 day lookback.
    # #If sma10/price rises above 1, price has fallen below the moving average; we buy because we think its cheap
    data['Price / SMA_10'], data['SMA_10'], data['BB_High'], data['BB_Low'], data['JPM_normalized']  = sma_indicator(data['JPM'])


    #Graph it!
    ax = data.plot(y = ['JPM_normalized', 'SMA_10', 'Price / SMA_10'], title='JPM: Normalized SMA 10 Day Lookback, BB in Shaded Gray', fontsize=12)
    ax.set_xlabel('Date')
    ax.set_ylabel('Normalized Stock Price')
    x_axis = data.index.get_level_values(0)
    ax.fill_between(x_axis, data['BB_High'], data['BB_Low'], color='grey')
    plt.show()
```

<img src="/img/t1.png" width="700px"/>

#### Linear Momentum

For any given day, look back exactly x days and divide the current price by that previous price. Then, subtract 1 to normalize around zero. We use x = 3 days.

The basis of this indicator is that stock prices follow trends; if it rises, the price will continue to rise and likewise for falling. When this indicator is above 0, the price is on an upswing, so we buy. If it is below 0, the price is falling, so we sell.

```python
def lin_momentum_indicator(prices_sym, N = 3):
    momentum_percent = [0] * N

    for i in range(N, len(prices_sym)):
        momentum_percent.append((prices_sym[i] / prices_sym[i - N]) - 1)

    indicator2 = momentum_percent
    return indicator2
```

Once again, I plot this indicator on the first two months of data to illustrate what it looks like, normalized.

```python
#Simple momentum: linear. Buy when crosses above zero, sell when crosses below zero. Min wait time to lower fees.
    data['Momentum'] = lin_momentum_indicator(data['JPM'])

    #Graph it!
    ax = data.plot(y = ['JPM', 'Momentum'], title='JPM: Normalized, Simple Linear Momentum 3 Day Lookback', fontsize=12)
    ax.set_xlabel('Date')
    ax.set_ylabel('Normalized Stock Price')
    x_axis = data.index.get_level_values(0)
    ax.axhline(y=0, color='k')
    plt.show()
```

<img src="/img/t2.png" width="700px"/>

#### Donchian Channels

The final technical indicator I describe will be Donchian channels.

For any given day, look back x days, or as many days as we can if there are less than x days of previous price history. During that time period, find the max price and min price. These are the bands of the Donchian channel. The actual signal is as follows: 1.1 if the price equals the max_price, 0.9 if the price equals the min_price, 1 otherwise. These values were chosen to allow the signal to be graphed alongwisde the normalized price. We use a 5 day lookback.

The basis of this strategy is that the price tends to mean revert and stays within the Donchian channel. If our current price is the min price (i.e. the donchian signal is below 1), the stock seems cheap and we buy. If our current price is the max price (donchian signal above 1), we sell.

```python
def donchian_indicator(prices_sym, lookback = 5):
    max_donchian = []
    min_donchian = []

    for i in range(0, len(prices_sym)):
        if i >= lookback:
            data_donchian = prices_sym[(i - lookback + 1):i + 1]
        else:
            data_donchian = prices_sym[0:i + 1]
        max_donchian.append(data_donchian.max())
        min_donchian.append(data_donchian.min())

    indicator3 = 1 + 0.1 * (
            (prices_sym == max_donchian).astype(int) - (prices_sym == min_donchian).astype(int))
    return indicator3, max_donchian, min_donchian
```

Once again I show this indicator on the first two months of data. The actual indicator as well as the Donchian channels are graphed.

```python
    #Donchian channel. Buy when it hits a minimum, sell when it hits a maximum.
    data['donchian_signal'], max_donchian, min_donchian = donchian_indicator(data['JPM'])

    #Graph it!
    ax = data.plot(y = ['JPM', 'donchian_signal'], title='JPM: Normalized, Donchian Channel 3 Day Lookback', fontsize=12)
    ax.set_xlabel('Date')
    ax.set_ylabel('Stock Price')
    x_axis = data.index.get_level_values(0)
    ax.fill_between(x_axis, max_donchian, min_donchian, color='grey')
    plt.show()
```

<img src="/img/t3.png" width="700px"/>

### Predicting the Future

Now that I have explained the idea and code behind these three technical indicators, I want to try to use them to predict future prices. I will do so by using an ensemble learner called a Random Forest, which consists of many weak learners (in this case Random Trees). By putting these together using bootstrap aggregation, we reduce variance without increasing bias like we normally would due to the variance-bias tradeoff.

Let's start by looking at the individual weak learner, the random tree.

#### Random Tree 

```python
class RTLearner(object):
    def __init__(self, leaf_size = 1, verbose = False):
        self.tree = []
        self.leaf_size = leaf_size

    def buildTree(self, dataX_sub, dataY_sub):
        if len(dataX_sub) != len(dataY_sub):
            print "Error: x and y lens do not match"
        if len(dataX_sub) <= self.leaf_size or len(set(dataY_sub)) == 1:
            return [['leaf', np.mean(dataY_sub), None, None]]
        best_feature = np.random.randint(len(dataX_sub[0]))
        dataX_best = dataX_sub[:, best_feature]
        split_value = np.median(dataX_best)
        idx_less = dataX_best <= split_value
        idx_more = idx_less == False
        if sum(idx_less) == 0 or sum(idx_more) == 0:
            return [['leaf', np.mean(dataY_sub), None, None]]
        left_tree = self.buildTree(dataX_sub[idx_less], dataY_sub[idx_less])
        right_tree = self.buildTree(dataX_sub[idx_more], dataY_sub[idx_more])
        root = [[best_feature, split_value, 1, len(left_tree) + 1]]
        root.extend(left_tree)
        root.extend(right_tree)
        return root

    def traverseTree(self, dataX_point, idx_cur = 0):
        branch_cur = self.tree[idx_cur]
        if branch_cur[0] == 'leaf':
            return branch_cur[1]
        if dataX_point[branch_cur[0]] <= branch_cur[1]:
            return self.traverseTree(dataX_point, idx_cur + branch_cur[2])
        else:
            return self.traverseTree(dataX_point, idx_cur + branch_cur[3])

    def addEvidence(self,dataX,dataY):
        self.tree = self.buildTree(dataX, dataY)

    def query(self,points):
        return np.array([self.traverseTree(x) for x in points])
```

A Random Tree learner is similar to the Decision Tree learner I described in my past Supervised Learning post, except instead of using a particular criteria to decide which feature to split values on, the Random Tree simply chooses one randomly. 

The code above shows my implementation of the Random Tree learner. I include a leaf size, below which all values are aggregated together using the mean value.

#### Random Forest 

```python
class BagLearner(object):
    def __init__(self, learner, kwargs, bags):
        self.learner = learner
        self.kwargs = kwargs
        self.bags = bags

    def sampleData(self,dataX, dataY):
        n = len(dataX)
        idx_samples = np.random.randint(low = 0, high = n, size = n)
        return dataX[idx_samples], dataY[idx_samples]

    def addEvidence(self,dataX,dataY):
        #Creates ensemble with (bags) number of learners
        self.ensemble = []
        for x in range(0, self.bags):
            dataX_sample, dataY_sample = self.sampleData(dataX, dataY)
            learner_instance = self.learner(**self.kwargs)
            learner_instance.addEvidence(dataX_sample, dataY_sample)
            self.ensemble.append(learner_instance)

    def query(self,points):
        #Get results from every learner in the ensemble and average them
        y_estimate_all = np.array([x.query(points) for x in self.ensemble])
        y_estimate = np.mean(y_estimate_all, axis = 0)
        return y_estimate
```

Now we can use bootstrap aggregation to create our Random Forest. The advantge of this method is the ensemble of Random Trees is more robust against overfitting, but at the expense of more training time. 


#### Putting it all together: Training and Testing our Learner

```python
class StrategyLearner(object):
		  		 			     			  	   		   	  			  	
    def __init__(self, verbose=False, impact=0.0):
        self.verbose = verbose
        self.impact = impact
        self.bl = BL.BagLearner(RT.RTLearner, {'leaf_size': 5, 'verbose': verbose}, 10, False, verbose)

    def classifyY(self, prices, n_lookforward = 3):
        changes = []
        for i in range(0, len(prices)):
            if i + n_lookforward >= len(prices):
                changes.append(0)
            else:
                change_percent = 100 * (prices[i + n_lookforward] - prices[i] ) / prices[i]
                changes.append(change_percent)
        y = []
        y_buy = np.percentile(changes, 75)
        y_sell = np.percentile(changes, 25)
        for i in range(0, len(prices)):
                if changes[i] > y_buy:
                    y.append(1)
                elif changes[i] < y_sell:
                    y.append(-1)
                else:
                    y.append(0)
        return np.array(y)

    # this method should create a BagLearner, and train it for trading
    def addEvidence(self, prices, symbol):

        y = self.classifyY(prices[symbol], n_lookforward = 3)
        x1, _sma, _bb_high, _bb_low, _price_norm = sma_indicator(prices[symbol])
        x2 = lin_momentum_indicator(prices[symbol])
        x3, _max_donchian, _min_donchian = donchian_indicator(prices[symbol])
        X = np.array([x1, x2, x3])
        X = np.transpose(X)
        self.bl.addEvidence(X, y)

    # this method should use the existing policy and test it against new data 			  		 			     			  	   		   	  			  	
    def testPolicy(self, prices, symbol):
        x1, _sma, _bb_high, _bb_low, _price_norm = sma_indicator(prices_all[symbol])
        x2 = lin_momentum_indicator(prices_all[symbol])
        x3, _max_donchian, _min_donchian = donchian_indicator(prices_all[symbol])
        X = np.array([x1, x2, x3])
        X = np.transpose(X)
        y_estimate = self.bl.query(X)

        actions = []
        pos = 0
        actions.append(0)
        y_buy = np.percentile(y_estimate, 75)
        y_sell = np.percentile(y_estimate, 25)
        for i in range(1, len(prices_all[symbol])):
            sig = 'Hold'
            
            if y_estimate[i] > y_buy:
                sig = 'Buy'
            elif y_estimate[i] < y_sell:
                sig = 'Sell'

            if sig == 'Buy':
                actions.append(1000 - pos)
                pos = 1000
            elif sig == 'Sell':
                actions.append(-1000 - pos)
                pos = -1000
            else:
                actions.append(0)

        df_trades = pd.DataFrame(index=prices.index, data=actions)
        return df_trades
```

Finally, we create our strategy learner. This learner uses the Random Forest ensemble learner I previously described. It then uses the technical signals as its input features and the percent by which the stock went up or down as its label.

The strategy learner can now take any new list of stock prices over any arbitrary time period and predict when the optimal time to buy or sell is. I trained the learner on our in-sample period, as shown below. Alongside, I plotted the benchmark (holding JPM stock over the time period without any trading) as well as a manual strategy I created on my own, without the fine tuning of machine learning.

<img src="/img/t6.png" width="700px"/>

As you can see, our strategy learner performs very well in the in-sample period, beating both my manual strategy and the benchmark considerably. This is to be expected as we trained the model using this dataset. Now, we run it on the out-of-sample period.


<img src="/img/t7.png" width="700px"/>

The Random Forest learner does surprisingly well even in the out of sample period. Meanwhile, my manual strategy is unable to beat the benchmark. This shows the signals we used did not significantly degrade in the out-of-sample period.

### Remarks

I hope this demonstration helps you understand how you might use machine learning to predict prices in the stock market. 

However, keep in mind this is an educational exercise. I would caution against trading with real money using these techniques.

In general, it is difficult if not impossible to beat the market. You could probably can beat the market, for brief periods of time, if you have teams of very smart people working on it full time. For example, famous hedge funds like Renaissance Technologies employ technical strategies, and do exactly that. But a typical retail investor stands little chance against more sophisticated investors, and our cognitive biases make us over-confident in our ability to find patterns in noise. In fact, if the efficient market hypothesis is true (which nobody knows for sure), it is impossible to predict future prices with technical analysis.

Even if the efficient market hypothesis is not true and one were to miraculously find a real signal, a retail investor has to contend with unfairly high trading commissions, slippage, and execution costs that would wipe out any potential positive expected value. Trading with technical analysis is a zero sum game and the odds are rigged against you!