---
title: Building a custom NER model in Spacy v3.1
categories: [data]
---

# Introduction

[Explosion](https://explosion.ai) makes [spaCy](https://spacy.io), a free open-source library for NLP in Python. Recently, they released an update to version 3.1 and this update changed quite a few things from v2, breaking many of the tutorials that were found on Medium previously. The only other article I could find on Spacy v3 was [this article](https://medium.com/analytics-vidhya/building-a-text-classifier-with-spacy-3-0-dd16e9979a) on building a text classifier with Spacy 3.0. 

Building upon that tutorial, this article will look at how we can build a custom NER model in Spacy v3.1, using Spacy's recommended Command Line Interface (CLI) method instead of the custom training loops that were typical in Spacy v2. As this article is a more practical one, we won't be covering the basics of what NER is etc. You can find more in-depth information [in this excellent article](https://medium.com/swlh/build-a-custom-named-entity-recognition-model-ussing-spacy-950bd4c6449f). Quoting from it, NER is essentially an "information extraction task" where we attempt to identify entities (like locations, monetary values, organizations, people etc." from within a given text. 

## Overview

Essentially, in Spacy v3, there has been a shift toward training your model pipelines using the `spacy train` command on the command line instead of making your own training loop in Python. As a result of this, the old data formats (json etc.) that were used in Spacy v2 are no longer accepted and you have to convert your data into a new `.spacy` format. There are hence two main things that we will explore:

1. Updating your data from the old NER format to the new `.spacy` format 
2. Using the CLI to train your data and configuring the training 
3. Loading the model and predicting

## 1. The new `.spacy` format 

In the past, the format for NER was as follows:

```
[('The F15 aircraft uses a lot of fuel', {'entities': [(4, 7, 'aircraft')]}),
 ('did you see the F16 landing?', {'entities': [(16, 19, 'aircraft')]}),
 ('how many missiles can a F35 carry', {'entities': [(24, 27, 'aircraft')]}),
 ('is the F15 outdated', {'entities': [(7, 10, 'aircraft')]}),
 ('does the US still train pilots to dog fight?',
  {'entities': [(0, 0, 'aircraft')]}),
 ('how long does it take to train a F16 pilot',
  {'entities': [(33, 36, 'aircraft')]}),
 ('how much does a F35 cost', {'entities': [(16, 19, 'aircraft')]}),
 ('would it be possible to steal a F15', {'entities': [(32, 35, 'aircraft')]}),
 ('who manufactures the F16', {'entities': [(21, 24, 'aircraft')]}),
 ('how many countries have bought the F35',
  {'entities': [(35, 38, 'aircraft')]}),
 ('is the F35 a waste of money', {'entities': [(7, 10, 'aircraft')]})]
```

Spacy v3.1, however, no longer takes this format and this has to be converted to their `.spacy` format by converting these first in `doc` and then a `docbin`. This is done using the following code, adapted from their [sample project](https://github.com/explosion/projects/tree/v3/tutorials): 

```python
import pandas as pd
from tqdm import tqdm
import spacy
from spacy.tokens import DocBin

nlp = spacy.blank("en") # load a new spacy model
db = DocBin() # create a DocBin object

for text, annot in tqdm(TRAIN_DATA): # data in previous format
    doc = nlp.make_doc(text) # create doc object from text
    ents = []
    for start, end, label in annot["entities"]: # add character indexes
        span = doc.char_span(start, end, label=label, alignment_mode="contract")
        if span is None:
            print("Skipping entity")
        else:
            ents.append(span)
    doc.ents = ents # label the text with the ents
    db.add(doc)

db.to_disk("./train.spacy") # save the docbin object
```

This helps to convert the file from your old Spacy v2 formats to the brand new Spacy v3 format. 

## 2. Using the CLI to train your model

In the past, training of the model would be done internally within Python. However, Spacy has now released a `spacy train` command to be used with the CLI. They recommend that this be done because it is supposedly faster and helps with the evaluation/validation process. Furthermore, it comes with early stopping logic built in (whereas in Spacy v2, one would have to write a wrapper code to have early stopping). 

The only issue with using this new command is that the documentation for it is honestly not 100% there yet. The issue with not having documentation however, is that I sometimes struggled with knowing what exactly the outputs on the command line meant. However, they are **extremely** helpful on their [Github discussion forum](https://github.com/explosion/spaCy/discussions) and I've always had good help there. 

The first step to using the CLI is getting a config file. You can create your own config file [here](https://spacy.io/usage/training#config), with Spacy creating a widget that allows you to setup your config in a pinch. 

<img src="https://user-images.githubusercontent.com/68678549/111624482-b9118500-8826-11eb-91e2-c93cba6ac1f6.png" alt="image" style="zoom:60%;" />

Once you've configured it (in this case, selecting "English" as the language and "ner" as the component), we're ready to fire up the CLI. 

### Filling in your config file

The config file doesn't come fully filled. Hence, the first command you should run is:

```
python -m spacy init fill-config base_config.cfg config.cfg
```

This will fill up the base_config file that you downloaded from Spacy's widget with the defaults. You can play around with the defaults and tweak it as you see fit but let's just go with the default for now. 

### Training the model

Once that's done, you're ready to train your model! At this point, you should have three files on hand: (1) the config.cfg file, (2) your training data in the `.spacy` format and (3) an evaluation dataset. In this case, I didn't create another evaluation dataset and simply used my training data as the evaluation dataset (not a good idea but just for this article!). Make sure all three files are in the folder that you're running the CLI in. Here, I also set `--gpu-id` to 0 in order to select my GPU. 

```
python -m spacy train config.cfg --output ./output --paths.train ./train.spacy --paths.dev ./train.spacy --gpu-id 0
```

This is the output that you should roughly see (it will change depending on your settings)

![image](https://user-images.githubusercontent.com/68678549/111624344-86678c80-8826-11eb-963a-819caa75db47.png)

Although there is no official documentation, the discussions [here](https://github.com/explosion/spaCy/discussions/7450) and [here](https://stackoverflow.com/questions/50644777/understanding-spacys-scorer-output) explain that:

1. **E** is the number of Epochs 
2. `#` is the number of optimization steps (= batches processed)
3. LOSS NER is the model loss 
4. ENTS_F, ENTS_P, and ENTS_R are the precision, recall and [fscore](https://en.wikipedia.org/wiki/Precision_and_recall) for the NER task

The LOSS NER is calculated based on the test set while the ENTS_F etc. are calculated based on the evaluation dataset. Once the training is completed, Spacy will save the best model based on how you setup the config file (i.e. the section within the config file on scoring weights) and also the last model that was trained. 

## 3. Making Predictions with the model

Once your model has been trained through the CLI, you load it and use it in the same way that you did in Spacy 2. In our case, it would look something like this:

```python
nlp1 = spacy.load(R".\output\model-best") #load the best model
doc = nlp1("Did you see the F16 fly by just now?") # input sample text

spacy.displacy.render(doc, style="ent", jupyter=True) # display in Jupyter
```

<img src="https://user-images.githubusercontent.com/68678549/111624385-93847b80-8826-11eb-8bb1-f86307fbf523.png" alt="image" style="zoom:67%;" />

And... it works! 

# Conclusion

Thanks for reading this article! Let me know if you have any questions and would love to discuss more about Spacy v3 and find out what others have been doing with it too. 