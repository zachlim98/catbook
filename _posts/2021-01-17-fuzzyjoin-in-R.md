---
title: Using fuzzyjoin to help in your analysis of pesky Online Survey Data
categories: [data]
---

# Introduction

A few days ago, I was asked to help someone collate results from an online survey. The problem was, the online survey had been done in two parts, each with its own Google Form. The request was to perform a simple linear regression model with the response variable recorded in one of the Google Forms, and the explanatory variables recorded in the other Google Form. However, there was no unique identifier for each survey participant and the only thing that they had been told to input were their names. This was going to be fun! Here's a record of how I did it (and the thought process behind it) using `fuzzyjoin` in R. 

## The Data

In order for you to have a good idea of how poor the data was, here's a snapshot of how the names had been recorded in both forms (names have been changed because this is personal data but this was essentially how it looked). The first set of data recorded their overall happiness...

![image](https://user-images.githubusercontent.com/68678549/104834611-9113ad00-58db-11eb-8b43-3250cc191624.png)

and the second set of data recorded the difficulties they faced at work and with their children. The second set of data had been done on Google Forms but a drop-down list was used for the names. This forced the names to be in a set format, in accordance with their registered names. 

![image](https://user-images.githubusercontent.com/68678549/104834701-244ce280-58dc-11eb-9a20-9f797300a359.png)

All 4 names are present in both sets of data but as you can see, there was quite a bit of discrepancies in how the names were entered. For instance, in Singapore, many folks from an older generation only had registered Chinese names (e.g. "Kwek Ting Rei") but had adopted English names (e.g. "Mary Kwek") that were not registered with the government. This was proving to be quite the problem. 

## Enter Fuzzyjoin

The person that I was working with was familiar with R and hence had asked me to do the project in R. Looking at the various libraries available, I eventually decided upon `Fuzzyjoin` as the best way forward. 

```R
library(dplyr)
library(fuzzyjoin)

happiness <- read.csv("rlsphap.csv") %>% 
  rename(Name = Name.of.Parent)
  
difficulties <- read.csv("diff.csv")
```

After importing the files, I used the `stringdist_join` function found in the `fuzzyjoin` library to perform the matching and combination of the two files. I'll put the entire block of code that I used, before going on to explain what each step was for.

```R
combined  <- stringdist_join(happiness, difficulties,
                by = "Name",
                mode = "left",
                method = "jw",
                ignore_case = TRUE,
                max_dist = 0.5,
                distance_col = "dist")
```

The first two arguments, `by` and `mode` are similar to the usual `dplyr` joins so I won't really touch on those. 

The first important argument is `method`. In this case, method allows you to choose which metric you want to use in order to quantify the distance between word strings. You can find the full list of metrics [here](https://www.rdocumentation.org/packages/stringdist/versions/0.9.6.3/topics/stringdist-metrics). I was not really familiar with **all** of the metrics but I knew that a few (for instance **Hamming**) were definitely out because they required the text strings to have the same number of characters. Instead, in order to figure out which method gave me the best result, I created a subset of the dataset and just played around with a few methods until I found one that I thought (from eyeballing the results) looked the best. As you can see, I eventually settled upon **Jaro distance**. It gave me the best results after some trial and error. 

The next argument is `ignore_case` which I obviously set to "TRUE" because some names were capitalized while others were not. 

`max_dist` then sets the maximal allowed distance. In this case, because I used "JW" which returns distances between 0 and 1, I chose somewhere in between with 0.5. Again, this would involve a little bit of trial and error in order to find an optimal distance. The greater the `max_dist`, the greater the number of matches it will return and the greater the size of the returned dataframe. In my subset of **only 25** names, setting it to 1.0 returned 1078 rows while setting it to 0.5 gave me 564 rows. 

`distance_col` allows you to create a column that stores the distance between matches - a very useful column to have in order to filter the matches later. 

```R
combined <- combined %>%
  group_by(Name.x) %>%
  slice_min(order_by = dist, n = 1) %>% 
  arrange(desc(dist)) %>%
  filter(dist < 0.27)
```

Once I had my 564 rows, I grouped them by Name.x (which was the Name column from the first dataset - the one with the "official names") and used `slice_min` in order to keep only the row with the lowest distance between each name. 

I then arranged all the names in descending order to look at the names with the highest distance. I had been told that not every survey participant had done **both** surveys and hence some names would have to be removed because there were in fact no matches at all. Quickly scanning through the names, I saw that the first "real match" began only after the distance dropped below 0.27 and hence added a `filter`. 

# Conclusion

And... that was done! I had taken two rather messy sets of names from two disparate datasets and joined them together using `fuzzyjoin` (and a lot of trial and error). Thanks for reading!