---
title: Cleaner Monte Carlo Simulation graphs
categories: [data]
---

# Introduction

A while ago, I was running some Monte Carlo simulations on portfolio allocations and trying to figure out how to make the graphs look better. Now, I'm not some graph expert of visualization master but I did figure out ways to make it look better and this will just be a short article to share that with you!

# The Data

As mentioned, I ran a number of Monte Carlo simulations in order to figure out what my portfolio could look like in 30 years. If you're interested, this is a snippet of the code (but the full code can be found on my [github](github.com/zachlim98)). I essentially had three different funds that I simulated over the next 30 years (using their mean and standard deviation). I weighted these funds differently and then ran 1000 simulations on them. 

```python
# function to simulate the number of years invested
def EstimateReturns(initial_invest, expected_returns, std_dev, start_date, duration, monthly_con):
    final_money = []
    monthcount = []
        
     
    for month in range(duration*12):
        this_return = np.random.normal(expected_returns,std_dev)
        final_money.append(initial_invest)
        initial_invest = (initial_invest * (1+this_return)) + monthly_con
        a_date = (start_date + relativedelta(months=month))
        monthcount.append(a_date.strftime('%Y-%m'))  
    
    d = {'Date' : monthcount, 'Available Funds' : final_money}
    df = pd.DataFrame(data = d)
    final_funds.append(final_money[-1])
    return(df)

# Running the simulation n times
final_funds = []

for i in range(simulations):
    Stock1 = EstimateReturns(10000, 0.00685, 0.034, datetime.date(2021, 1, 1), test_duration, 1000)
    Stock2 = EstimateReturns(10000, 0.00109, 0.0509, datetime.date(2021, 1, 1), test_duration, 1000)
    Stock3 = EstimateReturns(10000, 0.0015, 0.0184, datetime.date(2021, 1, 1), test_duration, 1000)
    Stocks_total = pd.merge(Stock1,Stock2,how='left',on='Date')
    Stocks_total = pd.merge(Stocks_total,Stock3,how='left',on='Date')
    Stocks_total = Stocks_total.rename(columns={"Available Funds_x" : "SP500", "Available Funds_y" : "STI", "Available Funds" : "EIMI"})
    Stocks_total['Portfolio_Value {}'.format(i)] = Stocks_total['SP500']*weights[0] + Stocks_total['STI']*weights[1] + Stocks_total['EIMI']*weights[2]
    tdf = pd.merge(tdf,Stocks_total[['Date','Portfolio_Value {}'.format(i)]],on='Date', how='left')
```
# Initial Plot

![image](https://user-images.githubusercontent.com/68678549/98444045-6aeac780-214a-11eb-820f-f2b8ef61b925.png)

As you can see from the initial plot, it was not useful in the slightest. All **1000** outcomes had been plotted in full colour and there was really no useful information that could be gleaned from this. I could tell the rough range of outcomes but other than that, it wasn't really particularly useful. 

I decided that the most important portfolios to visualize would be (a) the 75th Percentile, (b) the Median, (c) the 25th Percentile, and (d) the base portfolio (i.e. pure savings without investing in the market)

# Getting Data for New Plots

In order to get the position of the 25th, 50th, and 75th Percentile portfolios, I combined the quantile and index function, converting them to strings so that I could use their column names later when plotting

```python
quantiles = []
# getting positions of the following quantiles and converting to string
for i in [0.25,0.5,0.75]:
    indx = str(tryy[tryy[tryy.shape[1]-1]==float(tryy.iloc[1:,tryy.shape[1]-1].quantile([i], interpolation = 'higher'))].index.tolist()).strip("[], '")
    quantiles.append(indx)
```

When drawing the plot, I set all the plots to color grey. This would make them fade into the background and would allow me to highlight important portfolio outcomes (as per the aforementioned important ones)

```python
fig, ax = plt.subplots()
# plot all the outcomes
for i in range(1, len(tryy)):
    ax.plot(tryy.iloc[0,:], tryy.iloc[i,:], color='grey') #set color to grey
    labels.append("Portfolio No: {}".format(i))
```

![image](https://user-images.githubusercontent.com/68678549/98444216-67a40b80-214b-11eb-8dc7-f66c39240170.png)

Obviously nothing to write home about but now at least we have an empty canvas to work on!

```python
color=['y','r','b'] #create color map
lab=['25th Percentile', 'Median', '75th Percentile'] #create labels

for i in range(0,len(quantiles)): 
    ax.plot(tryy.iloc[0,:], tryy.loc[quantiles[i],:], color=color[i], label=lab[i])
    
leg = ax.legend() # show legend
```

The list of `quantiles` were all strings. This allowed me to easily call the particular portfolio with `.loc` and refer to the exact portfolio. I also created a list of colors to use as well as labels. I also added a legend to show the labels. With that, this was the final product!

![602020](https://user-images.githubusercontent.com/68678549/98444290-c8334880-214b-11eb-9350-425f8b4f9f60.png)

# Conclusion

Compared with the initial plot, the final result looked so much better. It gave the same background information (the rough range of possible outcomes) but also highlighted the important portfolios. What could perhaps have been even better would have been to remove the outlier outcomes. However, for the purposes of my study, I wanted to show the full range of outcomes and hence left it in there. 

Data visualization is such an important part of the data story. While we can have the coolest results and findings, a good graph that sends a clear message is also vital especially in today's graphic-intensive culture. 

