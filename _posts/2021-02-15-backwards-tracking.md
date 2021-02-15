---
title: Calculating Delta and Theta for Past Trades
categories: [trading]
---

# Introduction

(*note: This is going to be a very very niche subject but it might be helpful for some*)

So, I've been tracking my options trades but haven't really been the best at tracking ALL the different Greeks. In order to do it, I decided to use the `mibian` library on Python to go back and manually calculate **delta** and **theta** which were the most important option Greeks for me. This is not really a tutorial post per se but more of an archival post for me. 

## Getting Data

I'd already been tracking my trades but had only recorded the strike prices and credit received for each trades. This is how it looked like: 

![image](https://user-images.githubusercontent.com/68678549/107906811-0af78e80-6f8d-11eb-9e72-691ed09e301b.png)

Thankfully, this was all I needed in order to backwards calculate the Greeks. I was missing one important element - the price of the underlying at the time of purchase (which I have now decided that I will track) and so I used `yfinance` in order to retrieve those historical prices. 

```python
# import a bunch of libraries
import pandas as pd 
import plotly.express as px
import mibian as mb
import yfinance
from datetime import datetime

raw_options = pd.read_csv("rawtrades.csv") #import data
raw_options["Underlying"] = [yfinance.Ticker(i).history(period="6mo").loc[d][["High", "Low"]].mean() for i,d in zip(ticker_names,ticker_dates)] #get mean price of underlying
```

With the data done, all that was left to do was to set up a *mean* list comprehension and get my info. 

## Calculating Implied Volatility

The issue with `mibian` was that it required you to give Implied Volatility in order to calculate the Greeks. However, I only had the option prices. Thankfully, I could use the prices to backward calculate the IVs of each trade. 

```python
vol = [mb.BS([i,j,1,k], callPrice=l).impliedVolatility if t=="C" else mb.BS([i,j,1,k], putPrice=l).impliedVolatility for i,j,k,l,t in zip(raw_options["Underlying"], raw_options["Strike"], raw_options["DTE"], (raw_options["Cost"]/100), raw_options["Type"])] # get implied volaility for greek calculations
```

Here, I used the `mb.BS` which was a function that used mibian's Black-Scholes option price model. Throwing it all into a list comprehension, I now had my list of calculated implied volatilities. 

## Calculating the Greeks

And that made it easy for me to calculate Delta and Theta. The Thetas were all calculated as negative. However, most of my contracts were sold to open which meant that I was collecting positive Theta. I hence had to adjust the Theta to account for that. 

```python
theta = [mb.BS([i,j,1,k], volatility=l).callTheta if t=="C" else mb.BS([i,j,1,k], volatility=l).putTheta for i,j,k,l,t in zip(raw_options["Underlying"], raw_options["Strike"], raw_options["DTE"], raw_options["ImpliedVol"], raw_options["Type"])] #calculate theta from Implied Volatility

# adjust Theta for Selling To Open
adjThe = [th if a=="BTO" else -th for th,a in zip(theta,raw_options["Action"])]

#adding Greeks to my dataframe
raw_options["Theta"] = adjThe
raw_options["Delta"] = delta
```

## Summing and Plotting

The values had been calculated for each individual option. However, most of the trades that I had opened were spreads. Hence, I needed to group them according to Trade ID in order to see the overall Delta or Theta of the trade.

```python
summed = raw_options.groupby(by="Trade ID").sum() #group by ID and then sum
summed["PL"] = summed["Credit"] + summed["Debit"] #calculate price loss by adding credits and debits
summed = summed[summed["PL"] > -200] #filter in order to remove trades that are still open

px.scatter(summed, x="Theta", y="Delta") # plot scatter graph of theta and delta
```

![image](https://user-images.githubusercontent.com/68678549/107907299-46df2380-6f8e-11eb-8e7c-15ffd916f407.png)

## Conclusion

And... that's about it! The true lesson I learnt from this experience is the importance of **TRACKING. YOUR. TRADES.** and all the essential metrics like Implied Volatility, Underlying Price, Delta, Theta, and other Greeks if you wish to do so. This will make data analysis later MUCH easier. Thanks for reading!