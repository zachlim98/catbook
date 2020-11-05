---
title: Predicting Singapore Suicide Rates using Macro sociocultural variables
categories: [data]
---

# Introduction

Movember has arrived, bringing with it... well just your typical wet and humid Singapore weather (not the beautiful leaves of autumn as one would expect in the Northern Hemisphere). With me being an Asian male and hence having no chance of growing anything that could passably be called a moustache, I decided to contribute in my own way by examining suicide rates (both male and female) and trying to figure out if there was any way we could **predict suicide rates by looking at macro-societal factors.** 

<img src="https://images.unsplash.com/photo-1541670317641-d8294a62294c?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1876&q=80" alt="Not me" style="zoom: 10%;" />
*Pictured: Not me in Movemeber (Image from Unsplash)*

There has been much pre-existing research on this (see for instance [here](https://pubmed.ncbi.nlm.nih.gov/25599279/) and [here](https://www.ncbi.nlm.nih.gov/books/NBK223752/)) and one of the main conclusions drawn was the relationship between macroeconomic instability and the "exacerbation of mental disorders and suicide". However, I wanted to go beyond economic measures and look at random, possibly unrelated variables to see if there was any "outside-the-box" factor that we could possibly look to to predict suicide rates (and act accordingly). 

# tl;dr

1. **Poor Models:** I tried applying a Random Forest to the data and then a Linear Regression model. However, both turned out to have extremely low R^2 values, indicating poor predictive power (when tested on the validation set). 
2. **Lessons Learnt:** 
   1. **Data is King**: No matter the model you use, the data that you work with is key. As the saying goes, garbage in, garbage out. As much as I hated to admit it, I simply had insufficient data. Most of the datasets I had downloaded from SingStat only went as far back as 1990. The data was just too little for any meaningful work to be performed on it
   2. **Nature of Dependent Variable:** Suicide, as a function of mental health, has been traditionally difficult to manage, much less predict. Thinking that I could use macro variables to somehow predict suicide rates was a good but perhaps naïve idea.  
3. **Raising Awareness is Key:** At the end of the day, perhaps the best tool we have against this deadly trend in society is awareness and empathy. Removing the stigma of suicidal ideation and affording people spaces to receive help are key in combating this. As individuals, we can equip ourselves with the right knowledge and skills to be there for friends and family who may be suffering mentally. [This](https://www.sos.org.sg/learn-about-suicide/quick-facts) is a good website with key information on how we can play our part.  

# Exploration

## Gathering Data

Wanting to do a Singapore-specific exploration, I turned to 2 repositories of Singapore statistics: [SingStat](singstat.gov.sg/) and [Data.gov](data.gov.sg/). The datasets on these websites are really quite extensive, covering a wide variety of statistics. I downloaded a whole bunch of random stats from these websites, compiling them into a master csv file. 

While the suicide rate statistics ran all the way back to 1968, most of the other key statistics only ran to 1990. I was rather disappointed to find this as I'd always thought that Singapore, of all countries, would have good, reliable historical statistics (given our relatively young age as a country). Nonetheless, I worked with what I had and limited my dataset to 1990 - 2019. This would later prove to be to my detriment. 

From the huge array of datasets available online, I chose a bunch of random, uncorrelated datasets. There were a lot more variables that I wanted to include (random things like number of visitors at public swimming pools) but most only had data from 2005 (and some only from 2010!). There were also important consumer consumption variables that I wanted to include. Alcohol use (as measured by consumption and alcohol-related crimes) had been significant factors in other studies. However, the crime statistics only started from 2011 and the consumption statistics were done only once every 5 years. Not particularly useful. 

Nevertheless, I worked with what I could find and ended up with 14 different variables: Unemployment Rate, Rainy Days, Hours of Bright Sunshine, Marriage Rate, Number of Visual Arts Displays, Number of Mobile Phone Subscriptions, Divorce Rate, Number of Library Book Loans, Spending on Polytechnic students, Spending on University students, Total Vehicle Population, Number of Newspaper Subscriptions, Number of Road Accidents, and Rates of Electrical Consumption. 

<img src="https://images.unsplash.com/photo-1484480974693-6ca0a78fb36b?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1052&q=80" alt="Lists of many variables" style="zoom:25%;" />
*If only I had more variables... sigh (Image from Unsplash)*

I thought it was quite a broad list and was quite excited to run it through the models to see what we could find. Thankfully, the data downloaded was quite clean and with a little bit of 'pre-processing' in Excel, we were ready to go. 

## Data Peek

Having cleaned up the data, I imported it into Python and graphed out the suicide rate over the years as well as the correlation plot just to get a quick sense of the data. 

```python
#import libraries
import numpy as np
import matplotlib.pyplot as plt
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt
```


```python
#import datasets
dataset = pd.read_csv('suicide_rate.csv')
X = pd.DataFrame(dataset.iloc[:, 12:])
y = dataset.iloc[:, 1].values
```


```python
plt.plot(dataset["Year"],dataset["Suicide_Rate"],color='black')
plt.xlabel('Year')
plt.ylabel('Suicide Rate')
plt.title('Suicide Rate 1990 - 2019')
```

![image](https://user-images.githubusercontent.com/68678549/97960264-15ca5100-1dec-11eb-8a1b-235d5ce9e56b.png)

Looking at the graph of the suicide rate, it honestly did not look promising. There seemed to be no real trend (either up or down) throughout the years and while there appeared to be spikes at key moments (e.g. 2007 - 2008 financial crisis), they weren't particularly prominent either. 

```python
#Create labels for x axis
x_axis_labels = ["Unemployment Rate","Rainy Days","Bright Sunshine (hr)","Marriage Rate","Visual Arts Displays","Mobile Phone Sub","Divorce Rate","Library Loans", "Polytechnic Spending","University Spending","Vehicle Population","Newspaper Subscription","Road Accident","Electrical Consumption", "Suicide Rate"]

#Create correlation and mask
corrp = X.assign(target = y).corr().round(3)
mask = np.triu(np.ones_like(corrp, dtype=bool))

#Fix size of figure
plt.figure(figsize=(15,15))
#Plot the heatmap, using mask to create diagonal plot
sns.heatmap(corrp, mask=mask, cmap = 'coolwarm', annot = True, fmt=".2f", xticklabels=x_axis_labels, yticklabels=x_axis_labels).set_title('Correlation matrix', fontsize = 16)
```

![filename](https://user-images.githubusercontent.com/68678549/97960674-de0fd900-1dec-11eb-9047-5fa94656ca9a.png)

Looking at the heatmap, it honestly did not look promising. From the last row, none of the variables had a particularly strong correlation (positive or negative) with suicide rate. Even for unemployment rate (which had been shown in other papers to have a significant positive correlation), there didn't appear to be a particularly strong correlation. 

## Random Forest Regressor

Undeterred (or maybe I should have been deterred) by the overview of the data, I pushed ahead with implementing a Random Forest model to try to make sense of the data. Using `sklearn`, I split the data into a training and test set and then instantiated the RFG. 

*(Note: I recognize that there are different schools of thought with regard to splitting the dataset for RFs. Some argue that because of bagging, there is no need for a validation set for RF. However, there are others who argue that this can be compared to the OOB score for better evaluation. I also needed a test and train set for my Multiple Linear Regression model later hence the splitting)*

```python
from sklearn.model_selection import train_test_split
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size = 0.25, random_state = 42)
```


```python
from sklearn.ensemble import RandomForestRegressor
regressor = RandomForestRegressor(n_estimators = 1000, random_state = 42, oob_score=True, max_features=0.33)
regressor.fit(X_train, y_train)
```
Fitting the training dataset, I then compared the R^2 scores of the models. The importance of R^2 has been discussed [previously](https://zachlim98.github.io/me/2020-10/sgcarmart1) but in essence it represents the proportion of variance of the dependent variable explained by the independent variables. Or in (slightly incorrect) layman's terms, it represents how much of the outcome can be explained by the independent variables. 

```python
print('R^2 Score: {:.2f} \nR^2 OOB Score: {:.2f} \nR^2 Validation Score: {:.2f}'.format(regressor.score(X_train, y_train), regressor.oob_score_, regressor.score(X_test, y_test)))
```
```
    R^2 Score: 0.80 
    R^2 OOB Score: -0.54 
    R^2 Validation Score: 0.36
```
The end product was quite the disaster. While the training R^2 was a respectable 0.80 (80% of suicide rates can be explained by the inputs), the Out-Of-Bag score was **NEGATIVE** and the test set score of 0.36 was so far away from the OOB score. 

RF models typically have high R^2 scores so the score of 0.80 was nothing to write home about. While I initially intended to perform hyperparameter tuning on the model using `sklearn RandomizedSearchCV`, the initial evaluation of the model was already **so poor** that I didn't think any tuning would make sense. ***N***  (the number of data points) was simply TOO LOW. As Will Koehrsen writes in his [article](https://towardsdatascience.com/hyperparameter-tuning-the-random-forest-in-python-using-scikit-learn-28d2aa77dd74) on improving RF models, the first step is to gather more data (which in this case I unfortunately could not).

Just out of curiosity, I wanted to see the feature importance (i.e. which features were most useful in predicting the target). I used [Eryk Lewinson](https://medium.com/@eryk.lewinson)'s code in creating the plot for feature importance and tweaked it to fit the data.  

```python
def imp_df(column_names, importances):
    df = pd.DataFrame({'feature': column_names,
                       'feature_importance': importances}) \
           .sort_values('feature_importance', ascending = False) \
           .reset_index(drop = True)
    return df

fe_imp = imp_df(X_train.columns, regressor.feature_importances_)

fe_imp.columns = ['feature', 'feature_importance']
ax = sns.barplot(x = 'feature_importance', y = 'feature', data = fe_imp, orient = 'h', color = 'red')
ax.set(xlabel="Feature Importance",ylabel="Features",title="Feature Importance")
```

![image](https://user-images.githubusercontent.com/68678549/97969342-e1aa5c80-1dfa-11eb-99ac-f88c64a6ad42.png)

Just to prove how inaccurate the model was, the amount of government spending on Polytechnics was the most important feature. This was followed by the number of Visual Arts Exhibitions per year. While the second one made second, it was hard to see how spending on Polytechnics could have been the most important variable in predicting suicide rates. The unemployment rate (commonly shown to be quite a strong predictor) was 7th most important. This model was clearly quite broken. 

## Multiple Linear Regression

By this point, I'd realised that my lack of data was a huge problem. However, since I was already in Python, I decided that I might as well finish up the exploration and throw in the LR model. 

```python
import statsmodels.api as sm
T = sm.add_constant(X_train)
model = sm.OLS(y_train, T).fit()
print_model = model.summary()
print(print_model)
```
```
                                      OLS Regression Results                            
    ==============================================================================
    Dep. Variable:                      y   R-squared:                       0.625
    Model:                            OLS   Adj. R-squared:                 -0.125
    Method:                 Least Squares   F-statistic:                    0.8329
    Date:                Tue, 03 Nov 2020   Prob (F-statistic):              0.636
    Time:                        17:04:05   Log-Likelihood:                 12.780
    No. Observations:                  22   AIC:                             4.440
    Df Residuals:                       7   BIC:                             20.81
    Df Model:                          14                                         
    Covariance Type:            nonrobust                                         
    =========================================================================================
                                coef    std err          t      P>|t|      [0.025      0.975]
    -----------------------------------------------------------------------------------------
    const                    -0.9566      5.870     -0.163      0.875     -14.836      12.923
    Unemploy_Rate            -0.1598      0.366     -0.436      0.676      -1.026       0.706
    No_Rainy_Days            -0.0045      0.005     -0.989      0.355      -0.015       0.006
    Hours_Bright_Sunshine    -0.2057      0.325     -0.633      0.547      -0.974       0.563
    Crude_Marriage_rate       0.2403      0.327      0.735      0.486      -0.533       1.013
    VA_Rate                  -0.4851      3.317     -0.146      0.888      -8.329       7.359
    MPSub_Rate                1.6190      0.786      2.060      0.078      -0.239       3.478
    Divorce_Rate             -0.2286      1.113     -0.205      0.843      -2.861       2.404
    LBSub_Rate               -1.5025      1.142     -1.316      0.230      -4.203       1.198
    Poly_Spend               -0.0005      0.000     -2.199      0.064      -0.001    3.79e-05
    Uni_Spend             -9.098e-06      0.000     -0.086      0.934      -0.000       0.000
    Veh_Pop_Rate             17.0992     15.874      1.077      0.317     -20.436      54.634
    NewsP_Rate                0.1933      2.529      0.076      0.941      -5.787       6.173
    RoadA_Rate               -4.0852     18.256     -0.224      0.829     -47.253      39.083
    Elec_Cons_Rate          994.4616    516.944      1.924      0.096    -227.918    2216.841
    ==============================================================================

    Notes:
    [1] Standard Errors assume that the covariance matrix of the errors is correctly specified.
    [2] The condition number is large, 2.13e+08. This might indicate that there are
    strong multicollinearity or other numerical problems.
```

As can be seen from the results, the adjusted R-squared was **negative** (which essentially means 0 - i.e. the model is unable to predict any of the variance of suicidal rate). This was hence a basically useless model. 

<img src="https://images.unsplash.com/photo-1515472071456-47b72fb3caff?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=634&q=80" alt="I feel this" style="zoom: 33%;" />
*I feel like this balloon (Image from Unsplash)*

# Conclusions

1. **Data is King**: No matter the model you use, the data that you work with is key. As the saying goes, garbage in, garbage out. As much as I hated to admit it, I simply had insufficient data. Most of the datasets I had downloaded from SingStat only went as far back as 1990. The dataset was just too small for any meaningful work to be performed on it. [This article](https://towardsdatascience.com/hyperparameter-tuning-the-random-forest-in-python-using-scikit-learn-28d2aa77dd74) is a resource on talking about improving ML models. And the first thing it recommends is to get more data to train your models on. In this case, getting more data was not possible given my limitations in finding Singapore-related statistics. 
2. **Nature of Dependent Variable:** Suicide, as a function of mental health, has been traditionally difficult to manage, much less predict. Thinking that I could use macro variables to somehow predict suicide rates was a good but perhaps naïve idea. There have been other studies that have sought to use micro-variables (such as looking at [twits](https://www.nature.com/articles/s41746-020-0287-6) to detect suicidal ideation). While they did find "significant associations of algorithm SI scores with county-wide suicide death rates", it remains to be seen if this will be useful in *predicting* suicidal tendencies. 
3. **Raising Awareness is Key:** At the end of the day, perhaps the best tool we have against this deadly trend in society is awareness and empathy. Removing the stigma of suicidal ideation and affording people spaces to receive help are key in combating this. As individuals, we can equip ourselves with the right knowledge and skills to be there for friends and family who may be suffering mentally. [This](https://www.sos.org.sg/learn-about-suicide/quick-facts) is a good website with key information on how we can play our part.  

