---
title: A custom Telegram Bot to share your precious photos (and allow users to filter by theme)
categories: [coding]
---

# Introduction

Ever wanted to create a Telegram bot that would allow you to share your photos with the rest of the world? Not really? Well, after you're done with this article, you'll probably think differently. And if not... well, you're probably wrong. 

In this article, we will look at using the `Reply Keyboard` in Telegram (pictured below) to create our very own bot that sends photos to users based on categories. This article builds off [an earlier article](https://nosparechange.medium.com/asking-a-girl-out-with-a-telegram-bot-b1948d10e475) on a more basic use of a Telegram bot so look at that first if you've never had any experience building a Telegram bot. 

Here's how the final product will look like:

![doggy](https://user-images.githubusercontent.com/68678549/99953941-f906b500-2dbc-11eb-9f41-549e9ef701ab.gif)

## The Concept

So, the concept is simple. We will store our photos on [imgur](imgur.com) and then sort these photos into categories. We will then link the photos and categories in a `pandas` dataframe and then filter the photos based on user input, sending only those filtered photos back to the users. 

## Step 1: Uploading your photos

This part is relatively easy. Just create an account on imgur and upload dem photos. For my case, I was creating a bot to send photos of my dog. 

<img src="https://i.imgur.com/UorQbTA.jpg" alt="Imgur" style="zoom: 25%;" />

Isn't she sweeet? 

## Step 2: Categorizing your photos

Once you're done with uploading your photos, you need to get the **direct links** of the photos. The following screenshots show you how that's easily done in imgur. 

![Screenshot_1](https://user-images.githubusercontent.com/68678549/99947292-be981a80-2db2-11eb-9ede-2e61a26f58fc.png)

![Screenshot_2](https://user-images.githubusercontent.com/68678549/99947299-bfc94780-2db2-11eb-94de-db34f7c31d92.png)

Once you've grabbed your direct links, throw them into excel. Of course, you could probably do it in Python but I decided to go with Excel just because it was simpler (at least in my head).

![Screenshot_3](https://user-images.githubusercontent.com/68678549/99947301-c061de00-2db2-11eb-8f66-c2252f889aec.png)

With the links in excel, you want to use the `SUBSTITUTE` function to get rid of the front part of the url. 

``` vbscript
SUBSTITUE(cell,"https://i.imgur.com/","")
```

Drag this down the entire list and you have your unique identifiers for each image. 

![Screenshot_4](https://user-images.githubusercontent.com/68678549/99947304-c0fa7480-2db2-11eb-9281-ddcdf71cd004.png)

The next step then was to categorize each photo with a certain tag. As you can see, I was very creative with my tags. 

## Step 3: Writing the bot

### 3.1 Creating the Reply Keyboard and Tagging

After all the basic stuff of importing your libraries and enabling logging (which can be found at [this article](https://nosparechange.medium.com/asking-a-girl-out-with-a-telegram-bot-b1948d10e475)), create your Reply Keyboard. Each list represents one row in the keyboard, so create as many rows as you need. Don't try to cram too many items into a single row because it will cause the text to be compressed (and look really ugly!)

```python
# Create shape of reply keyboard
reply_keyboard = [['Sitting', 'Sleeping', 'Baby']
				,['Cone','Outside','Hoomans']]
```

I then imported the csv we created earlier on as a dataframe. 

```python
#import matching table
name_list = pd.read_csv(R"X:\Projects\BdayPhoto-app\url_links.csv")
```

### 3.2 Writing the functions

For this simple bot, I had 3 functions - the standard "start" function, a "photo" function to get the user input, and a "reply" function to send the photos. 

```python
#standard start function
def start(bot, context):
    context.bot.send_chat_action(chat_id=bot.message.chat_id, action=telegram.ChatAction.TYPING)
    bot.message.reply_text("This bot sends you photos of our beloved family pet! Send /photo to start.")

#function to ask for user input
def photo(update, context):
    choice = update.message.reply_text(
        "Choose a category of photos and you'll be sent the photos",
        reply_markup=ReplyKeyboardMarkup(reply_keyboard, one_time_keyboard=True, resize_keyboard=True)) #use replykeyboardmarkup and set one_time_keyboard to true to make it disappear after selection is made
    return choice    

# function to send photos
def reply(update, context):
    user_input = update.message.text # store user input
    update_name = name_list[name_list['CAT']==user_input] # filter matching table by category
    for i in range(0,len(update_name)): # loop to send the photos using the unique identifier + the imgur url header
        url = "https://i.imgur.com/" + update_name.iloc[i,0]
        context.bot.send_photo(chat_id=update.message.chat_id, photo=url) #send the photo
```

As you can see, it's not a lot of code. The main chunk of code is at the end (and even then, 'chunk' is a stretch). We use a for loop to cycle through the filtered list of photos and then tag them on to the imgur header before sending those photos out. 

### 3.3 Creating the message handlers

After I created the individual functions, I had to add them to the main function  through `CommandHandler` and `MessageHandler`.

```python
def main():
    updater = Updater("your-bot-TOKEN", use_context=True)

    dp = updater.dispatcher

    dp.add_handler(CommandHandler("start",start)) 
    dp.add_handler(CommandHandler("photo",photo))
    dp.add_handler(MessageHandler(Filters.regex('^(Sitting|Sleeping|Baby|Cone|Outside|Hoomans)$'), reply))
    dp.add_error_handler(error)

    updater.start_polling()

    updater.idle()
```

The "start" and "photo" commands were your regular type of command. The `MessageHandler` for replying with photos was the interesting one. I used a regex (regular expression) filter to filter all the messages being sent by the user for the categories that I had decided upon for my photos. When these phrases were detected, the "reply" function would be called, starting the whole process of sending photos. 

## Conclusion

So there you have it, a really simple way to create a little Telegram Bot-photo album that you can share with family and friends! Thanks for reading! 

