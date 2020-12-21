---
title: Using R's Tidytext package to analyse Covid-19 speeches (in a R-approved tidy way!)
categories: [data]
---

# Introduction

I learnt about [tidytext](www.tidytextmining.com/) awhile ago and thought it was such a neat framework. The essential idea behind it was that you could perform text mining/sentiment analysis using `dplyr` and "tidy" dataframes. So, I just **had** to try it out! I looked around for awhile to find things to perform this exploration on and finally settled on Singapore Prime Minister Lee's Covid-19 related speeches - thought it might be fun to see how that changed over time. As always, a tl;dr to get things going. 

(*a side note here: this was a rather basic and easy introduction to tidytext but it really is a much more powerful package and a lot more (which I plan to write on when I have more time) can be done with it*)

# tl;dr

1. There was a change from negative to positive sentiment from February to December (with the release of the vaccine)
2. The top 5 words used in each speech actually gave quite a good summary of the speech and the core issues 
3. Tidytext is a really intuitive and easy to use framework and should definitely be explored in more depth!

# Exploration

## Data Gathering

It was quite easy to get the data since all of the speeches had been nicely organised and transcribed (in true Singapore fashion!). If you're interested, the first one can be found [here](https://www.pmo.gov.sg/Newsroom/PM-Lee-Hsien-Loong-on-the-Novel-Coronavirus-nCoV-Situation-in-Singapore-on-8-February-2020) and the rest are just in the search pages after that. I initially wanted to use `beautifulsoup` and `selenium` to scrape it but... the web addresses were not nicely organised by date and weren't numbered properly but instead used full titles. Given that there were only 8 speeches between Feb and now, I decided to do it manually. Granted, there were probably better ways of doing it and I would likely have tried harder if there had been more speeches. 

But, given that there were only 8 speeches, I just copied and pasted each one into its own little .txt file and that was done!

## Initial Cleanup 

Next, I had to clean the data and this was where `tidytext` really came through. The tidy way of storing the data made it really easy to clean the data since I could manipulate it as I would a normal tidy dataframe - pretty neat!

```R
library(readr)
library(stringi)
library(tidytext)
library(tidyr)
library(ggplot2)
library(ggthemes)

#import list of stop words
data("stop_words")

#create list of speech file names
speech_files = list.files(path = "./Speeches")

#create tidy dfs of speeches
for (i in 1:length(speech_files)) {
  speech <- read_lines(paste0("./Speeches/",speech_files[i])) %>% stri_remove_empty()
  speechdf <- tibble(line = 1:length(speech), sentence=speech) %>% 
    unnest_tokens(word,sentence) %>%
    anti_join(stop_words)
  
  assign(paste0("df",i),speechdf)
}

#bind all words together
#label the dfs with dates before binding
binded_words <- bind_rows(mutate(df1, date=as.Date("08-02", format="%d-%m")),
                          mutate(df2, date=as.Date("16-02", format="%d-%m")),
                          mutate(df3, date=as.Date("12-03", format="%d-%m")),
                          mutate(df4, date=as.Date("03-04", format="%d-%m")),
                          mutate(df5, date=as.Date("10-04", format="%d-%m")),
                          mutate(df6, date=as.Date("21-04", format="%d-%m")),
                          mutate(df7, date=as.Date("23-06", format="%d-%m")),
                          mutate(df8, date=as.Date("14-12", format="%d-%m")))
```

Most of this step is self-explanatory. I basically imported the bunch of libraries that I would have to use and also imported a list of stop words. Then I got the names of all the files in the folder where the speeches were kept. Following which, I used a `for` loop to import each of the files and create a `tibble` for every speech. 

I used `stri_remove_empty` because there was somehow an empty line between each line of the speech. I then used `unnest_tokens` which is really the core of the whole tidytext framework. This command facilitates the tokenisation of the words (i.e. breaking them up into individual words) and also does a whole bunch of clean-up (e.g. removing punctuation, changing everything to smaller caps etc.). This would obviously not be the most useful thing in certain circumstances (for e.g. when an exclamation mark or question mark could provide good context clues) but the default settings were fine for our basic usage. 

Next, I used an `anti_join` to remove all the "stop words" - words in the English language which functioned as filler or connecting words (i.e. probably did not provide much important meaning). I then used `assign` to create a new df for each of the speeches. Finally, I bound all the speeches together in one giant df, giving them all a new column of properly formatted dates.

With that, it was time for the fun bits!

## Top 5 words across speeches

I thought this would be interesting to look at because it would give a sense of which words were used the most over the course of the year. When the graph was out, it was also interesting because it actually gave a relatively good summary of the speech and the important points within that speech. 

```R
#compare word counts across speeches
count_words <- binded_words %>% 
  #remove all numerics in the words
  mutate(word = stri_extract(word, regex = "[a-z']+")) %>% na.omit() %>%
  count(date,word) %>%
  #count proportions instead of raw count to account for length of speech
  mutate(proportion = n / sum(n)) %>%
  select(-n) %>% 
  group_by(date) %>%
  #arrange the words by proportion used
  arrange(desc(proportion), .by_group = TRUE) %>%
  #extract top 5 words used
  slice(1:5) %>%
  #add in column for graph highlighting
  mutate(tohigh = ifelse(proportion == max(proportion), "Yes", "No"))
```

Most of this is pretty self-explanatory. I used regex to and `stri_extract` to ensure that only words (and not random symbols) were extracted and dropped all the empty rows. I then used `count` to provide a count of each word, grouping it by date. Next, I created a proportion column as recommended by [Silge and Robinson](https://www.tidytextmining.com/index.html) since comparing actual counts would be misleading if comparing between a very long and a very short speech.

I then grouped it by dates before arranging it by proportion, making sure to set `.by_group` to TRUE ensure that it would sort it within the groups. Finally, I sliced out the top 5 words from each group. I also created a new "tohigh" column to help with graphing later on as I wanted to highlight the top word used. To achieve this, I used `ifelse` and `max()` to find the highest proportion value and then set the tohigh column to "Yes". 

```R
#plot graph using "reorder_within" to get bars to line up nicely
count_words %>% ggplot(aes(y=proportion, x=reorder_within(word,-proportion,date), fill=tohigh)) +
  scale_x_reordered() +
  geom_bar(stat="identity") +
  facet_wrap(~date, ncol=2, scales="free", labeller = labeller(date = date.labs)) + 
  labs(title = "Top Words used by PM Lee in Speeches", x="", y="Proportion Used") +
  scale_fill_manual(values = c( "Yes"="tomato", "No"="lightblue" ), guide = FALSE ) +
  theme_economist(dkpanel = TRUE) +
  theme(axis.title.y = element_text(family = "sans", size = 15, margin=margin(0,30,0,0)),
        plot.title = element_text(family = "sans", size = 18, margin=margin(0,0,10,0)),
        panel.margin.x=unit(1, "lines") , panel.margin.y=unit(3,"lines"))
```

Graphing it was rather fun! I wanted to facet the graphs and also ensure that there were arranged from highest to lowest. To do that, I used `reorder_within` so that it would be ordered within their respective date groups and then added scale_x_reordered so that the labels would come out properly (try it without scale_x_reordered and you'll see what I mean!)

And finally...

![Top Words](https://user-images.githubusercontent.com/68678549/102754747-5edc6180-43a8-11eb-957d-6aeb63af0bb0.png)

It was quite the interesting result because of the fact that it highlighted quite well the key takeaways of each speech. For instance, around the end of April, Singapore experienced an outbreak of Covid-19 cases amongst the foreign workers (hence the top word used being "workers"). Or on 10th April when a lockdown ("circuit breaker" was what the government called it) was put in place and hence "home" was the top word. And obviously for 14th Dec, with the vaccines out, it was the word of the month. 

The keener eye might notice that I chose not to lemmatize or stem the words. I chose not to do it for two reasons: (a) I felt that because of the relatively short length of the speeches, lemmatizing the words might detract from some of its meaning. For instance, Singapore and Singaporean are two words that likely would have been fused. However, I felt that in the context of relatively short speeches, they carried different contexts/meanings. Of course, this meant that things like "vaccines" and "vaccine" appeared in the top 5 words but this issue (as you can see) didn't affect the other 7 speeches. (b) I wanted to try to use only the `tidytext` package as much as possible for the actual data manipulation and it surprisingly (unless I'm wrong then please let me know!) doesn't have any in-built functions to do that. One would have to turn to `textstem` to do that. 

## Sentiment Analysis of speeches

```R
#quick sentiment analysis of each speech
sentiment_words <- binded_words %>% inner_join(get_sentiments("bing")) %>%
  group_by(date) %>%
  count(sentiment) %>%
  spread(sentiment, n, fill = 0) %>%
  mutate(sentiment = positive - negative) %>%
  mutate(newdate = format(date, "%d %b"))
```

I did this in quite the basic way. Going along with the "tidytext" idea, I performed an inner join with the words in the [Bing Liu](https://www.cs.uic.edu/~liub/FBS/sentiment-analysis.html) sentiment corpus, grouped it by date, counted the sentiments and then took the difference between the number of positive and negative sentiments. I also reformatted the dates for graphing purposes.

```R
#plot out sentiments
sentiment_words %>% ggplot(aes(y=sentiment, x=factor(newdate, levels = newdate), fill=sentiment)) +
  geom_bar(stat="identity") +
  labs(title="Change in sentiment over time", x="Date", y="Sentiment", fill="Sentiment Score") +
  theme_economist(base_size = 15) +
  scale_color_economist() +
  theme(legend.position = "right", axis.title.y = element_text(family = "sans", size = 15, margin=margin(0,30,0,0)),
        axis.title.x = element_text(family = "sans", size = 15, margin=margin(30,0,0,0)),
        legend.title = element_text(size=15))
```

Here, I had to use "factor" to get the dates to follow the same order as the df. 

![Sentiment](https://user-images.githubusercontent.com/68678549/102756143-5dac3400-43aa-11eb-8fa3-7d51ce4e1628.png)

Even though it was a rather quick and dirty way of doing it, I must say that the results were pretty good. Again, it reflected the general sentiments of the time - low/negative points in Feb and Apr when the virus was first breaking out and when the virus hit the workers hard. To obviously much brighter days when the vaccine came out!

## Change in words used over time

Finally, a cute one just to see how the number of keywords used changed over the various speeches. 

```R
#use regex to extract words of interest
word_series <- binded_words %>%
  mutate(word = stri_extract(word, regex = "covid|vaccine|virus|singapore")) %>% na.omit() %>%
  count(date, word)

#plot graph over time of change in the count of words selected
word_series %>% ggplot(aes(y=n, x=date, color=word)) +
  labs(title="Change in words used in speech from Feb - Dec", x="Date",y="Count", color="Word") +
  theme_economist(base_size = 15) +
  geom_line(size=1) +
  theme(legend.position = "right", axis.title.y = element_text(family = "sans", size = 15, margin=margin(0,30,0,0)),
        axis.title.x = element_text(family = "sans", size = 15, margin=margin(30,0,0,0)),
        legend.title = element_text(size=15))
```

![Rplot01](https://user-images.githubusercontent.com/68678549/102756313-a4019300-43aa-11eb-917e-db15b71e341d.png)

As someone in the Reddit thread said, "Vaccines go brrrr"

# Conclusion

Welp! That was a rather long article - thanks for reading to the end. In this article, we looked briefly at how one can use the `tidytext` package to perform text mining in a R-approved tidy way. I found it really easy to use, especially as someone who was familiar with working with dataframes. It felt much more intuitive than working with raw strings or other types of corpus so definitely something to consider! 