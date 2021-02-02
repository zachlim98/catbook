---
title: Using Google Maps' Direction API for Group Sorting Problems
categories: [coding, data]
---

# Introduction

With Singapore now in recovery from the Covid-19 pandemic, the government has began to allow people to meet up in groups of 8. My church group (of 37 people) wanted to meet up for dinner. I hence decided to use R to write some code that would allow us to organise the dinner in the best way possible! (*fictitious names have been used for privacy*)

## Sorting Requirements

1. People stay all around Singapore 
2. There are 5 dinner locations 
3. People studied in different regions around the world. 
4. We needed to sort people first by *region* and then by *dinner location*. 
5. Each group has a maximum of 5 people, and each group must have at least 2 people from the same region. 

This was looking to be a fun challenge! 

## Measuring Convenience

With these requirements in mind, I had to start thinking about how I would accomplish this. In order to ensure that people went to the most convenient location, the obvious metric we could use was the travel time between their houses and the dinner location. In order to find that, I used the `mapsapi` which allows us to access the Directions API within the Google Maps API suite. Below is an example of how I used that! (note: MRT stands for Mass Rapid Transit is Singapore's subway)

```R
library(mapsapi)

key = "YOUR API KEY HERE"

doc = mp_directions(
  origin = "Woodlands MRT, Singapore",
  destination = "Holland Village MRT, Singapore",
  alternatives = FALSE,
  mode = "transit",
  key = key, 
  quiet = TRUE
)

r = mp_get_routes(doc)

time = r$duration_text
time
```
```
## [1] 55 mins
```

Once I had the travel time, all I had to do was sort it by descending travel time across the various locations (while grouping by study region) and that would allow me to fit them into groups. 

## Finding Travel Times

Here's how the (fake) data looked like: 

```R
data = read.csv("datas.csv")
head(data)
```

![image](https://user-images.githubusercontent.com/68678549/106590092-b8c76e00-6587-11eb-8b46-23529638d42c.png)

So we had the names of the people, their study region and the closest subway station to their house. I created 5 additional empty columns (to store the travel time to the 5 locations) and then ran a loop to populate that using the `mp_directions`.

```R
for (i in 1:nrow(data)){
  start_pt = as.character(data[i,3])
  
  doc = mp_directions(
    origin = start_pt,
    destination = "Holland Village MRT, Singapore",
    alternatives = FALSE,
    mode = "transit",
    key = key, 
    quiet = TRUE
  )
  
  r = mp_get_routes(doc)
  
  time = r$duration_text # extract just duration from output
  
  data[i,4] <- time # store it in appropriate row
}
```

The problem was... it returned the travel time in a string, with returns like "1 hour 15 mins" and (as you can see from the example above), "55 mins". Hence, I needed to do something to convert these into numeric values so that we could sort them. I accomplished this using an if/else and `stringr`

```R
library(stringr)

# continuing my for loop from above...
  if (str_detect(time, "hour")==TRUE) { # if there is hour, need to convert it
    t <- str_extract_all(time, regexp)
    converted <- (as.numeric(t[[1]][1])*60)+as.numeric(t[[1]][2]) # multiply the hour to minutes
  }
  else {
    converted <- as.numeric(str_extract(time, regexp)) # else, just extract the number and convert to numeric
  }
  data[i,4] <- converted # store it into the row and column
}

head(data)
```

I did this for all 4 locations, wrapping it in a larger for loop to account for all the different locations. After it was done, this is how the dataframe looked!

![datahead](https://user-images.githubusercontent.com/68678549/106590499-1f4c8c00-6588-11eb-9337-f894a7fce7f6.png)

## Sorting and more sorting

With the travel times in hand, all we had to do now to find the nearest location to a person was to find the minimum value. In order to do this, I used `apply`, going through each row one by one and sorting them to find the lowest value, followed by the second lowest value etc. I also used `names()` in order to store it as the names of the location.

Finally, I grouped it by Region and then arranged them from first to fifth choice.

```R
data$first_choice <- apply(data, 1, function(x) names((sort(x))[1]))
data$second_choice <- apply(data, 1, function(x) names((sort(x))[2]))
data$third_choice <- apply(data, 1, function(x) names((sort(x))[3]))
data$fourth_choice <- apply(data, 1, function(x) names((sort(x))[4]))
data$fifth_choice <- apply(data, 1, function(x) names((sort(x))[5]))

data <- data %>% 
  select(Name, Region, first_choice:fifth_choice) %>%
  group_by(Region) %>%
  arrange(first_choice,second_choice,third_choice,fourth_choice,fifth_choice, .by_group=TRUE)

head(data)
```

![datahead2](https://user-images.githubusercontent.com/68678549/106590555-2e333e80-6588-11eb-91e6-cd6f2f7f780b.png)

## Allocation of Groups

With the sorting done, all that was left to do was to figure out the allocation of people to groups. I decided to do it one region at a time, given that they had to be sorted by region. 

First, I decided to take a quick overview of the allocations using `count`.

```R
data %>% count(Region, first_choice)
```

![count](https://user-images.githubusercontent.com/68678549/106590584-37241000-6588-11eb-8344-34aa46e57626.png)

Looking at the data, Bishan was obviously the most central location for most people. However, only one group could be there The instruction given to me during the sorting was that I would *try my best* to make it convenient for most people. Unfortunately, the limited number of locations meant that some people would **have** to travel. 

I decided to prioritize travel time but also made sure that there were at least 2 people from the same region within each dinner location. I filtered based on the nearest location, the second nearest location and then sliced the first three names from each region. 

```R
Group1 <- data %>% 
  select(Name, Region, c(first_choice:fifth_choice)) %>%
  filter(first_choice=="Bishan"|second_choice=="Bishan") %>%
  group_by(Region) %>% 
  slice_head(n=3) %>%
  ungroup() %>%
  slice_head(n=8)
```

I won't bore you with the rest of the sorting but I basically went through each of the locations and sliced them, ensuring that each person was assigned to a group. 

# Conclusion

And that's about it! Not a very difficult problem but it can be tedious if done manually. Using the Google Maps API makes it easier for one to find the distance between locations and then sort them accordingly. 