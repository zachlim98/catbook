---
title: Replicating Tastywork’s Probability of 50% Profit for Option Trades
categories: [trading]
---

# Introduction

For those of you who use the Tastyworks platform, you may have noticed that they have a **P50** metric. If you're wondering what that does, it essentially gives you the Probability of closing that option at 50% profit. Now, I always wondered how that calculated that metric and recently stumbled across [this video](https://www.tastytrade.com/shows/the-skinny-on-options-modeling/episodes/probability-of-50-profit-12-17-2015?locale=en-US) where they explained how they did. And I **needed** to replicate it. So, for all you nerds out there who wonder what goes on behind the hood... here ya go! 

*note: this would also be useful if you didn't want to close at 50% profit but wanted to maybe close at 70% profit etc. You could just tweak the numbers around to get your desired PXX metric*

## Background

From the video, it was clear that they were using a Monte Carlo simulation in order to simulate 1000 occurrences of the stock price. From the stock price, they used the Black-Scholes Model in order to calculate the price of the stock (across those 1000 occurrences). They then summed up the number of times the price of the option fell below 50% of its initial value (assuming that you were selling the option) in order to get the P50. Sounds easy enough eh? 

I decided to run this simulation on AAPL - a well-known ticker. Looking first at Tastywork's option chain... 

![image](https://user-images.githubusercontent.com/68678549/107608843-19396800-6c78-11eb-818b-7980a9c086b5.png)

I decided to simulate it on the 122.5 Put, with 37 days to go. You can see at the bottom left hand corner the P50 which currently stands at 89%. We'll try to match that. 

## 1. Simulating Stock Price

As they explained in their video, the first thing they did was run a Monte Carlo simulation on the stock price. We'll use the standard Geometric Brownian Motion model to simulate our stock price. 

```R
library(riingo)
library(ggplot2)
library(tidyverse)

RIINGO_TOKEN <- "YOUR-TOKEN-HERE"

#set token for riingo
riingo_set_token(RIINGO_TOKEN) 
#get prices for ROKU
aapl_price <- riingo_prices("AAPL", start_date = "2019-01-01") 
#calculate log daily returns
returns_tib <- tibble(returns = diff(log(aapl_price$adjClose), lag=1))

#plot log daily returns
returns_tib %>% ggplot(aes(x=returns)) +
  geom_density(fill="#69b3a2", color="#e9ecef", alpha=0.8)
```

I pulled Apple's stock price from Tiingo, this free website that gives better data than Yahoo and then calculated the log daily returns. Plotting it out, you get this: 

![image](https://user-images.githubusercontent.com/68678549/107609046-b8f6f600-6c78-11eb-8d5f-d4ffbdfe398a.png)

Which looks fine. Next, in order to perform GBM calculations, we have to find the drift (the mean) and the sigma (the standard deviation). I'm not going to go into detail about GBM calculations since the focus of this article is the P50 but you can find more of it [here](https://blog.quantinsti.com/random-walk/#Simulation-Stock-Price). 

```R
u = mean(returns_tib$returns)
stdd = sqrt(var(returns_tib$returns))

gbm_vec <- function(nsim = 100, t = 25, mu = 0, sigma = 0.1, S0 = 100, dt = 1./365) {
  # matrix of random draws - one for each day for each simulation
  epsilon <- matrix(rnorm(t*nsim), ncol = nsim, nrow = t)  
  # get GBM and convert to price paths
  gbm <- exp((mu - sigma * sigma / 2) * dt + sigma * epsilon * sqrt(dt))
  gbm <- apply(rbind(rep(S0, nsim), gbm), 2, cumprod)
  return(gbm)
}
```

I got the gbm_vector calculation from [here](https://robotwealth.com/efficiently-simulating-geometric-brownian-motion-in-r/) so credits to RobotWealth for that. I then set it up for our particular put (which has 37 days to go), with 1000 simulations (as per the video). 

```R
gbm <- gbm_vec(nsim = 1000, t = 37, mu = u*100, sigma = stdd*100, S0 = 135.39)

gbm_df <- as.data.frame(gbm) %>%
  mutate(ix = 1:nrow(gbm)) %>%
  pivot_longer(-ix, names_to = 'sim', values_to = 'price')

gbm_df %>%
  ggplot(aes(x=ix, y=price, color=sim)) +
  geom_line() +
  labs(x="Days", y="Price", title="1000 Simulations of AAPL price") +
  theme(legend.position = 'none')
```

![image](https://user-images.githubusercontent.com/68678549/107609256-636f1900-6c79-11eb-9b14-b7b8b3b1a5ad.png)

And that's done! So we've completed the first part, which is the simulation of the stock price. 

## 2. Simulating Put Price

Now that we had the stock price, all we had to do was calculate the Put price for each simulated price path. I wrote a function to calculate the Put price based on the BSM. Next, I created an empty matrix to hold the price of the Puts. And ran a nested loop to take into account (a) the 1000 simulations and (b) the number of days we were simulating for. 

```R
BlackScholes <- function(S, K, r, T, sig, type){
  if(type=="Call"){
    d1 <- (log(S/K) + (r + sig^2/2)*T) / (sig*sqrt(T))
    d2 <- d1 - sig*sqrt(T)
    value <- S*pnorm(d1) - K*exp(-r*T)*pnorm(d2)
    return(value)}
  if(type=="Put"){
    d1 <- (log(S/K) + (r + sig^2/2)*T) / (sig*sqrt(T))
    d2 <- d1 - sig*sqrt(T)
    value <- (K*exp(-r*T)*pnorm(-d2) - S*pnorm(-d1))
    return(value)}
}

putPrice <- matrix(0,38,1000)

for (s in 1:ncol(gbm)){
  for (t in 1:nrow(gbm)) {
    putPrice[t,s] <- BlackScholes(gbm[t,s], 122.5, 0.01, (nrow(gbm)+1-t)/365, 0.37, "Put")
  }
}
```

Once I had the matrix of the Put price, I converted it to a dataframe and then plotted it!

```R
plottest <- as.data.frame(putPrice) %>%
  mutate(ix = 1:nrow(gbm)) %>%
  pivot_longer(-ix, names_to = 'sim', values_to = 'price')

plottest %>%
  ggplot(aes(x=ix, y=price, color=sim)) +
  geom_line() +
  labs(x="Days", y="Price", title="1000 Simulations of AAPL 122.5 Put Price") +
  coord_cartesian(
    xlim = NULL,
    ylim = c(0,25),
    expand = TRUE,
    default = FALSE,
    clip = "on"
  ) +
  theme(legend.position = 'none')
```

![image](https://user-images.githubusercontent.com/68678549/107609475-f8721200-6c79-11eb-849f-a020ce0b0db9.png)

I intentionally limited the "zoom" to only up to 25 as the put started at ~$1.70 and we wanted to see when it would hit P50 (which is \$0.85) so it didn't make sense to see the puts that had gone up to the \$100s. Of course, this graph was still incredibly messy so I just selected 10 random ones to make it look cleaner and limited the yaxis.

![image](https://user-images.githubusercontent.com/68678549/107609774-e6dd3a00-6c7a-11eb-8cce-d424e38622f2.png)

As you can see, some did indeed dip below the $0.85 mark. All we had to do now was quantify *how many* had dipped below that mark. 

## Calculating P50

I first filtered the rows that had prices of less than 0.85. I just wanted to find out how many simulations had *at least one* value below 0.85 so I grouped it by simulation number and then kept only the head of each group. 

```R
query <- plottest %>% dplyr::filter(price<0.85) %>% group_by(sim) %>% slice_head()
cat("P50: ",(nrow(query)/1000))

## P50: 0.859
```

Comparing our value of 86% to Tastywork's, we see that theirs is 89%! So not too far off!

# Conclusion

Of course, this could easily be adapted to P70 if you wished or P20, just by changing the filtered price. Additionally, we could also extend it to spreads by calculating the individual prices of each leg and then adding the prices up. With that said, that's about it for this article and thanks for reading! Let me know if you spot any errors or have any comments.  

 