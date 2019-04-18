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
import util
import math
import pandas as pd
import matplotlib.pyplot as plt
import numpy as np

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

Now that I have explained the idea and code behind these three technical indicators, I want to try to use them to predict future prices. I will do so by using an ensemble learner called a Random Forest. 

>>>>>**WORK IN PROGRESS**

#### Random Tree 

#### Random Forest 

#### Simulating a Market

For the purposes of this post, we'll assume we can take at most a long position of 1000 shares or a short position of -1000 shares on any given day.

#### Putting it all together: Training and Testing our Learner

### Can we beat the market with technical analysis?

The short answer is almost certainly no. 

The long answer is that maybe you can beat the market, for brief periods of time, if you have teams of PhDs working on it full time. Renaissance Technologies is considered one of the best hedge funds that employs technical strategies, and does exactly that. But you or I, trading casually in our free time, are almost certain to lose against the market.

To understand why it is difficult or potentially impossible to beat the market, I will first explain the ideas behind the Efficient Market Hypothesis and then explain some of the human cognitive biases which many retail traders fall prey to.

#### Efficient Markets

#### Cognitive Biases and Trading
