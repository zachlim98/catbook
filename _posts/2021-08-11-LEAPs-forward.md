---
title: "Deep ITM LEAPs as a safe form of leverage - A Simulation"
categories: [data, trading]
---

# Introduction

Two books that I recently read got me thinking about using LEAPs as a substitute for holding stock. While I do trade on the side, the bulk of my investments are in Index Funds and I'm a diehard Boglehead-kinda guy. However, [Lifecycle Investing](https://www.amazon.com/Lifecycle-Investing-Audacious-Performance-Retirement-ebook/dp/B003GYEGK2) (Ayres and Nalebuff, 2010) argues that even for passive investors who only buy into large index funds, their diversification is only in terms of *assets* but not in ***time***. It's an interesting premise and they essentially argue that younger investors should use *leverage* and then decrease that amount of leverage over time as one grows older. This allows for a diversification of risk over time (since it allows you to hold a constant proportional risk over time). They mentioned LEAPs as a way to do this and that got me into [In The Money](https://www.amazon.com/Money-Options-Strategy-Guaranteed-Market-ebook/dp/B08HJJDW2M) (Cullen, 2020) which advocated for the use of LEAPs as a stock substitute. I hence decided to test this out with some simulated forward testing.

In this article, I run a single-simulation, multiple simulations, and then explore the consequences of this strategy in a bear market. 

*note: I would highly recommend reading Lifecycle Investing if only for the perspective it brings. In The Money is good if you're completely new to options; otherwise it doesn't provide much else that you can't find on the internet.*

## What are LEAPs?

You can find a thousand definitions of this on the internet so I'll keep it simple and define LEAPs as any call option that is:

- Greater than 1 year in time
- Greater than 90 delta

Cullen advocates for DITM (deep in-the-money) LEAPs which she defines as any LEAP that has a strike price of **half** the current price of the stock. Since this makes simulation easier, we will be using DITM LEAPs. The primary value of DITM LEAPs is in its [low extrinsic value](https://www.quora.com/Why-do-deep-in-the-money-call-options-not-have-any-time-value-premium-Is-it-best-to-then-wait-to-exercise-at-expiration-versus-selling-early), which means that the time decay is almost negligible (i.e. you don't lose money from holding the option over time). 

## The Simulation

### Getting the data

As a passive investor, I like to passively hold SPY (an ETF mimicking the S&P500 - 500 biggest companies in the US Stock Market). I hence ran the simulation on SPY alone.

```python
# import libraries

import pandas as pd
import numpy as np
import mibian # for options 
import yfinance as yf # for stock data

#get the historical prices for SPY
tickerDf = yf.Ticker("SPY").history(period="max")
#calculate the daily percentage returns
tickerDf["returns"] = tickerDf["Close"].pct_change()
```

Now that I had the daily returns, I used a Geometric Brownian Motion model to simulate the daily stock price over the next 857 days (until December 15, 2023, the expiry date of the furthest out LEAP contract).

### Single-run Simulation

Given that the current price of the SPY was around 420 when I simulated this, I chose the DITM option with a strike price of 210 (half of the current price as per Cullen's definition).

```python
mu = np.mean(tickerDf["returns"])
sigma = np.std(tickerDf["returns"])
days_of = 857
start_price = 420

# simulating random daily returns for 857 days
returns = np.random.normal(loc=mu, scale=sigma, size=days_of)
price = start_price*(1+returns).cumprod()

# simulating DITM price
forward = [mibian.BS([price[-d], 210, 0.01, d], volatility=21.63).callPrice for d in range(days_of,0,-1)]
days = [d for d in range(0,days_of,1)]
```

I then created a dataframe from these lists and created some columns in order to record various metrics.

```python
# create dataframe
returns = pd.DataFrame({"Underlying Price": price, "LEAP Price": forward, "Days": days})

# create PL columns
returns["LEAP PL"], returns["BnH PL"] = (returns["LEAP Price"] - returns["LEAP Price"][0])*100, (returns["Underlying Price"] - returns["Underlying Price"][0])*100

# create PL (%) columns
returns["LEAP Returns(%)"] , returns["BnH Returns(%)"] = (returns["LEAP PL"]/(returns["LEAP Price"][0]*100))*100, (returns["BnH PL"]/(returns["Underlying Price"][0]*100))*100
```

Finally, I plotted it using plot.ly

```python
import plotly.express as px

fig = px.line(returns,x="Days", y=["LEAP Returns(%)", "BnH Returns(%)"], labels={"value":"Profit/Loss (%)", "variable":"Strategy"})
fig.show()
```

![newplot](https://user-images.githubusercontent.com/68678549/128971888-70a7335b-b7cb-4c2b-8552-83b033242bd6.png)

As you can see, given that it was essentially a 2x leverage, the % returns of the LEAP more or less doubled (both positively and negatively) the return of BnH - one important thing to keep in mind though, was that with the LEAP, you still had HALF of your entire portfolio leftover to buy other 'safer' investments like bonds or the like. 

As a check, I also graphed out the extrinsic value of the LEAP over time. 

![newplot2](https://user-images.githubusercontent.com/68678549/128971892-0e48677e-67b0-4901-9308-775c3e0ff678.png)

The extrinsic value at the beginning was almost negligible (~$72 for a \$20+k investment). As the simulated dipped, it did experience a rise in extrinsic value due to the changing of the delta. However, it eventually settled down again. Cullen and a number of others on the internet recommend rolling your LEAP out to the next expiration cycle when it reaches ~ 300 days left. This could make sense since the (extremely little) extrinsic value has not yet reached its point of acceleration. 

How would this strategy perform over a number of occurrences though? That was my next exploration.

### Multi-Run Simulation

```python
sim = []

for i in range(0,1000):
    price = start_price*(1+(np.random.normal(loc=mu, scale=sigma, size=days_of))).cumprod()
    LEAP_val = (mibian.BS([price[-1], 210, 0.01, 1], volatility=21.63).callPrice - mibian.BS([price[0], 210, 0.01, days_of], volatility=21.63).callPrice)/ mibian.BS([price[0], 210, 0.01, days_of], volatility=21.63).callPrice
    BnH_val = (price[-1] - price[0] )/ price[0]
    diff = LEAP_val - BnH_val
    sim.append(diff)
    
px.histogram(sim)
```

To do this, I simply created a list to store all the value differences between LEAP end values and Buy-and-Hold end values. 

![newplot3](https://user-images.githubusercontent.com/68678549/128971894-553d5f06-87cc-444d-a8a2-41f9aa0c278a.png)

We see that the bulk of the simulations are positive. This would make sense given the upward drift of the market, which would see the ITM LEAP outperforming Buy and Hold (since it's leveraged). What about the negative days though? One of the negative outcomes, for instance, looked something like this:

![newplot4](https://user-images.githubusercontent.com/68678549/128971897-6a107df1-4d75-4216-9d49-6b33fc303773.png)

## Rainy Days and Bear Markets

As seen in the figure above, let's assume that just as our option is about to expire, the market takes a nice big dump and crashes ~30%. This puts us in the red at about ~60% and we need to re-open a new ITM option. There are two possibilities here - (a) we open it at the same strike so as to save on costs or (b) we re-open it at the new 1/2 of the current price ITM option. The crucial thing here is that we would need to fork out **additional amounts of capital** since the new DITM option would be more expensive than our currently expiring not as ITM option. Let's explore both. 

 #### Scenario A

In scenario A, we open the new ITM option at the same strike price. In this case, our original strike price was $210 and so we maintain the same strike. Given that we have to add additional capital to buy the new strike, I add this to our initial starting capital, which hence skews the % drops before the new LEAP was opened (this is more obvious in Scenario B)

```python
# Additional amount of capital to be used
add_payment_3 = bear_day_3["LEAP Price"].iloc[0] - bear_day["LEAP Price"].iloc[-1]

# Simulation of HTS at STK Price 210
buy_LEAP_HTS = [mibian.BS([bear_day_2["Underlying Price"].iloc[-d], 210, 0.01, d], volatility=21.63).callPrice for d in range(days_of,0,-1)]
days = [d for d in range(0,days_of,1)]

fig = px.line(bear_complete_2,x="Days", y=["LEAP Returns(%)", "BnH Returns(%)"], labels={"value":"Profit/Loss (%)", "variable":"Strategy"})
fig.show()
```



![newplot6](https://user-images.githubusercontent.com/68678549/128971904-71b330a0-c539-42f8-a23f-82ded5e4bde9.png)

The results are not great. As seen, the HTS LEAP severely underperforms Buy and Hold. This is due to the higher extrinsic value of the option, given that it is not as ITM. Graphing it out, we see the drag that the extrinsic value has on the option as it approaches expiration. Compared to our DITM $77 extrinsic value previously, this option begins with more than \$750+ of extrinsic value. 

![newplot7](https://user-images.githubusercontent.com/68678549/128974352-4df01a69-5262-47c3-bd47-aed609e744a2.png)

It is also affected by the lower delta compared to a more ITM option. In comparing it to the option we use in Scenario B, we see that the delta is initially lower, which hurts the option value even as the SPY recovers.

![newplot8](https://user-images.githubusercontent.com/68678549/128974356-a1158ff9-fc86-4606-9bdd-ce2f56b655ef.png)

#### Scenario B

In scenario B, we re-open the option at 145, about 1/2 the current price of the SPY. Given that we are opening it at a lower strike, the amount of additional capital we have to fork out is significant. I add this to the initial starting capital which softens the returns since it decreases the amount of leverage we are using. For example, the initial fall is no longer 60% but around 45%. This also means, however, that every increase is similarly dampened. 

![newplot5](https://user-images.githubusercontent.com/68678549/128974346-b7ec305b-a9d3-4a62-bb5d-84fbdf622649.png)

The results are much better than Scenario A and essentially matches Buy and Hold. However, the leverage is decreased because of the additional capital that we have plowed into buying the next roll. The time period of around 1700 days for recovery is not impossible - after the 2008 financial crisis, the SPY took around 1900 days to recover to its peak in 2007. 

## Conclusion

In this article, we explored the use of DITM LEAP options as a form of leverage to augment Buy and Hold returns. These options generally 2x returns during bull markets and were relatively 'safe' replacements in sustained bear markets, provided one had additional capital to plow back into the market. 

Three important observations come to mind when considering the results of this. First, one important difference that was not considered is the presence of additional capital. The freed up capital can augment returns since this capital can be deployed into other assets that may act as a hedge during market downturns. This could be a potential avenue for future articles to explore. Second, if one had additional capital during a market downturn, one could 'buy the dip' and add on additional LEAPs for relatively lower prices. This could boost the leveraged amount, which would capitalise on the market recovery. Third, and most importantly, one could **roll-up** the LEAP every year to lock in profits during a bull-run. This would allow one to take profits safely, which could then be used to buy additional LEAPs or for other assets.