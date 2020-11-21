---
title: A simple exploration of a Telegram chatgroup
categories: [data]
---


# Introduction

A couple of days ago, one of my friends mentioned that it would be interesting to do a visualisation of our group chat - to see things like who spoke the most, who spoke after who the most, and what were the words most commonly used. In this article, I show you how I did using `pandas`, `json`, and `plotly`. 

## Post Outline
1. Gathering Data
2. Cleaning and Preparing Data
3. Visualizing the Data

## 1. Gathering Data

<img src="https://user-images.githubusercontent.com/68678549/99867692-9368cc00-2bf6-11eb-872d-e0e9ff34c7cf.png" alt="how-to-export" style="zoom:50%;" />

The group that we were in is hosted on telegram and the data was easily to pull. As shown, you just had to use Telegram Desktop to export the files. You have the option of either exporting as a .html or .json but I would 100% recommend .json just because its easier for python to read. 

## 2. Cleaning and Preparing Data

The cleaning of the data was less fun. The files came in a _nested_ json file and hence had to be flattened (or `normalized` in python). So, we imported the files...

```python
import pandas as pd
import json
from pandas.io.json import json_normalize

#open json file
with open('hist.json', encoding="utf8") as f:
    d = json.load(f)

#normalize it according to parent node
norm_msg = json_normalize(d['messages'])
msg_df = pd.DataFrame(norm_msg) #store it in a dataframe
```

...and then proceeded to normalize it. So, how json_normalize works is that it basically unpackages a nested json based on the parent node. In this case, if we had just used the pandas `read_json`, it would have looked something like this. 


```python
pdread = pd.read_json('hist.json') #use pandas to read json file
pdread.loc[:, pdread.columns != 'name'] #show all columns except group name
```

![image](https://user-images.githubusercontent.com/68678549/99868503-06297580-2bfe-11eb-97de-4b8168d68fe5.png)

As you can see, the actual messages are stored in the parent node "messages" and hence we unpackaged it by `norm_msg = json_normalize(d['messages'])` and pointed it to the parent node "messages".

Once that was done, I needed to extract the important columns. I took only the messages types that were labelled "message" because the output had also recorded random things like starting a poll, sending a location, sending a sticker etc. and these did not contain actual text. I then also filtered it to only give me the important components of date, text, and who the text was from. I then relabelled it to make it easier to understand.  


```python
msg_df_filtered = msg_df[msg_df.type=="message"] #filter only message type message 
msg_df_filtered = msg_df_filtered[["date","text","from"]] #filter important variables 
msg_df_filtered = msg_df_filtered.replace({'from' : { 'FriendA' : 'A', 'FriendB' : "B", 'FriendC' : "C", "FriendD" : "D", "FriendE" : "E" , "FriendF" : "F" , "FriendG" : "G", "FriendH" : "H" }}) #rename senders
msg_df_filtered.head(3)
```

![image](https://user-images.githubusercontent.com/68678549/99868484-d24e5000-2bfd-11eb-9258-fad30c738696.png)

I also noticed that some of the messages contained random characters and I needed to get rid of those so that proper words could be formed. 


```python
temp = msg_df_filtered #duplicate the df
temp['text'] = temp['text'].str.replace('[^A-Za-z0-9]+', " ") #replace all non-alaphabet characters
temp = temp.dropna() #remove the na
```

## 3. Visualizing the Data

### Most common word used

Finally, we were done and ready to get on with the fun stuff! In order to count the number of words, I first joined together all the words from the text column together before changing them all to lower caps and splitting them up. I then used `value_counts` and pulled the top 100 words used in the chat


```python
 common_word = pd.Series(' '.join(temp['text']).lower().split()).value_counts()[:100] #show top 100 words used
 common_word = common_word.reset_index().rename(columns={"index":"Word", 0:"Count"}) #reset the index and rename the columns 
```

I then plotted it out using `plotly.express` in a simple bar chart. Clearly the word "the" was extremely popular


```python
import plotly.express as px

fig = px.bar(common_word, x='Word', y='Count')
fig.show()
```

![common word](https://user-images.githubusercontent.com/68678549/99868539-51dc1f00-2bfe-11eb-869b-bcd94c355dd3.png)

### Who sent the most messages?

In order to see who had sent the most messages, I then used `value_counts` on the from column and plotted it out too. 


```python
messages_sent = temp['from'].value_counts()
messages_sent = messages_sent.reset_index().rename(columns={"index":"User", "from":"Count"})

fig2 = px.bar(messages_sent, x='User', y='Count')
fig2.show()
```

![AtoA](https://user-images.githubusercontent.com/68678549/99868538-50125b80-2bfe-11eb-8121-309e8da85cd4.png)

### Who responds to who?

This was an interesting one. I wanted to gauge interactions and see who replied the most after whom. I hence created a simple loop to create a list, recording which person tended to reply after which other person. From the looks of it, I guess person A had a lot of fun replying after himself. 


```python
replies = [temp.iloc[n,2] + " to " + temp.iloc[(n-1),2] for n in range(1,len(temp))] #list constructor to create replying list
replies.insert(0,"A to A") #insert an initial convo to make list same length as df

temp['Replies'] = replies #added list to df

replies_to = temp['Replies'].value_counts() #count
replies_to = replies_to.reset_index().rename(columns={"index":"Replies", "Replies":"Count"}) #reset index and rename

fig3 = px.bar(replies_to, x='Replies', y='Count')
fig3.show()
```

![Atoreply](https://user-images.githubusercontent.com/68678549/99868536-50125b80-2bfe-11eb-8473-6575cb751775.png)

### Custom words

Finally, I wanted to a list of custom words that allowed me to see how many times the word had been used. I created a list of these words and then counted the number of times they appeared in the `pd.Series` from our text data. 


```python
list_words = ["fun", "love","joy","anger","hate","upset"] #create list of custom words
int_word_count = [sum(pd.Series(' '.join(temp['text']).lower().split())==i) for i in list_words] #count number of times they appeared

#create df to store
int_words = pd.DataFrame(
{"Word" : list_words,
"Count" : int_word_count}
)

#plot df
px.bar(int_words, x="Word",y="Count",color_discrete_sequence=px.colors.qualitative.G10)
```

![funwords](https://user-images.githubusercontent.com/68678549/99868535-4db00180-2bfe-11eb-9816-4be2bba97793.png)

I guess we all love each other a lot and have plenty of fun. 

## Conclusion

So there you have it! A fun little exploration of words and phrases in a group chat. Thanks for reading and hope you found it interesting!

