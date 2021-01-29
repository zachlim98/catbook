---
title: Exploring Demography with Python - Part 4
categories: [coding, data]
---

# Introduction

Welcome back to the fourth part of our demographic functions series! Thus far, we've looked at measures of fertility and childbearing. But... the issue with the TFR is that it does not account for mortality and hence doesn't actually give a good idea of measures of reproduction and replacement. Today, we'll be looking at the **Gross Reproduction Rate** and **Net Reproduction Rate**, which will allow us to do **Population Projection**

## Key Measure 6 - Gross and Net Reproduction Rate

### Theory

Since females are the only ones that can reproduce naturally, we usually take the Reproduction Rate as benchmarked against the number of females. Hence, the Gross Reproduction Rate (GRR) is defined as the size of the cohort of daughters relative to the cohort of mums. To get this, we essentially multiply the TFR against the ratio of female births over total births. This is usually assumed to be *0.49*, 105 male births for every 100 female births. Another way to calculate GRR is as the sum of age-specific fertility rates, only considering daughters -  we will use this method later for calculations.

However, Gross Reproduction Rate does not take into account mortality and mortality would affect the reproduction potential of the next generation. We hence have to use the Net Reproduction Rate (NRR) which takes into account mortality levels. The NRR, hence, quantifies how a population will growth (NRR > 1 will lead to population growth and NRR < 1 will lead to a decline in population). 

### Calculations

The calculations aren't that difficult. We essentially have to transform age-specific fertility rates to *female-only* and then sum it up to get GRR. We can then multiply this by mortality to get NRR. Let's get to it!


```python
import pandas as pd
import numpy as np 

brazil = pd.read_csv(R"./Resources/brazil.csv").replace(to_replace=0.00,value=np.nan).dropna() #import data file and get rid of non-reproductive ages
brazil["daughter_ASFR"] = brazil["Age-specific fertility rate"]*(100/(105+100)) #convert to daughter-specific ASFR by multiply by sex ratio at birth 

GRR = brazil["daughter_ASFR"].sum()*5/1000 #multiply by 5 since 5 ages per bracket, and per thousand
print("Gross Reproduction Rate is: {:.2f}".format(GRR))
```

    Gross Reproduction Rate is: 0.91


To get NRR, we have to take into account mortality. Let's assume here that we are told that the lx (the radix used) is 100,000. Hence, for each age bracket, the number of person-years are 500,000 (1x 100,000) and we can find mortality by taking the Lx (the person-years lived in that age bracket) over the original number of person-years.

*A segue here: On the coding side of things, I faced an interesting issue. I initially couldn't multiply by the Lx column because Python had read it as an object. I hence had to convert it to an integer first.*


```python
brazil["Lx"] = brazil["Lx"].astype('str').str.replace(' ','').astype('int')
brazil["daughter_ASFR_mort"] = brazil["daughter_ASFR"] * (brazil["Lx"]/500000)

NRR = brazil["daughter_ASFR_mort"].sum()*5/1000 #multiply by 5 since 5 ages per bracket, and per thousand
print("Net Reproduction Rate is: {:.2f}".format(NRR))
```

    Net Reproduction Rate is: 0.88


## Key Measure 7 - Intrinsic Rate of Natural Increase (r)

### Theory

Using the NRR, we can also calulcate the intrinsic rate of natural increase (r) or the population growth rate. This is easily approximated by taking the ln of NRR and dividing it by the mean age of childbearing. The mean age of childbearing can also be thought of as the mean length of a generation because it represents the length of time that the next generation will begin childbearing. Hence, we divide it by the length of the generation to find out the intrinsic rate of natural increase. 

### Calculations

The culculations are not difficult because we've essentially done everything before already. Remember the MAC from the previous post? Well, let's just do that all over again. We first calculate the total number of births and then divide it by the daughter ASFRs (accounted for mortality) to get the average age of birth.


```python
brazil["Tot_birth"] = brazil["daughter_ASFR_mort"] * (brazil["age"]+2.5)
MAC = sum(brazil["Tot_birth"])/sum(brazil["daughter_ASFR_mort"])
r = np.log(NRR)/MAC
print("The intrinsic rate of natural increase is {:.4f}".format(r))
```

    The intrinsic rate of natural increase is -0.0047


# Conclusion 

And... that's it for the fourth part of this series. In the final part next week, we will look at putting all of these together in order to project a population forward! Stay tuned for it and thanks for reading!
