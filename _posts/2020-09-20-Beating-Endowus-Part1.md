---
title: "Beating" an Investment Company - Part 1
categories: [trading]
date: 2020-09-20
comments: true
---

_Disclaimer: This is not directed particularly at Endowus per se or any other investment company. Endowus is simply being used as a point of reference for rate of return comparisons and benchmarking_

# Introduction

So, a couple of days ago, I got into a little squabble with my dad about what to do with some cash that I had lying around in my bank. I wanted to invest the money to generate a better interest rate than the pathetic [~0.5%](https://www.valuechampion.sg/bank-accounts/average-effective-interest-rate-savings-accounts#average) offered by most banks. My dad leaned more toward just parking my money with a random advisor and trusting them to invest in, while I was more inclined toward managing the money on my own. Unable to come to a conclusion, we decided that an experiment was in an order and decided to split the money in a 2:1 ratio, parking 2/3 of the money with [Endowus](https://endowus.com/how-we-invest) and leaving 1/3 of the money for me to invest with. 

In starting this journey, I recalled the words of Sun Tzu (yucks sorry for quoting the likely most overly quoted Chinese philosopher(is he??)) and the importance of "knowing thy self, and knowing thy enemy." Feeling that I was in touch with myself (jk), I decided to start off with first breaking down the strategy of the enemy. Here's the outline of the article and the tl;dr for you ~~lazy~~ amazing readers

### tl;dr

 1. Who is Endowus: Singapore finance company, boasting data driven solutions and proprietary systems
 2. Problems
    1. Inaccuracy/Unreliability of simulations: The accuracy of simluations/models are contingent on the accuracy of their input. Running a model 1,000 or 1,000,000 times does not increase its forecasting accuracy if the inputs are flawed to begin with. Hence, Endowus' 1,000 simulation forecasts aren't that impressive. 
    2. Historical Returns + Fees: Historical returns (which is used by Endowus for projection) are never indicative or provide certainty about future returns. Also fees affect return rates.
3. Conclusion

# Knowing thy enemy

## Who is Endowus

Endowus bills itself as a "Singapore-based ***financial technology company*** that empowers people to take control of their financial future". I've highlighted the "financial technology company" part because that's where Endowus really sells themselves highly, contrasting this 'advanced' technological approach to traditional financial advisors. Their mission(??) statement goes on to explain that they use a "proprietary system [to] provide data-driven wealth advice in constructing personalised solutions." Wow - to the ordinary folk, this all sounds really cool and (to use a Singapore colloquialism) *cheem*. Data-driven advice? Proprietary system? Sign me up BABY. 

![tenor](https://user-images.githubusercontent.com/68678549/96328568-2a31ee00-1077-11eb-9047-b0d148b85c98.gif)

When one first opens an account with them, you are given the option to set some financial goals. You put in your initial investment, your monthly contributions, and your risk tolerance (represented by maximum % drawdown). Here's where the impressive, data-magic happens. After running some **INTENSE** calculations, Endowus then spits out a projected portfolio growth graph (based off of ***1000*** simulations; yes ONE THOUSAND). Here's how it looks:

![Endowus_GoalProjection](https://user-images.githubusercontent.com/68678549/96326219-5478b100-1061-11eb-931d-6bfde8cbb09c.gif)

Pretty fancy no? So, this supposedly data-driven approach appears to convey to potential investors that- Hey! We're using some cutting edge technology to manage your money and hence you can trust us with it. But here's the thing... is it really that impressive? Let's find out.

## Problems

### Inaccuracy of simulations

First off, running a 1000 simulations sounds really cool. I mean, if we tried it a 1000 times, surely it has to be accurate right? But let's strip that back a bit and think about what this means. Running a 1000 simulations to get a range of possibilities could be considered a **Monte Carlo Simulation**. This is defined as any "model used to predict the probability of different outcomes when the intervention of random variables is present". Essentially, it allows you to use repeated sampling to account for underlying randomness. 

Is it really that 'advanced' of a concept? Sure, it sounds and looks impressive but it's actually easily replicable. So, I created a function to  estimate returns, given expected returns and standard deviation (this, as we will discuss later, is also another key issue with the simulation)

```python
import random
import numpy as np
import matplotlib.pyplot as plt
import scipy.stats as stats
import seaborn as sns
import math

def EstimateReturns(initial_invest, expected_returns, std_dev, duration):
    final_money = []
    yearcount = []
    
    year = 1
    
    for year in range(duration):
        this_return = np.random.normal(expected_returns,std_dev)
        initial_invest = initial_invest * (1+this_return)
        final_money.append(initial_invest)
        yearcount.append(year)
        
    plt.ylabel('Amount of $')
    plt.xlabel('Number of years')
    plt.plot(yearcount,final_money)
    
    final_funds.append(final_money[-1])
    return(final_funds)
```

After creating the function, I ran a loop on it, allowing users to input their initial investment, yearly rate of return, and standard deviation (as per the Endowus simulation).

```python
final_funds = []

investment_start = int(input("how much is your initial investment?\n"))
RORY = float(input("What is the yearly rate of return?\n")) 
YearStd = float(input("What is the standard deviation of return\n"))
YearC = int(input("How many years are you investing for?\n"))
simulation_no = int(input("how many times to simulate?\n"))

np.random.seed(123)

for i in range(simulation_no):
    ending_fund = EstimateReturns(investment_start,RORY,YearStd,YearC)

average_made = str(sum(ending_fund)/len(ending_fund))
average_made
```

For my simulation, I used the yearly rate of return and standard deviation for their 60/40 portfolio - a conservative but growth orientated strategy. 

![image](https://user-images.githubusercontent.com/68678549/96326391-c69dc580-1062-11eb-9ff4-a3aad62e86f2.png)

![image](https://user-images.githubusercontent.com/68678549/96326418-ff3d9f00-1062-11eb-86e9-4043efce35df.png)


    how much is your initial investment?
    10000
    What is the yearly rate of return?
    0.0771
    What is the standard deviation of return
    0.0870 #I used 8.78% because this was an updated figure I got from their website
    How many years are you investing for?
    30
    how many times to simulate?
    1000

After putting in my settings (following the given rate of returns), I then calculated the 10th percentile, the median, and the 75th percentile of the portfolio (mimicking the results given by the Endowus' simulation)

```python
port_std = round(np.std(ending_fund), 2)
port_median = round(np.median(ending_fund), 2)
port_10per = round(np.percentile(ending_fund, 10), 2)
port_75per = round(np.percentile(ending_fund, 75), 2)
port_max = round(np.max(ending_fund), 2)
port_min = round(np.min(ending_fund), 2)

print("The median return of the portfolio is %s.\nThe 10th percentile is %s and the 75th percentile is %s.\nThe highest amount is %s, and the lowest is %s." % (port_median,port_10per,port_75per,port_max,port_min))
```

    The median return of the portfolio is 71657.24.
    The 10th percentile is 38651.81 and the 75th percentile is 96055.78.
    The highest amount is 438889.69, and the lowest is 16925.8.

![image](https://user-images.githubusercontent.com/68678549/96326564-4ed09a80-1064-11eb-924f-67391b7be688.png)

Comparing my results to Endowus' (pictured below), I must say that it was pretty close. My simulation overshot the median by ~$7000, but this was likely because I didn't take into account fees and stuff. The top 25% also overshot by ~$6000, but the bottom 10% was nicely similar with only a ~$2000 difference. All in all, quite satisfied with the result.  

![Endowus' return](https://user-images.githubusercontent.com/68678549/96326626-c56d9800-1064-11eb-8970-dd2d25891c08.png)

At this point, you might argue, "Zach! See? It's so accurate! What's not to like about Endowus?" Well, that brings me to the issue with models. The output of models depend entirely on the INPUT of the model. This may seem obvious but that's exactly the problem with these fancy-shamancy things that tell you they ran 1000 simulations. I can run 1,000,000 simulations (I did in fact do that LOL) and it wouldn't increase my accuracy one bit if my input was completely wrong. For instance, here's me running the model again but with different inputs (and simulating it 10,000 times). 

```
    how much is your initial investment?
    10000
    What is the yearly rate of return?
    0.7
    What is the standard deviation of return
    0.0781
    How many years are you investing for?
    30
    how many times to simulate?
    10000
```
```
    The median return of the portfolio is 79830026990.74.
    The 10th percentile is 57374377254.75 and the 75th percentile is 94655289488.97.
    The highest amount is 225225174502.04, and the lowest is 34068119785.18.
```

![image](https://user-images.githubusercontent.com/68678549/96326786-39f50680-1066-11eb-929f-38dda6c3a567.png)

Did me running the model 10,000 times make it any more accurate? NO! If my input is flawed (in this case, a 70% annual return), my output will still be flawed, regardless of how many simulations I ran. So, these things look nice and fancy but they cannot be markers of "accuracy" because they are heavy reliant on the input. Which brings me to my next point... 

### Historical Returns + Fees

Endowus, as a company, has only been around since 2017 and hence the rate of returns projected come from the [historical returns of the funds that Endowus invests in](https://endowus.com/how-we-invest). The projected rate of returns, when graphed, look like this:

```python
# code from https://moonbooks.org/Articles/How-to-plot-a-normal-distribution-with-matplotlib-in-python-/

fig, ax = plt.subplots()
x_min = 4
x_max = 12

mean = 7.71
std = 0.8

x = np.linspace(x_min, x_max)

y = stats.norm.pdf(x,mean,std)

plt.plot(x,y, color='black')

# truncated code with the "fill area code removed"

plt.xlim(x_min,x_max)
plt.ylim(0,0.6)

plt.title('Endowus Rate of Return for 60/40 Portfolio',fontsize=10)

plt.xlabel('Rate of Return')
plt.ylabel('Probability Density')

plt.show()
```

![image](https://user-images.githubusercontent.com/68678549/96327479-90fdda00-106c-11eb-8b56-adf5ffa32f42.png)

With a standard deviation of 8.7%, what this means is that there is a  68% chance (one standard deviation), that your yearly rate of returns will be between 6.9% and 8.4%. Endowus, as a principle, does not 'stock pick' but instead helps clients to diversify by buying into equity and bond ETFs/mutual funds (and charging a management fee for intelligent rebalancing). The projections are hence not of Endowus' portfolio's historical return, but an amalgamation (with weighting) of the historical returns of these funds! And we all know the issue with using historical returns... As David Blanchett (head of retirement research for Morningstar Investment Management) [notes](https://www.advisorperspectives.com/articles/2014/08/26/the-power-and-limitations-of-monte-carlo-simulations), using historical returns to gauge future returns "implicitly incorporates the recurrence of historical events and allows for only a limited amount of data". Furthermore, he argues that "a problem with Monte Carlo tools that is exacerbated today is that they can often paint an unrealistic picture of returns", giving a semblance of certainty when there is, in fact, "considerably less certain[y]". 

Another potential factor that will affect returns are fees. I understand that every investment has to charge fees. After all, someone *has* to be paid to keep the company running. Endowus compares their 0.6% fee (for below $200,000 portfolios) to the industry average of about ~2%. While this is surely lower, we have to remember once again that Endowus really isn't an investment company per se. Rather, they allocate money into funds and the fund managers are the ones that invest the money. These funds *also* charge management fees (which add up to ~0.5%).  

## Conclusion

So, where does all this leave us? My intent in writing this was not to discredit Endowus or discourage anyone from investing with them. Rather, by dissecting the 'fanciful' technology behind the company, I wanted to bring out two key points.

1. First, be wary of nice visuals and *cheem* sounding things. Running a 1000 simulations sounds really cool but you have to remember that the forecasting output of a model is only as accurate as its input. If the output is inaccurate, a 1000 or a million simulations really won't make that much of a difference to the model's accuracy.
2. Second, historical returns do provide certainty for future returns. We can definitely use historical returns to approximate what future returns will look like BUT we need to remember that there is certainly no certainty, no matter how nice the graphs look. 

So, I'm not knocking Endowus. They certainly lower fees and do appear to have sound strategies of diversification etc. BUT, as a potential investor, just be aware that there are limitations to the beautiful data-driven models that they present as their 'edge'. 

## Personal Conclusion

Having examined Endowus, the next step was to then think of ways to beat their returns over the course of the next year. After all, as a smaller capitalized investor answering to no one, I was surely more nimble and could afford higher risk higher return strategies that could beat them. With this, I decided to turn to Options... which you can read all about in [Part 2](). 







