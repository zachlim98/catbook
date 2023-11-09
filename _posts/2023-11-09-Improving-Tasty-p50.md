---
title: "Improving TastyTrade's P50 metric - with Stop Losses and Time Limits!"
categories: [data, trading]
---

# Introduction
I've [written previously](https://zachlim98.github.io/me/2021-02/tastyworks-p50) about calculating the P50 metric that TastyTrade (TT) uses in their platform. As explained in that post, the core idea is running a Monte Carlo simulation and then calculating option pricing based off of those price paths. There are 2 key assumptions, however, that this uses:


*   You hold the option to expiry: This does not take into account the "managing at 21 DTE" portion that is so key to the TT mechanic. Therefore, this P50 metric is inaccurate because it does not reflect the potential loss that might be incurred by closing at 21 DTE, only for the stock to recover _after_ that.
*   You do not use a Stop Loss (SL): Yes, yes - I know that there are differing sentiments on using a SL for options. But if one were to employ them for naked strategies (e.g. Strangles or Straddles), this P50 calculation also does not take into account probability of hitting your SL before hitting 50% take profit.

These issues hence drove me to try to re-do my previous P50 calculation post, but with the ability to tweak the aforementioned assumptions

## Overview

Unlike my previous post where I used R, I'm going to be lazy and just use Python + libraries (so I don't have to re-write the Black-Scholes formula). Here's a general overview:

1. We will use the CAPM to get expected returns ($\mu$) and just regular standard deviation (with one degree of freedom since it's a sample) for volatility.
2. As before, we will simulate prices using [Geometric Brownian Motion](https://ro.uow.edu.au/cgi/viewcontent.cgi?article=1705&context=aabfj).
3. Write functions using `mibian` to calculate strangle pricing and run it through the price path, using IV% from market chameleon (or your brokerage platform of choice)

## Step 1: Calculating returns


```python
# install required libraries and import them
! pip install pyportfolioopt yfinance mibian

import pandas as pd
import numpy as np
from pypfopt.expected_returns import capm_return, mean_historical_return
import yfinance as yf
import mibian

SPY_data = yf.Ticker("SPY").history(period="3Y")["Close"]
EBay_data = yf.Ticker("EBAY").history(period="3Y")["Close"]

capm_return = capm_return(pd.DataFrame(EBay_data), pd.DataFrame(SPY_data), risk_free_rate = 0.0467)["Close"]
std = pd.DataFrame(EBay_data)["Close"].pct_change().dropna(how="all").std(ddof=1)
```

Here, we use the `capm_return` function from pyfopt to calculate the returns. We pull data for SPY and EBAY (our example underlying today) using yfinance. Why 3 years of historical data? To be honest there's no hard and fast rule but I felt 3 years was a good size. You could go back further if you wanted.

Note also that we used `ddof=1` but due to the large sample size, this did not ultimately affect the final calculation that much. For `risk_free_rate`, I simply googled what the 10-year treasury rate was and used that as a proxy for it.

## Step 2: Simulating Stock Price

Here, we write a simple function that simulates the stock price based on the above metrics.


```python
def simulate_stock_price(S0, mu, sigma, T, num_sim, dt=1):
    """Simulates stock price paths using Geometric Brownian Motion.

    Parameters:
    S0 : float
        Initial stock price
    mu : float
        Expected return
    sigma : float
        Volatility of the stock
    T : int
        Time to expiration in days
    num_sim: int
        How many simulations to run
    dt : int (optional)
        Time step (defaults to 1 day at a time)

    Returns:
    numpy.ndarray
        Simulated stock price paths
    """
    num_steps = T
    # The number of simulations
    num_simulations = num_sim
    # Simulating the stock price paths
    stock_price = np.zeros((num_steps, num_simulations))
    stock_price[0] = S0
    for t in range(1, num_steps):
        z = np.random.standard_normal(num_simulations)  # Random shocks
        stock_price[t] = stock_price[t - 1] * np.exp((mu - 0.5 * sigma**2) * dt + sigma * np.sqrt(dt) * z)
    return stock_price

```

Essentially, we create an empty array and then apply the GBM formula to run it through the number of DTE that our option has, simulating `num_sim` price paths for the stock. An example below, showing how it looks like. Note that we have to divide capm_returns by 252 as the returns from the formula above is an _anualized_ return. Hence, we divide it by the number of trading days in the year to get the (estimated) daily rate of return.


```python
arr = simulate_stock_price(38.50, capm_return/252, std, 40, 1000)
pd.DataFrame(arr).plot()
```


    
![Options_Pricing_5_0](https://github.com/zachlim98/historical-to-delta/assets/68678549/a20f839a-dc55-4e7c-a2e5-c98b617da51c)
    


## Step 3: Simulating Strangle Profitability


```python
import numpy as np
import mibian

def option_price(stock_price, call_strike, put_strike, int_rate, days_to_expiration, ivr):
    """Calculate the combined price of a strangle for a given stock price."""
    call = mibian.BS([stock_price, call_strike, int_rate, days_to_expiration], volatility=ivr)
    put = mibian.BS([stock_price, put_strike, int_rate, days_to_expiration], volatility=ivr)
    return call.callPrice + put.putPrice

def evaluate_strangle_profit_probability(S0, call_strike, put_strike, call_credit, put_credit, int_rate, T, mu, sigma, num_sim, ivr, time_limit):
    """Simulates strangle profitability using Black Scholes model.

    Parameters:
    S0 : float
        Initial stock price
    call_strike, put_strike, call_credit, put_credit : floats
        Costs basis and strikes for each leg of the strangle
    int_rate : float
        Risk-free rate
    T : int
        Time to expiration in days
    mu : float
        Expected return
    sigma : float
        Volatility of the stock
    num_sim: int
        How many simulations to run
    ivr : float
        IV% of the underlying
    time_limit:
        How many days the trade is to be held

    Returns:
    Profitability_of_profit
        Float representing POP of TP before SL
    Days_in_trade
        Float representing average days in trade
    """

    initial_credit = call_credit + put_credit
    target_profit = 0.5 * initial_credit
    stop_loss = 2 * initial_credit

    simulated_stock_prices = simulate_stock_price(S0, mu, sigma, T, num_sim)

    profit_hits = 0

    days_in_trade = []

    if time_limit > T:
      raise ValueError("Time Limit must be less than DTE of option")

    for path in simulated_stock_prices.T:
        for day, stock_price in enumerate(path[0:time_limit]):
            remaining_days = T - day
            strangle_value = option_price(stock_price, call_strike, put_strike, int_rate, remaining_days, ivr)

            # Check for 50% profit or 100% loss
            if strangle_value <= target_profit:
                profit_hits += 1
                days_in_trade.append(T - remaining_days)
                break
            elif strangle_value >= stop_loss:
                days_in_trade.append(T- remaining_days)
                break

    probability_of_profit = profit_hits / len(simulated_stock_prices.T)
    return probability_of_profit, np.round(np.mean(days_in_trade), 1)

```

The code for these functions are rather self-explanatory - essentially we write a function using mibian that calculates the price of the put and call separately and then adds them up to get the strangle value.

We then define what the stop loss and take profit are. I used a 2:1 ratio for this example but you could define it however you want. Finally, we run the price paths through and calculate the option value at each price point.

As mentioned in the introduction, this function allows for customisation of (1) The TP and SL and (2) The time in trade (`time_limit`) which hence allows you to simulate and account for the "managing at 21 DTE" thing if you so desire.


```python
evaluate_strangle_profit_probability(38.50, 45, 35, 0.30, 0.71, 0.0467, 40, capm_return/252, std, 100, 36.7, 40)

# (0.75, 13.3)
```


## Conclusion

So there you go - a little improvement over TT's P50 calculation, allowing for more customization. Drop me a DM if you have any questions/ideas!

Also, a quick aside - if you're curious about how long it takes to run this simulation - it is quite time/resource intensive because of how it has to manually loop through each iteration to see if it hit the SL or TP.


```python
%%timeit
evaluate_strangle_profit_probability(38.50, 45, 35, 0.30, 0.71, 0.0467, 40, capm_return/252, std, 1000, 36.7, 40)

# 55.3 s ± 839 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```