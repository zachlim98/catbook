# Introduction

So, having decided that I was going to use Options as my Investment Company-beating strategy, I needed to delve deeply into Options. I turned to [tastytrade](https://www.tastytrade.com) for some instruction and inspiration. *Side note*: This post likely won't be as interesting/data-related but will mostly encapsulate what I intend to be my trade strategy and show the payoff diagrams for the relevant strategies. 

The two main strategies that I intend to deploy are:

1.  Iron Condors 
2. Cash-Secured Puts (with the Wheel)

I separated my trading into two accounts - one in tastyworks and one in TDAmeritrade Singapore and intended to run Iron Condors on my tastyworks account (due to lower commissions) and CSPs on TDAmeritrade (due to the possibility of getting assigned and hence the lower costs of stock buying and selling under TDA). 

## Iron Condors

```python
IC = IC_long_call + IC_short_call + IC_long_put + IC_short_put

fig, ax = plt.subplots()
ax.spines['top'].set_position('center')
ax.set(xlim=[250,400],ylim=[-25,25])
ax.plot(sT,IC,label='Iron Condor')
plt.xlabel('Stock Price')
plt.ylabel('Profit and loss')
plt.legend()
plt.grid()
plt.show()
```

![image](https://user-images.githubusercontent.com/68678549/96592344-e101c880-131a-11eb-8dfd-e13cd81000be.png)

So, an iron condor (for the uninitiated) is basically an option selling strategy, with a neutral assumption. It's a combination of a call and put spread and you make money if the stock stays within the strike prices (in this case if the stock price stays between 300 and 350). These are my entry and management rules:

### Entry: 

1. DTE: 30 - 45 days
2. Collection: 1/3 of strike-width or 1/2 of strike width with more aggressive management 
3. Strike selection: Typically sell 30D or 1SD/2SD. May sell wider if doing earnings plays (but **rare**)
4. IV Rank: Above 30, Optimally Above 50

**Management:**

1. Management at either 50% profits or 21 DTE
2. Will roll up spreads on untested side after 21 Days 
3. Largely little management due to it being risk-defined from the get go 

**Universe:**

1. Mostly ETFs or broad underlyings like UNG, FXI, SLV, GLD etc. 
2. Lower priced, liquid ETFs (liquidity here is super important given that an IC consists of 4 legs and hence liquidity is vital to getting good fills)

**Backtested Results**

![image](https://user-images.githubusercontent.com/68678549/96748259-df5c0180-13fb-11eb-9fef-44e43bb3c4b6.png)

Using CMLviz [TMP](https://cmlviz.com/signup/options-success-www.php) (*which has a 30 day 'free trial'*), I backtested iron condors over a range of 5 years with different ETFs. Most were profitable but these were the results from FXI. The buy-and-hold metric here isn't the most accurate because it does not take into account the capital risked. So, if we were to compare apples to apples, we would calculate:  

```python
initial_invest_shares = 100
initial_stock_price = 39.49
end_stock_price = 44.84 
total_gain = (end_stock_price-initial_stock_price)*initial_invest_shares

Perc_return = (total_gain/(initial_invest_shares*initial_stock_price))*100
print("%.2f" % Perc_return)
```

    13.55

Giving us a %return of 13.55% on the initial amount invested if we had simply bought and held over the 5 years. Even taking into account the $77 of commissions ($0.60 per option on TDAmeritrade), the **%return** of $202/$50 was still a respectable 404%. In terms of buying power reduction, buying and holding would also have required an upfront of $3949 (for buying 100 share), compared to $50 buying power reduction when opening a FXI Iron Condor. 

![image](https://user-images.githubusercontent.com/68678549/96749961-0ca9af00-13fe-11eb-94c3-e4b6e2340bbc.png)

Looking at UNG, another 'stable'-ish ETF, in a period where the stock price barely rose, selling ICs on UNG made consistent profits. The 1141% return here does look glamorous but one needs to look at the fact that this was due to how much was risked vis how much was made. Still, not too shabby a performance and definitely having a chance of outperforming a "fund of funds" (hint hint).

## Cash-Secured Puts

```python
payoff_short_put = put_payoff(sT, strike_price_short_put, premium_short_put) * -1.0

fig, ax = plt.subplots()
ax.spines['bottom'].set_position('zero')
ax.set(xlim=[250,400],ylim=[-20,10])
ax.plot(sT,payoff_short_put,label='Cash-Secured Put',color='m')
plt.xlabel('Stock Price')
plt.ylabel('Profit and loss')
plt.legend()
plt.grid()
plt.show()
```

![image](https://user-images.githubusercontent.com/68678549/96751821-3cf24d00-1400-11eb-9b74-f017d0efd5ca.png)

Cash-Secured Puts are basically the selling (or shorting) of Put options. In this instance, they are called Cash-Secured because I must have sufficient funds to buy the stocks should the puts expire in-the-money. This is the "Cash-Secured" part because my puts are covered by the funds I have to potentially buy the actual underlyings. 

### Entry: 

1. DTE: 30 - 45 days
2. Strike selection: Typically sell 30D 
3. IV Rank: Above 30

**Management:**

1. Management at either 50% profits or 21 DTE
2. Will roll down (to lower strikes) and out (to a further expiration cycle) if put is tested (i.e. if put becomes ITM)

**Universe:**

1. Lower priced ($10-$50) stocks with good fundamentals (which you can read about [here]()) and bullish sentiment
2. Solid cash flow and generally stable chart without wild gyrations and swings in price

**Backtested Results**

For CSPs, I found them harder to backtest on TMP because of the fact that TMP did not support the "Cash-Secured" part of writing puts.     The backtesting system typically just wrote off the trade as a loss, not taking ownership of the stock and then buying and holding on the stock, waiting for it to pass breakeven before selling it. One place that I did find rather interesting 'backtesting' results was CBOE's own webpage. CBOE has a number of synthetic indexes which track certain strategies and one of these strategies is a Put-Write (i.e. Put Selling) strategy. Looking at this graph of returns vs standard deviation (which can be taken as a rough substitute for volatility), it appears that the Put-Write (PUT) strategy just narrowly underperformed buying and holding the S&P500 but offered a significantly lower standard deviation. Something to think about, I guess. 

![image](https://user-images.githubusercontent.com/68678549/96753396-64e2b000-1402-11eb-89bd-f08e5a186bfe.png)

# Conclusion

So, there you have it. The two main strategies that I intend to deploy in this quest to beat an investment company. While I would provide more backtested results here, most of my last post was spent talking about why historical returns do not guarantee future returns and I do strongly believe in that. Hence, backtested results, while impressive, are not things that I trust 100% to be indicative of future results. It is forward-testing and time that will tell - here's to one year ahead of competing with Endowus. 