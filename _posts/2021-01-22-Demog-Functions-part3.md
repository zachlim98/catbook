---
title: Exploring Demography with Python - Part 3
categories: [coding, data]
---

# Introduction

Welcome back to part 3 of our demography functions series! Thus far, we've looked at population growth rates and life expectancy. But, what causes a population to grow? That's right - babies! Lots and lots of babies. Hence, this week, we'll be taking a look at fertility rates - learning how to calculate the two most popular fertility measures of Total Fertility Rate and Mean Age of Childbearing. Let's dive right in!

## Key Measure 5 - Total Fertility Rate

### Theory

The total fertility rate is essentially the sum of the **age-specific fertility rates** for women in their childbearing ages (which is usually assumed to be 15 - 50). This gives us the average number of children a women would theoretically have if she were to pass through all the childbearing age brackets. The assumption is that each age bracket carries the same weight and is hence a *synthetic cohort* because the women has **not** yet lived through those ages. 

### Calculations

The calculations for TFR then, are relatively simple. We just have to add up the age specific fertility rates, multiply it by the width of the age brackets (making the assumption that each age within the same age bracket has the same fertility rate), and then divide it by 1000. We divide it by 1000 because that's the most common way to measure TFR (births /1000 women).


```python
import pandas as pd 
import matplotlib.pyplot as plt

tfr = pd.read_csv(R"./resources/sweden5.csv") # read file 
tfr["TFR"] = (tfr.drop('Year', axis=1).sum(axis=1)*5)/1000 # sum up TFR
```

And literally, there we have it! The total fertility rate for each year in Sweden. And in case you're wondering how I got age specific rates, it's literally the total number of babies born to an age bracket over the number of person-years lived in that age bracket. Go back to Part 2 [here]() if you need a quick refresh on what person-years are!


```python
tfr.plot(x='Year',y='TFR')
plt.ylabel("Total Fertility Rate")
plt.title("TFR of Sweden over time") 
```

![output_3_1](https://user-images.githubusercontent.com/68678549/105466683-9142ec80-5ccf-11eb-9945-25be91a7f9fb.png)


## Key Measure 6 - Mean Age of Childbearing

### Theory

TFR gives an indication of the *quantum* of fertility - the actual number of children a woman would bear. In order to see the *tempo* of fertility (the timing of fertility), a useful measure would be the mean age of childbearing - at what ages do women have children?

### Calculations

Again, this is a relatively simple calculation to make. We essentially take the age-specific fertility rates and weight them by the mid-point of each age bracket. We then sum these up and divide it by the sum of the age specific rates. Each age bracket (in our case) is 5 years (e.g. from 15-19, 20-24) and to get the midpoint, we just have to add 2.5 to the first age of the bracket.


```python
weights = [] # create temp list for individual year weights
mac = [] # create list for mean age of childbearing

for z in range(0, len(tfr)): # go through each row
    for i in range(1,(len(tfr.columns)-1)): # then go through each column
        mid = int(tfr.columns[i])+2.5 # calculate the mid-point
        weighted = mid*tfr.iloc[z,i] # calculate the weighted ASFR
        weights.append(weighted) # append it to the temp list
    mac.append((sum(weights)/float(tfr.iloc[z,1:8].sum()))) # sum up the weighted ASFR, divide by sum of ASFR and add to yearly mac list
    weights.clear() # clear the temp list

tfr["MAC"] = mac # add MAC calculation to dataframe
```


```python
tfr.plot(x="Year", y="MAC")
plt.ylabel("Mean Age of Childbearing")
```

![output_6_1](https://user-images.githubusercontent.com/68678549/105466686-92741980-5ccf-11eb-83f6-f075f8955452.png)


We observe a steady increase in the mean-age of childbearing over time! Which aligns with the general trend in developed economics where women prioritise work and may postpone childbearing till later in their lives or careers.

## Bonus Measure - Tempo-Adjusted Total Fertility Rate

Before we end, just a point to take note of! The issue with Total Fertility Rate is that it uses a synthetic cohort. Since all the women in the respective age groups have not gone through their entire childbearing years, we are *anticipating* that a woman who is currently in her 20s will have the same fertility rate when she is in her 30s as women who are currently in their 30s (I hope this makes sense!). This assumption may not always hold true, especially when the MAC is fairly high or low. As a result of this, the TFR may either overestimate (if the MAC is low) or underestimate (if the MAC is high).  This is because...
- MAC is low/decreasing. As women progress up the age brackets, they will have fewer children than the women at those ages CURRENTLY. Hence, the TFR is overestimating how many children they will eventually have at that age 
- MAC is high/increasing. As women progress up the age brackets, they will have more children than the women at those ages CURRENTLY. Hence, the TFR is underestimating how many children they will eventually have at that age

The calculations for this require higher resolution data than what we currently have. Hence, we won't be going through the specifics of how to calculate this. However, if you're interested, you can find it [here](https://demography.subwiki.org/wiki/Tempo-adjusted_total_fertility_rate).

# Conclusion

And that's it for today! Thanks for stopping by! Next week, we'll finish off this series by looking at reproduction rates and more population projection! 
