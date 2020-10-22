---
title: Asking a girl out with a telegram bot
categories: [coding]
comments: true
---

# Introduction

So, when I first mooted this idea to my colleagues, they were not exactly the most impressed. "Zach," these wise, married 30-year olds said, "she's not going to be impressed with a telegram bot... that's literally the lamest, most geeky idea in the world..._if you write the bot in R_" And with that challenge, I HAD to attempt to use Python to write it. 

Having been working with R for the entire summer and not having touched Python in **ages**, it felt nasty using `=` instead of the ever so beautiful `<-`. The loss of `%>%` was also a pretty sad thing. (what I did not realize at this juncture, was that I wouldn't even have to use these because my bot would truly be one of the _lamest_  bots around). 

**Steps:**

1. Request a new bot to be birthed from the BotFather
2. Plan out my bot
3. Write some code to get the bot running
   1. Send and receive messages
   2. Send stickers
   3. Send a location
   4. Send a random number generator (RNG) dartboard
4. Cry because I get rejected

# The Bot Begins

### Step 1: The birth of a new bot

This step was relatively simple. All you had to do was create a [telegram](telegram.org) account and then initiate a chat with @BotFather. Then, send him a `/newbot`, give your bot a name and a username (that has to end in _bot) and he'll give birth to your bot and give your bot a token (not of appreciation; just a token)

![Botfather](https://user-images.githubusercontent.com/68678549/95556279-a9a33a00-0a45-11eb-8f91-a9607b47bc2e.png)

Take care of your token because, as the BotFather says, anyone with the token can access your bot!

### Step 2:  Planning out the bot 

This turned out to be the actual hardest part of the entire project (and there was absolutely no coding involved in this). I had to spend time thinking about what I wanted the bot to do and how I was to incorporate it into the process of asking her out. This really wasn't easy so I went to sketch out a diagram (the only reason I'm including this here because I wanted to show off [typora](https://typora.io/), this amazing markdown editor which allows you to create diagrams super elegantly and simply)

```javascript
st=>start: "Pretend to forget we're going out for dinner"
cond=>condition: "She gets mad"
sucks=>operation: "That sucks. I guess it's the end"
telebot=>operation: "Send her the telegram bot"
e=>end

st->cond
cond(yes)->sucks
cond(no)->telebot
```

![diag1](https://user-images.githubusercontent.com/68678549/95558591-0f44f580-0a49-11eb-88e2-84f97bc120cb.png)

Was basically what I had in my head. But then I realized that I also needed to design the telegram bot (I mean, it couldn't just be a literal:

_opens up telegram bot_ 

"Will you get together with me?"

right..?)

So back to more flowcharts...

```javascript
st=>start: "She opens the bot"
cond=>condition: "Does she start the bot?"
sucks=>operation: "That sucks. I guess it's the end"
telebot=>operation: "Apologise for tricking her and 
tell her we're still going for dinner!"
part2=>operation: "Send her dinner location"
part3=>operation: "Send her cute sticker"
part4=>operation: "Ask her (by using telegram's inbuilt RNG)"
e=>end

st->cond
cond(no)->sucks
cond(yes)->telebot->part2->part3->part4
```

![diag2](https://user-images.githubusercontent.com/68678549/95558409-c68d3c80-0a48-11eb-9206-a78dd8e709c8.png)

I _obviously_ needed the flowchart to help me craft out such an intricate plan. But at least I was clear now! I needed to make the bot send her our dinner location, send her a cute telegram sticker, and then send her a (**RIGGED**) RNG to trick her into agreeing to get together. 

### Step 3:  Coding the bot 

This was honestly the simplest part of the entire project. There's a fantastic telegram [wrapper](https://github.com/python-telegram-bot/python-telegram-bot) that has fantastic [documentation](https://python-telegram-bot.readthedocs.io/en/stable/) and code examples which allowed me to more or less ~~copy~~ borrow what I needed for this bot. 

The first step was to import the telegram library and enable logging on the bot. This turned out to be super useful later on (when I struggled to send the stupid sticker and couldn't figure out what the issue was)

```python
import logging

import telegram
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters

# Enable logging
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
                    level=logging.INFO)

logger = logging.getLogger(__name__)
```

After the standard (template) headers, it was time to start defining some functions for our bot. The first two standard ones that one has to define is `/start` and `/help`. Which is exactly what I did. 

```python
def start(bot, context):
    context.bot.send_chat_action(chat_id=bot.message.chat_id, action=telegram.ChatAction.TYPING)
    bot.message.reply_text("I'm not actually busy. We're still going for dinner.")
    
def help(update, context):
    update.message.reply_text("Help!")
```

I added the `action=telegram.ChatAction.TYPING` to create more **realism**. With this, the bot would pretend to type (wow so cool) before sending the message. 

I then added the standard template bits below to get the bot running. We needed a dispatcher to send and receive from the telegram API and then needed to add CommandHandlers to respond to the incoming telegram commands.

```python
def main():
    updater = Updater("BOTTOKEN", use_context=True)

    dp = updater.dispatcher

    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(CommandHandler("help", help))
    dp.add_error_handler(error)

    updater.start_polling()

    updater.idle()


if __name__ == '__main__':
    main()
```

And it was live. 

![alivebot](https://user-images.githubusercontent.com/68678549/95556319-b9bb1980-0a45-11eb-8b1a-68b01b93d2ee.png)

Now came the tricky bit, sending her a sticker! I really struggled with this one for awhile because I could not for the life of me figure out how to identify the sticker - each telegram sticker has a unique ID. I tried using the "Sticker ID bots" on telegram but they kept giving me incorrect IDs. I eventually figured out that the easiest way to do it was to run the bot, send yourself a sticker and then check the log for the sticker ID. 

In order to check the log, I stopped the bot and then headed over to: https://api.telegram.org/botYOURTOKENHERE/getUpdates (so to clarify, it's /bot[yourBotToken]/getUpdates) and sent myself a sticker. 

![stickid](https://user-images.githubusercontent.com/68678549/95556753-5da4c500-0a46-11eb-8767-44919f12a508.png)

(yes, the sticker pack I sent it from was indeed _beymax_ kmn). The file_id is (incidentally) the id of the sticker, and to send the sticker, we just had to create the sticker function: 

```python
def sticker(update, context):
    update.message.reply_sticker(sticker="CAACAgQAAxkBAAP8X3fhTE1oKX3aziRqZ9jC8EifIe4AAh8AA7mQDgABvw5_ATZHWfkbBA")
```

and add it in as a command:

``` python
dp.add_handler(CommandHandler("sticker", sticker))
```

![image](https://user-images.githubusercontent.com/68678549/95571564-3dcbcc00-0a5b-11eb-883c-8cc708770667.png)

Next, we needed to send her our dinner location, which was easily done with `reply_location`. Note that the location has to be given in lat, long (which you can easily get from Google Maps)

```python
def dinnerplace(update, context):
    update.message.reply_location(1.29053,103.84646)
    context.bot.send_chat_action(chat_id=update.message.chat_id, action=telegram.ChatAction.TYPING)
    update.message.reply_text("Here's where we're going for dinner!")
```

And finally, the RNG (which I rigged to force her to agree to getting together with me). You could choose between three different emojis (dice, dartboard, and basketball) and by setting the "value", it would pre-determine what the outcome was. For dartboards, setting it to '6' meant that it would hit the BULLSEYE

```python
def decision(update, context):
    context.bot.send_dice(chat_id=update.message.chat_id, emoji="🎯", value=6)
    update.message.reply_text("Heh, I think that's a yes..?")
```

Throw all these into `CommandHandlers`... (its '24' for one of the commands because I made her do some sort of quiz before finding out the dinner place and '24' was the correct answer to the quiz)

```python
dp.add_handler(CommandHandler("24",dinnerplace))
dp.add_handler(CommandHandler("decision",decision))
```

And presto~ the bot is DONE! 

![image](https://user-images.githubusercontent.com/68678549/95571629-55a35000-0a5b-11eb-9581-feba513e3c47.png)

# Conclusion: Getting to the BOTtom of it

So, the clear question on everyone's mind is... did it work? Can I impress potential partners by doing nerdy things like coding a telegram bot? Does lying to a potential partner BUT then sending them a telegram bot make up for the lying? The answer, I guess, is for me to know and for you to (now that you have the rough skeleton of how to code a telegram bot) find out :p 
