---
title: Building a simple engima machine
categories: [coding]
---

# Introduction 

This was for a random school assignment where we had to build an enigma machine (or a pseudo-enigma machine) which basically had to encode and decode a message when we sent it. I peeked around the interwebs looking for something to draw inspiration from but most of them were tutorials for how to do it in `python` when I had to do it in R. So, this is just a quick little post sharing how I did it. 

# The Coder

I decided to break down my pseudo-engima machine into three main parts: 

1. Function that generated the randomized plugboard (to match one letter to another letter)
2. Function that asked for an input (and pre-processed it to make the scrambling easier)
3. The main function that would allow for the input to be taken in and spat out

## Randomizing the plugboard

In order to create the plugboard, I used R's inbuilt `letters` and from there sampled 13 random letters and then matched these to the set difference of 13 that had not been picked the first round. I then appended them to each other and created a named list (in order to get key-value pairs). I also allowed users to select their seed (this seed would then theoretically be passed from one spy to another who would then use the same seed to decrypt the message HAHA)

```R	
#function to allow allies to generate new plugboard config
new_code <- function(){
  seed <- readline(prompt = "Choose the seed for the configuration: ")
  #set.seed to allow for different config to be generated
  set.seed(seed)
  #generate new key-pair value
  sample1 <- sample(letters, 13) 
  sample2 <- sample(setdiff(letters,sample1),13) 
  sets1 <- append(sample1,sample2)
  sets2 <- append(sample2,sample1)
  names(sets1) <- sets2
  return(sets1)
}
```

## Input Function

The next thing I needed to create was a function that asked for an input. I faced an issue here with users typing in non-letters, causing the enigma machine to return an error. I hence wrote a small function that checked to ensure that only letters had been inputted. It used *regex* (which I have found to be super helpful honestly and want to learn more about)

```R
#function to check that message typed in is only alphabets
letters_only <- function(x) !grepl("[^A-Za-z ]", x)
```

I then created the function that would prompt for a new message and then prepare the entered message to be encrypted/decrypted by splitting up the strings to individual characters. Also some basic error handling should the user not type in a message with only alphabets.

```R
#function to prompt user to enter new message
enter_message <- function(){
  message <- readline(prompt="What is your message: ")
  if (letters_only(message)==T){ #check if messages are alphabets
    store <- strsplit(tolower(message),split="") #split and convert to lower letters
  } else{
    print("ERROR: Please type in only letters") #error msg if disallowed entry
  }
  return(store)
}
```

## Enigma Secrets

And... finally the encryption/decryption function itself. It was quite a simple one wherein I just created a new list by drawing from the plugboard list. 

```R
#function to encode and decode
enigma <- function(){
  cyp <- new_code()
  new_msg <- enter_message() #prompt for new message
  decode <- c()
  for (i in new_msg){
    char <- cyp[i] #access PB config
    decode <- append(decode,char) #add to encoded/decoded character list
  }
  readable <- gsub("NA"," ",paste(decode, collapse = "")) #make it easier to read
  cat("Your message is: ",readable) #print out the encoded/decoded message
}
```

Of course I had to test it out with something historically relevant (note: I'm not sure if this message was ever sent by the enigma machine lol don't quote me on it)

```
enigma()
###[1] Choose the seed for the configuration: 43
###[2] What is your message: the war is over
###[3] Your message is:  lpk sdq bw zxkq
```

And then decoding it (maybe decoded by [Benedict Cumberbatch himself](https://en.wikipedia.org/wiki/The_Imitation_Game)??)

```
enigma()
###[1] Choose the seed for the configuration: 43
###[2] What is your message: lpk sdq bw zxkq
###[3] Your message is:  the war is over
```

# Conclusion

So, nothing much to conclude really. Just wanted to share the code I had written in R so it hopefully serves as a guide/reference for future peoples!