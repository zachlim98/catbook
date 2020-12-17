---
title: Using the Right Tools for the job
categories: [trading]
---

# Introduction

As I wrote about in my last post [here](https://zachlim98.github.io/2020-12/Using-Pandas-to-set-SL), backtesting is an important part of any strategy. So, I spent the better part of the last few days trying to backtest a [Dual Momentum strategy](https://robotwealth.com/dual-momentum-review/). I'd seen how it had been done in R and had managed to do it manually but wanted to try to do it with one of the pre-existing backtesting libraries available in Python. Having used both backtrader and bt, I eventually settled on backtrader as I was more comfortable with the library. Furthermore, it had greater depth to it, allowing for more things to be run "in-house", as compared to bt where one had to do more preparation for the data (as described here). 

But... I quickly ran into some problems using backtrader and I realized the importance of using the right toolkit for the job you're trying to achieve. Just wanted to share that here today in hopes that it helps someone!  

## Contents
1. Individual Stock vs Whole Portfolio Testing
2. The Right Toolkit
3. Conclusion

## 1. Individual Stock vs Whole Portfolio Testing

The Dual Momentum strategy is essentially a whole portfolio, "all-in" strategy. This means that the portfolio is almost always fully invested. Rules-wise, it really isn't that complicated and essentially involves calculating the rate of return of assets over the last 12 months and rebalancing between 3 different ETFs (US Stock Market, Global (excl. USA) Stock Market, and Bonds). Its a systematic rules-based system that compares the rate of return of the respective stock markets to the "risk-free" return (from Treasury Bills). 

It seemed easy enough and hence I ventured to use backtrader to do it. Alas, I ran into a huge problem that I just could not seem to overcome. Since the portfolio was an "all-in" strategy, it would involve the simultaneous closing and opening of a position (for e.g. switching from the US Stock Market to bonds or vice versa). This was how I simulated a section of it in backtrader:

```python
    def next(self):
        if self.mospy[0] > self.moveu[0]:
            if self.mospy[0] > self.mobil[0]:
                for i in [1,2,3]:
                    self.order = self.order_target_percent(data=self.datas[i], target=0)
                self.order = self.order_target_percent(data=self.datas[0], target=1)
                print('Opening a position of SPY')
                
            else:
                for i in [0,1,2]:
                    self.order = self.order_target_percent(data=self.datas[i], target=0)
                self.order = self.order_target_percent(data=self.datas[3], target=1)
                print('Opening a position of AGG')
```

 For those of you familiar with backtrader, what this essentially did was to close all existing positions (setting the target to 0), and then opening a position in either the SPY (a US S&P500 ETF) or AGG (an aggregate bond market ETF). In order to set the target to 1 however, it meant that the broker within backtrader needed to calculate the amount of cash left in the system. However, the way that backtrader ran was via lines, with each day (or in this case month) being a line on its own. As a result of that, since the month wasn't over, the cash balance was still **0** since it had been "all-in" on the previously opened position!

![image](https://user-images.githubusercontent.com/68678549/102229049-4c84a280-3f26-11eb-9c30-667914e5612d.png)

This resulted in this issue, whereby the orders kept getting rejected due to a lack of margin! The only way to go about was to use `cheat-on-close` which would allow the backtrader broker to ignore the next bar's price but instead take the close of the current bar. I would also have to set `cerebro.broker.set_checksubmit(checksubmit=False)` which would force the broker to not check the current available margin before submitting. Now, the man behind backtrader (on forums) constantly iterated that this using `cheat-on-close` would result in inaccurate results because of how backtrader was built - i.e. not perfectly suited for "all-in" strategies.

And that was when I realised the importance of...

## 2. The Right ToolKit

You see, despite my comfort with backtrader, I was also quite familiar with the bt system. And I knew that the bt system was more suited toward full portfolio, always invested strategies. I had started with bt, but then wandered into trying to test individual stock strategies (which backtrader was much better for) and decided to switch to backtrader. And because I'd become so comfortable with backtrader, I so stubbornly tried to use it, even though I knew (THE WHOLE TIME I WAS STRUGGLING) that bt was a better option.

Having made this realization, I switched to bt and the problem was easily resolved since bt accounts for this "all-in" issue and its rebalancing module checks after each opening and closing of a position. 

# Conclusion

This was a really short article but the main point here is really that there are some tools that are better suited for certain jobs. I just got so bogged down in trying to use backtrader that I persisted in using it, even when I KNEW that there was a better option available. In coding (as in life), flexibility is the key! Thanks for reading! :) 
