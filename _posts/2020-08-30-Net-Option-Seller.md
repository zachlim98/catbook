
# Introduction

There's a famous subreddit known as /r/thetagang that is completely centered around the selling of options vis the buying of options. This selling bias has also been most famously promoted on [tastytrades](https://www.tastytrade.com/tt/shows/ryan-beef/episodes/the-argument-for-selling-options-09-22-2017) as the "tastytrade" way. One of the main justifications for why one should sell options is the concept of premium and volatility. This article will explore whether this bias is statistically justified. 

## tl;dr

1. It appears that implied volatility has indeed been historically higher than realized volatility (statistically significant-ly higher)
2. However, there are prolonged periods of time (particularly in market downturns) where this is not the case. Risk management/position sizing continues to be important. 

## Premium and Volatility
All options have a certain premium. When one sells or writes an option, one receives premium. In contrast, when one buys an option, one has to pay premium. The pricing of options has been the subject of much mathematical modelling/debate. Through these models, one can estimate implied volatility (IV). 

IV is the market's forecast of a likely movement in a security's price ([Investopedia](https://www.investopedia.com/terms/i/iv.asp)). More specifically, it expresses the expected annualized one standard deviation move for the price of stock. For instance, a $100 stock with IV 0.2 implies that there is a ~68%  (one standard deviation) chance that the stock might end up between $80 and $120 in a year's time. Much has been written about IV and [this](https://www.optionsplaybook.com/options-introduction/what-is-volatility/) is a good, simple resource for someone with no knowledge of this. In contrast, historical volatility (HV) is backward looking. It takes into account "average deviation from the average price of a security" ([Fidelity](https://www.fidelity.com.sg/beginners/what-is-volatility/implied-vs-historical-volatility)) and the formula uses annualized standard deviations of returns to get HV for a single day. The formula for HV is as follows:

![image](https://user-images.githubusercontent.com/68678549/97110411-2f94d580-1714-11eb-8b2c-98e8e166f309.png)
(source: [macroption.com](https://www.macroption.com/historical-volatility-calculation/))

As options are a form of 'insurance', they tend to get more expensive when there is fear/uncertainty in the markets. The argument that is put forth by tastytrade is that this fear/uncertainty is typically overstated and people pay more premium for options than what they are actually worth (*note: this is just layperson generalizing - there's a lot more to it than this but this is the general argument*). As a result, option sellers have an edge since they are selling 'overpriced' products. 

![image](https://user-images.githubusercontent.com/68678549/97111118-f6f6fb00-1717-11eb-8d53-93b4469747d4.png)
(source: [tastytrade.com](https://www.tastytrade.com/tt/shows/market-measures/episodes/should-we-buy-options-03-31-2020?_sp=d1d718d3-4499-43b0-b673-e0be2c80bdcf.1603639078915))

Graphs like the one above are commonly shown, with the argument that IV was higher than HV (or what they called realized volatility) **84%** of the time. However, while I did generally trust tastytrade's research, I wanted to see this for myself and also wanted to test whether there was a statistical difference. I decided to test this out on SPY. 

## Gathering Data

In order to gather my data, I turned to the thinkorswim platform offered by TDAmeritrade. Thinkorswim has a built in IV indicator (which use the Bjerksund-Stensland model to estimate IV) and a HV indicator. To export my data, I stumbled upon this fantastic piece of [code](https://usethinkscript.com/threads/exporting-historical-data-from-thinkorswim-for-external-analysis.507/#post-14606) by korygill which allowed one to export data from thinkorswim by creating a fake strategy. 

```c++
# data_exporter_IV
#
# Strategy to capture every bar OHLC and P, the previous close.
# Useful for exporting data from TOS into a CSV file for further processing.
#
# Author: Kory Gill, @korygill
# Edited: Krismerful, @krismerful
declare upper;
declare once_per_bar;

input startTime = 820; #hint startTime: start time in EST 24-hour time
input endTime = 1600; #hint endTime: end time in EST 24-hour time

def adjStartTime = startTime;# - 1;
def adjEndTime = endTime;# - 1;

def agg = GetAggregationPeriod();

# we use a 1 bar offset to get orders to line up, so adjust for that here
def marketOpen = if agg >= AggregationPeriod.DAY then 1 else if SecondsTillTime(adjEndTime) >= 60 and SecondsFromTime(adjStartTime) >= -60 then 1 else 0;

def HV = HistoricalVolatility();

AddOrder(OrderType.BUY_TO_OPEN, 
    marketOpen, 
    low, 
    1, 
    Color.WHITE, 
    Color.WHITE, 
    name = ";" + open[-1] + ";" + high[-1] + ";" + low[-1] + ";" + close[-1] + ";" + imp_volatility[-1] + ";" + HV[-1]);
AddOrder(OrderType.SELL_TO_CLOSE, marketOpen, high, 1, Color.WHITE, Color.WHITE, name = "SellClose");
```

The full explanation for how this works can be found in the link above. I modified it to suit my needs (adding in the `imp_volatility` indicator and defining a new `HistoricalVolatility` indicator) and exported the values...into one of the messiest CSV files I'd ever seen. 

![image](https://user-images.githubusercontent.com/68678549/97111407-8224c080-1719-11eb-8693-7e905a40fac4.png)

I wrote a function to import **and clean** the csv file

```python
import pandas as pd
import datetime as dt
import matplotlib.pyplot as plt
import seaborn as sns
import numpy as np
import matplotlib.dates as mdates

#create function to import TDA exports

def cleanTD(csvfile):
    header_list = ["Fill1","Fill2","Open","High","Low","Close","ImpVol","HistVol","Fill3","Fill4","Fill5","Date"]
    data = pd.read_csv(csvfile,sep=';',skiprows=5,names=header_list)
    data = data.iloc[::2].drop(0).iloc[:,[6,7,11]]
    data = data.dropna()
    data['HistVol'] = data['HistVol'].str.strip(')').astype(float)
    data['ImpVol'] = data['ImpVol'].astype(float)
    data['Date'] = data['Date'].apply(pd.to_datetime)

    return(data)

SPY0510 = cleanTD("data.csv")
SPY1520 = cleanTD("data2.csv")
```

And then imported SPY (an S&P500 ETF) data from 2005 - 2010 (to visualize the market crash) and 2015 - 2020 (for more recent data and also to see the 'corona' crash). 

## Plotting Data

I then wrote another function to plot out the IV and HV and plotted out the two sets of data. 

```python
def plotVol(df):
    ax = df.plot(x='Date')
    sns.set_style('whitegrid')
    sns.set_context("notebook",font_scale = 1.5)
    ax.set_title('Volatility over time')
    ax.set_ylabel('Volatility')

    ax.xaxis.set_major_locator(mdates.YearLocator())
    ax.xaxis.set_minor_locator(mdates.MonthLocator(bymonth=6))
    ax.xaxis.set_major_formatter(mdates.DateFormatter("\n%Y-%b"))
    ax.xaxis.set_minor_formatter(mdates.DateFormatter("%b"))
    plt.setp(ax.get_xticklabels(), rotation=0, ha="center")
    L=plt.legend()
    L.get_texts()[0].set_text('Implied Volatility')
    L.get_texts()[1].set_text('Realized Volatility')

plotVol(SPY0510)
plotVol(SPY1520)
```

![output_6_0](https://user-images.githubusercontent.com/68678549/97111508-155df600-171a-11eb-865b-cacb868b7665.png)

![output_7_0](https://user-images.githubusercontent.com/68678549/97111515-1db63100-171a-11eb-886e-a455881210d2.png)

Unsurprisingly, the graphs more or less resembled what tastytrade constantly showed. Visually, it did appear that IV was consistently higher than RV. However, I wanted to find out if they were statistically significant. The IV and HV data did not follow a normal distribution and hence I could not use the standard Paired T-test. 

## Significance Testing

Instead, I used a Wilcoxon Sign-Rank Test which allows one to compare two related samples to assess if their population mean ranks differ. In this case:

H₀  = The medians of IV and HV are equal
Hₐ = The median of IV is higher than HV

I wanted to run a one-tailed test to see if the difference was positive. This test was easily done with `scipy` and `wilcoxon`, setting the alternative hypothesis to 'greater'. 

```python
from scipy import stats

SPY0510['d'] = SPY0510.ImpVol - SPY0510.HistVol
stat, p = stats.wilcoxon(SPY0510['d'],alternative='greater')
print('2005 - 2010: Statistics=%.3f, p=%.3f' % (stat, p))

SPY1520['d'] = SPY1520['ImpVol'] - SPY1520['HistVol']
stat, p = stats.wilcoxon(SPY1520['d'],alternative='greater')
print('2015 - 2020: Statistics=%.3f, p=%.3f' % (stat, p))
```
    2005 - 2010: Statistics=667591.000, p=0.000
    2015 - 2020: Statistics=987040.500, p=0.000

The p-values of both sets of data were <0.05, leading us to reject the null hypothesis that the medians were equal. 

```python
perc1_overstated = ((SPY0510[SPY0510.d>0.0].count())/SPY0510['d'].count())*100
perc1 = perc_overstated.values
perc2_overstated = ((SPY1520[SPY1520.d>0.0].count())/SPY1520['d'].count())*100
perc2 = perc2_overstated.values
print('Percentage of time that IV was overstated: \n2005-2010:%.1f\n2015-2020:%.1f' % (perc1[1],perc2[1]))
```
```
    Percentage of time that IV was overstated: 
    2005-2010:84.6
    2015-2020:83.5
```

Checking on the 84% claim of tastytrades, I was pleased to see that their purported 84% was in fact relatively accurate and reflected in both periods. 

## Capitalizing on IV and HV Differences

![output_9_1](https://user-images.githubusercontent.com/68678549/97113516-e6e61800-1725-11eb-89af-3689b5011bc3.png)

However, despite the fact that IV is in fact overstated 84% of the time, we must account for the times where IV isn't and an option seller could get hurt by it. For instance, during the "Coronavirus Crash" in 03/20, HV stayed higher than IV for a good month and a half. Although tastytrade always argues that volatility is "mean-reverting", the mean reversion may take awhile to materialize and one could lose a significant amount if one were short an undefined risk trade (e.g. a short strange/straddle). One way to get around this, some argue, is to only engage in short selling when IV Rank or IV Percentile (two measures of IV) are above a certain value. However, this does not solve the problem as IV can stay above these values for a significant period of time, especially during a period of prolonged market downturn. 

Personally, I don't have an answer to this (after all if I did, I wouldn't really be sharing it and would be laughing my way to the bank). Instead, as with all trading strategies, I think there's an element of risk management/position sizing. This ensures that in periods where IV is high and **remains** high, your losses are limited and it does not wipe out whatever gains you made when IV was indeed overstated. 

# Takeaways

In conclusion, this was in no way intended to be a piece of serious market research. Instead, it was just an interesting experiment in extracting data from thinkorswim and manipulating/testing it to see if tastytrade's claims were indeed true (from an independent viewpoint). While their claims do appear to be validated, the shortcomings of their strategy (as with all strategies) is that it works until it doesn't. This is where position sizing and one's ability to manage one's risk on each trade comes into play. 

