---
title: Using Selenium's get_attribute method to scrap JavaScript tables
categories: [data]
---

# Introduction

Many web scrapping tutorials on the web currently focus on using BeautifulSoup to scrape websites. However, JavaScript is becoming increasingly common on many websites. As a result, site content is dynamically loaded and BeautifulSoup no longer suffices. While some tutorials have moved toward using `selenium` to scrap websites that are dynamically loaded, many do not cover how to scrap JavaScript tables. Using selenium to load the page and then using BeautifulSoup to extract the HTML source is not sufficient as these tables may not be found in the source code. In this article, we will look at how we can extract such tables. 

## JavaScript rendered tables

We will be using TDAmeritrade's trade confirmations table as an example. The tables on this page are loaded dynamically. Using selenium, we can access the page as such. 

```python
# create driver to access tdameritrade
driver = webdriver.Chrome()

driver.get(r"https://invest.tdameritrade.com.sg/tdaa/index.html#!/accountLobby/statements/confirms")

# wait for all javascript elements to load
driver.implicitly_wait(20)
```

This gives us a table of trade confirmations that look like this:

![image](https://user-images.githubusercontent.com/68678549/113254145-ccc5ec80-92f8-11eb-87d6-dcc8efd13579.png)

Despite the table loading, if you check the source of the page, you will see that the tables are not in the page HTML source. This is due to the table being rendered using JavaScript and hence not appearing in the source code. You can observe this by inspecting the page (CTRL+SHIFT+I) and selecting the table. 

![image](https://user-images.githubusercontent.com/68678549/113254363-0e569780-92f9-11eb-8089-2293bbe383f1.png)

As you can see, the entire table is contained within "account-summary-block". This does not show up in page source, which is unfortunate because all the data that we want is inside that block.  

## Using Selenium's get_attribute

The way to workaround this then is to first select the table using selenium's id selection. Following which, we then use the "get_attribute" method to get the `innerhtml` attribute of the table. This allows us to access the previously unexpended source code within the "account-summary-block" object.

```python
# retrieve html info
tags = driver.find_element_by_class_name('account-summary-table')
html = tags.get_attribute('innerHTML')
soup = BeautifulSoup(html, "html.parser")
```

Only after we have done this can we then use `BeautifulSoup` in the usual ways to extract the html source and begin working on the tables. A useful function that I found [here](https://srome.github.io/Parsing-HTML-Tables-in-Python-with-BeautifulSoup-and-pandas/) then allows us to parse the html table and create a pandas dataframe from it. 

# Conclusion

So, just a very short article (more so to remind myself!) on how to scrape data from JavaScript-rendered tables, particularly if they are nested and do not show up in the source code. Thanks for reading!  