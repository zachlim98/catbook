---
title: Does the World's Worst Investor still make money?
categories: [data, trading]
---

# Introduction

I read an article recently about why a regular investment plan far outweighs trying to time the market. While that definitely made sense, I also wondered what happened if one were to constantly invest just before the market collapse. What if one was the **worst** investor in the world? There have been a few articles on this before but they gave rather skimpy details and I was curious to see it on my own and visualize it. Furthermore, I also wanted to explore if dollar-cost-averaging (DCA) (monthly) or DCA (yearly) would be more profitable. Would it make a difference?  

# tl;dr

As always, a tl;dr to save your time

1. The simple answer: Even if one were the WORST investor of all time and invested annually just before a market crash, one would still have walked away with a cool ~534k (for a ~180% ROI from investing ~$320,000 in the S&P500 from 1990) 
2. DCA (yearly) seems to be slightly more profitable than DCA (monthly). This could be a function of your funds having more time in the market and hence benefitting from the bull-runs of the past decade. 

# Pre-Exploration

## Data Collection

To get the data for both the S&P500 and the Straits Times Index (just to see how homeboy does), I used Yahoo Finance and extracted the annual returns (including dividends for the S&P500) and the end of month data for the STI (excluding dividends). The data was already quite clean so not much formatting needed to be done. 

## Visualizing the Rate of Return

Before I plunged into the actual exploration, I just wanted a quick overview of the annual rate of returns for the S&P500. I plotted it out quickly into a histogram.

```python
plt.hist(annual_return["Return"])
plt.axvline(x=avg, color='red')
plt.text(avg+1,10,'Mean',rotation=0, color = 'white', fontsize='large')
plt.ylabel("Count")
plt.xlabel("Annual Returns")
plt.title("Annual Returns of SP500 since 1926")
```

![Annual Returns](https://user-images.githubusercontent.com/68678549/98247498-a6e92580-1fae-11eb-885c-0799f31143d5.png)

The mean annual rate (granted since 1926) was fairly high but it was also clear that there were outlier events of -40%. What happens if one were to **ALWAYS** invest just before those crashes? Let's find out!

# The World's Worst Investor - S&P500

Let's imagine for a moment that I'm Tom. I'm born in 1969 and turn 21 years old in 1990. With the savings that I've accumulated over the past few years ($10,000), I decide that I want to start investing in the market. But I'm a rather fearful investor and so I only invest when the markets are **SOARING**. Unfortunately, I have such bad luck that whenever I invest, the markets always plunge more than 5% the next year... Let's see how that turns out! 

Here, I filter the annual_return for only years after 1990


```python
EOM_SP_df = pd.read_csv("sp500m.csv")
EOM_SP_df["pct monthly return"] = 0.00

for i in range(1, len(EOM_SP_df)):
    EOM_SP_df['pct monthly return'][i] = ((EOM_SP_df['Adj Close'][i] - EOM_SP_df['Adj Close'][i-1]) / EOM_SP_df['Adj Close'][i-1]) * 100

EOM_SP_df['Date'] = pd.to_datetime(EOM_SP_df['Date'])
EOM_SP_df = EOM_SP_df[EOM_SP_df["Date"]>"1989-12-01"]
```
Following which, I run through all the years, simulating two different scenarios. In the first scenario, my initial investment is $10,000. Every month, I intend to invest $833.33 ($10,000 per year/12). However, because I am fearful, I don't mechanically invest the money each month. Instead, I save up the money and invest only when the markets are at their highest (because this gives me confidence that the market is going **up**) 

Unfortunately, luck works against me and every time after I invest, the month following that, the market drops by more than 10%! 

```python
today_port_val = 10000
reg_inv = 10000/12
port_val_us_poor = []
no_adds = 0

for i in range(1,len(EOM_SP_df)):
    if EOM_SP_df.iloc[(i),7]<=-10:
        today_port_val = ((today_port_val)*(1+(EOM_SP_df.iloc[(i-1),7]/100)))+(reg_inv*len(EOM_SP_df.iloc[previousi:i]))
        port_val_us_poor.append(today_port_val)
        previousi = i
    else:
        today_port_val = today_port_val*(1+(EOM_SP_df.iloc[(i-1),7]/100))
        port_val_us_poor.append(today_port_val)
```
In contrast, my friend Jane couldn't care less about the markets. Every month, she just puts in $833.33, regardless of whether its raining or shining. 

```Python
#Jane's portfolio calculation
today_port_val = 10000
reg_inv = 10000/12
port_val_us_reg = []
no_adds = 0

for i in range(1,len(EOM_SP_df)):
    today_port_val = (today_port_val+reg_inv)*(1+(EOM_SP_df.iloc[(i-1),7]/100))
    port_val_us_reg.append(today_port_val)
```
Finally, I have a friend James who doesn't want to invest in the stock market at all. He stuffs his money in a pillowcase under his bed.

```python
#James
today_port_val = 10000
reg_inv = 10000/12
port_val_us_sav = []
no_adds = 0

for i in range(1,len(EOM_SP_df)):
    today_port_val = today_port_val+reg_inv
    port_val_us_sav.append(today_port_val)
```
Tidying up and appending the data back to the dataframe...

```Python
port_val_us_poor.append(port_val_us_poor[-1])
port_val_us_reg.append(port_val_us_reg[-1])
port_val_us_sav.append(port_val_us_sav[-1])

EOM_SP_df['Portfolio Value Reg'] = port_val_us_reg
EOM_SP_df['Portfolio Value (Worst Case)'] = port_val_us_poor
EOM_SP_df['Portfolio Value (Saving)'] = port_val_us_sav
```
And plotting it out!

```python
import matplotlib.dates as mdates
fig, ax = plt.subplots()
ax.plot("Date", "Portfolio Value (Worst Case)", data=EOM_SP_df, color='r')
ax.plot("Date", "Portfolio Value Reg", data=EOM_SP_df, color='b')
ax.plot("Date", "Portfolio Value (Saving)", data=EOM_SP_df, color='purple')
years = mdates.YearLocator(5) 
years_fmt = mdates.DateFormatter('%Y')
ax.xaxis.set_major_locator(years)
ax.xaxis.set_major_formatter(years_fmt)
plt.setp(ax.get_xticklabels(), rotation=70, ha="center")
plt.annotate("%.2f" % EOM_SP_df.iloc[-1,9], xy=(mdates.date2num(EOM_SP_df.iloc[-1,0]),EOM_SP_df.iloc[-1,9]), xycoords='data', color='r', bbox=dict(boxstyle="round,pad=0.3", fc="white", ec="b", lw=0.5))
plt.annotate("%.2f" % EOM_SP_df.iloc[-1,8], xy=(mdates.date2num(EOM_SP_df.iloc[-1,0]),EOM_SP_df.iloc[-1,8]+1000), xycoords='data', color='b', bbox=dict(boxstyle="round,pad=0.3", fc="white", ec="b", lw=0.5))
plt.annotate("%.2f" % EOM_SP_df.iloc[-1,10], xy=(mdates.date2num(EOM_SP_df.iloc[-1,0]),EOM_SP_df.iloc[-1,10]+1000), xycoords='data', color='purple', bbox=dict(boxstyle="round,pad=0.3", fc="white", ec="b", lw=0.5))
ax.legend(['Worst Case','Regular Monthly Investment','Pure Savings'])
plt.ylabel("Portfolio Value")
plt.xlabel("Year")
plt.title("S&P500 Returns (Monthly)")
plt.grid()
```

![SP500 Returns](https://user-images.githubusercontent.com/68678549/98276684-4c14f580-1fd1-11eb-8ca5-1b3e400bf284.png)


Not unsurprisingly, we see that the regular monthly investment (go Jane!) worked out the best. It returned ~1.2 million (for a ~390% ROI compared to the savings plan). 

Shockingly, Tom, the world's "worst" investor didn't do too badly either. He still made a respectable ~534k (for a ~180% ROI). Looks like even if you get the timing wrong every.single.time but **don't sell** and keep holding it, you'll still do relatively well. 

# The World's Worst Investor - STI

So, in a parallel universe, Tom lives in Singapore and has changed his name to Ah Kow. Again, he decides to invest in the stock market but his timing is really (in colloquial Singlish) *suay*. Not unlike his alternative character, Ah Kow is fearful too and decides to only invest when the markets are at All-Time-Highs. Which means that they inevitably go down the next month (each time by more than 10%!) 

```python
import matplotlib.dates as mdates
fig, ax = plt.subplots()
ax.plot("Date", "Portfolio Value (Worst Case)", data=EOM_STI_df, color='r')
ax.plot("Date", "Portfolio Value Reg", data=EOM_STI_df, color='b')
ax.plot("Date", "Portfolio Value (Savings)", data=EOM_STI_df, color='purple')
years = mdates.YearLocator(5) 
years_fmt = mdates.DateFormatter('%Y')
ax.xaxis.set_major_locator(years)
ax.xaxis.set_major_formatter(years_fmt)
plt.setp(ax.get_xticklabels(), rotation=70, ha="center")
plt.annotate("%.2f" % EOM_STI_df.iloc[-1,9], xy=(mdates.date2num(EOM_STI_df.iloc[-1,0]),EOM_STI_df.iloc[-1,9]-20000), xycoords='data', color='r', bbox=dict(boxstyle="round,pad=0.3", fc="white", ec="b", lw=0.5))
plt.annotate("%.2f" % EOM_STI_df.iloc[-1,8], xy=(mdates.date2num(EOM_STI_df.iloc[-1,0]),EOM_STI_df.iloc[-1,8]+30000), xycoords='data', color='b', bbox=dict(boxstyle="round,pad=0.3", fc="white", ec="b", lw=0.5))
plt.annotate("%.2f" % EOM_STI_df.iloc[-1,10], xy=(mdates.date2num(EOM_STI_df.iloc[-1,0]),EOM_STI_df.iloc[-1,10]), xycoords='data', color='purple', bbox=dict(boxstyle="round,pad=0.3", fc="white", ec="b", lw=0.5))
ax.legend(['Worst Case','Regular Monthly Investment','Pure Savings'])
plt.ylabel("Portfolio Value")
plt.xlabel("Year")
plt.title("STI Returns")
plt.grid()
```

![STI Returns](https://user-images.githubusercontent.com/68678549/98276685-4cad8c00-1fd1-11eb-96a3-d460c613a800.png)


Now this was slightly more surprising than the S&P500 study. In this case, it turns out that Ah Kow actually LOST money by entering during the worst periods of the market. Compared to just saving his money, he was down about $33,000 - not an insignificant sum! 

Of course, the regular monthly investments didn't do particularly well either. While the returns were definitely higher than the savings plan for most of the backtested period, the portfolio suffered heavily from market drops (and this is why <10% of my portfolio is in the STI)

# Yearly or Monthly DCA?

The next thing I wanted to examine was whether it made a difference if one invested monthly or saved for the whole year and then invested one-shot at the end of the year. In order to compare the two, I had to convert the yearly dataset to follow the `numpy datetime` convention. I did this by converting everything and appending it to a list called ears (geddit hehe)

```Python
EOY_SP_df = annual_return_recent

ears = []

for i in range(1991,2021,1):
    currenty = ("{}-01-01".format(i))
    ears.append(currenty)

ears.reverse()
EOY_SP_df['Date'] = ears

EOY_SP_df['Date'] = pd.to_datetime(EOY_SP_df['Date'])
```

Following that, I plotted it out again. 

```Python
import matplotlib.dates as mdates
fig, ax = plt.subplots()
ax.plot("Date", "Portfolio Value Reg", data=EOY_SP_df, color='darkblue')
ax.plot("Date", "Portfolio Value Reg", data=EOM_SP_df, color='deepskyblue')
years = mdates.YearLocator(5) 
years_fmt = mdates.DateFormatter('%Y')
ax.xaxis.set_major_locator(years)
ax.xaxis.set_major_formatter(years_fmt)
plt.setp(ax.get_xticklabels(), rotation=70, ha="center")
plt.annotate("%.2f" % EOY_SP_df.iloc[0,5], xy=(mdates.date2num(EOY_SP_df.iloc[0,6]),EOY_SP_df.iloc[0,5]), xycoords='data', color='darkblue',bbox=dict(boxstyle="round,pad=0.3", fc="white", ec="b", lw=0.5))
plt.annotate("%.2f" % EOM_SP_df.iloc[-1,8], xy=(mdates.date2num(EOM_SP_df.iloc[-1,0]),EOM_SP_df.iloc[-1,8]+1000), xycoords='data', color='deepskyblue',bbox=dict(boxstyle="round,pad=0.3", fc="white", ec="b", lw=0.5))
ax.legend(['Yearly Reg Investment','Monthly Reg Investment'])
plt.ylabel("Portfolio Value")
plt.xlabel("Year")
plt.title("S&P500 Yearly DCA vs Monthly DCA")
plt.grid()
```
![MonthvYear](https://user-images.githubusercontent.com/68678549/98276678-4ae3c880-1fd1-11eb-84ac-47f9fba4aa26.png)

Interestingly enough, the Monthly DCA actually did slightly better than the Yearly DCA. This is slightly different from comparing Lump Sum Investment to DCA (which has been touched on extensively). Here, its a shorter DCA compared to a longer DCA and it appears that the shorter one wins likely due to your funds having *more time in the market* (which, as the saying goes, is more important than timing the market). 

# Conclusion

1. You can be the world's 'worst' investor...and still make money. As shown by the S&P500 example, the returns generated by the S&P500 meant that even if you invested just before every significant market correction in the last 30 years, you would still have come out with a 100% ROI. This is significant because people are constantly worried about entering the market at the wrong time. This research shows that even if you were SO UNLUCKY to enter wrong every single time, you'd still make *some* money 
2. Investing every month seems to perform slightly better than investing every year. This could be merely a function of the bull market that we have been in but giving your money more time in the market (by investing monthly) gave it a ~200k edge over the last 30 years 
3. You can't call the bottoms of markets - the best way to invest is to mechanically invest every month. In the case of the STI, investing every month still gave you a slight edge over savings (a very very slight edge).  

