---
title: Building a Recommendation System using Keras
summary: Using keras to build a recommendation system with data from half a million user visits to EverydayHealth.com.
date: 2020-02-26
authors: ["admin"]
tags: ["python", "machine learning", "keras"]
output: html_document
---



## Introduction

This project is about creating a recommendation system using a dataset acquired from EverydayHealth.com. This is a continuation of a project I did for my Machine Learning class senior year, although this time I will be using a model built with Keras. EverydayHealth.com is an online health website where users can browse articles about different health topics. The goal is to recommend relevant articles to users, and I will compare two models for this goal. The first will be a user based system, recommending articles based on those viewed by similar users. The second is a content-based system, recommending articles that cover topics similar to those already read by the user. 

## Dataset

The dataset consists of three tables, one with basic demographic data for each user, one with keywords from each article in the data, and one with the clicks for each user. The dataset consists of about 500,000 users, 63,000 articles, and 20 million clicks. In order to ensure that I have enough behavior from users to model them, I limited the data to those users who have at least 20 clicks. 

For each user, I removed one click to use as a test set, then split the remaining data into training (80%) and validation (20%) sets.



```python
from keras.layers import Input, Embedding, Flatten, Dot, Dense, Concatenate
```

```
## Using TensorFlow backend.
```

```python
from keras.models import Model, load_model
from scipy.sparse import csr_matrix
import pandas as pd
import numpy as np

train_df = pd.read_csv("data/recommendation/train_clicks.csv")
valid_df = pd.read_csv("data/recommendation/valid_clicks.csv")

train_df.head()
```

```
##    user_id  url_id
## 0   112590   32442
## 1   112590   57672
## 2   112590    8055
## 3   112590   27601
## 4   112590   31862
```

Since this is implicit data, I also need to generate negative examples for the model to learn from. For each user, I'll generate roughly 50 false clicks by creating random permutations of the user IDs and the url IDs, then merging that data set with the real one to eliminate duplicates. This won't produce exactly 50 negative examples for each user, but it will come close and runs much faster than the more exact method I originally tried (33 seconds vs 33 hours).


```python
def get_negative_examples(train_df, num_false):
    """ Add negative examples to training data"""
    url_ids = train_df['url_id']
    cols = url_ids
    n_url = max(cols)+1
    
    user_ids = train_df['user_id']
    rows = user_ids
    n_user = max(rows)+1
    
    new_user_ids = np.repeat(np.random.permutation(np.arange(n_user)), num_false)
    new_url_ids = np.random.randint(n_url, size = len(new_user_ids))
    new_data = pd.DataFrame({'user_id':new_user_ids,
              'url_id':new_url_ids})
    new_data = new_data.merge(train_df, how='outer', indicator=True).loc[lambda x : x['_merge']=='left_only']
    
    user_ids = np.append(user_ids, new_data['user_id'])
    url_ids = np.append(url_ids, new_data['url_id'])
    labels = np.append(np.ones(train_df.shape[0]), np.zeros(new_data.shape[0]))
    
    return user_ids, url_ids, labels
```


```python
np.random.seed(101)
user_ids, url_ids, labels = get_negative_examples(train_df, 50)
user_valid, url_valid, labels_valid = get_negative_examples(valid_df, 50)
```

## Building the Model

In order to convert the user and url IDs into meaningful inputs, I'll create an embedding layer using 5 latent variables. I'll train three model architectures and compare their perfomance, all of which use a sigmoid activation function in the output layer to get the probability that a user will visit a given url based on their history. The first model will just take the dot product of these vectors and send that to the output layer while the other two will concatenate the embeddings and then send that vector through dense layers. One of these models will use one dense layer with 16 nodes while the second will use two dense layers, one with 16 nodes and the second with 8. I used AWS to train these models since it would take about a day to train them on my laptop and only a few minutes on a powerful EC2 instance. 


```python
n_users = max(user_ids) + 1
n_urls = max(url_ids) + 1

def create_model(n_url, n_user, combine='dense', k=5, nodes=[16]):
    url_input = Input(shape=[1], name="UrlInput")
    url_embedding = Embedding(n_url+1, k, name="UrlEmbedding")(url_input)
    url_vec = Flatten(name="FlattenUrl")(url_embedding)

    user_input = Input(shape=[1], name="UserInput")
    user_embedding = Embedding(n_user+1, k, name="UserEmbedding")(user_input)
    user_vec = Flatten(name="FlattenUser")(user_embedding)
    
    if combine == 'dense':
        vector = Dot(name="DotProduct", axes=1)([url_vec, user_vec])
    else:
        vector = Concatenate(name="Concat", axis=-1)([url_vec, user_vec])
        
    if nodes:
        for i, n in enumerate(nodes):
            layer = Dense(n, activation='relu', name=f'layer_{i}')
            vector = layer(vector)

    result = Dense(units = 1, activation='sigmoid', name='result')(vector)
    model = Model([url_input, user_input], result)
    
    return model
```

For my scoring metric, I'll use binary cross entropy to evaluate perfomance, and train them in two epochs, saving the best performance across the epochs. 


```python
# Dot Product Model
model_dot = create_model(n_urls, n_users, combine='dense', k=5, nodes=None)
model_dot.compile('adam', 'binary_crossentropy')

model_dot.fit([url_ids, user_ids], labels, batch_size=512,
         epochs=2, 
         validation_data=([url_valid, user_valid], labels_valid))
         
# First Concat Model
model_concat_16 = create_model(n_urls, n_users, combine='concat', k=5, nodes=16)

model_concat_16.compile('adam', 'binary_crossentropy')

model_concat_16.fit([url_ids, user_ids], labels, batch_size=512,
         epochs=2, 
         validation_data=([url_valid, user_valid], labels_valid))
         
# Second Concat Model
model_concat_16_8 = create_model(n_urls, n_users, combine='concat', k=5, nodes=[16, 8])

model_concat_16_8.compile('adam', 'binary_crossentropy')

model_concat_16_8.fit([url_ids, user_ids], labels, batch_size=512,
         epochs=2, 
         validation_data=([url_valid, user_valid], labels_valid))
```

```python
model_dot = load_model("data/recommendation/model_dot.h5")
model_concat_16 = load_model('data/recommendation/model_concat_16.h5')
model_concat_16_8 = load_model('data/recommendation/model_concat_16_8.h5')
```

In order to evaluate the performance of these models, I saved one url that each user visited as part of the training set. I'll generate 100 urls for each user that they didn't visit, and then rank the probabilities for these url and see if the url they did visit is in the top 10 recommended.


```python
def get_hr_10(frame, pred='pred'):
    results = frame.groupby('user_id')[[pred, 'label']].apply(lambda x: x.sort_values(by=pred, ascending=False)[:10]).reset_index()
    acc = sum(results['label'])/len(frame['user_id'].unique())
    return acc
    
def score_model(model, test_url, test_user, labels, batch_size=512):
    predictions = model.predict([test_url, test_user], batch_size=batch_size)
    predictions = pd.DataFrame({'user_id':test_user,
                                'url_ids':test_url,
                                'label':labels,
                                'pred':predictions.flatten()})
    
    return get_hr_10(predictions), predictions
```


```python
test_clicks = pd.read_csv("data/recommendation/test_clicks.csv")
test_user_ids, test_url_ids, test_labels = get_negative_examples(test_clicks, 100)

score_dot, predictions_dot = score_model(model_dot, test_url_ids, test_user_ids, test_labels)
score_concat_16, predictions_concat_16 = score_model(model_concat_16, test_url_ids, test_user_ids, test_labels)
score_concat_16_8, predictions_concat_16_8 = score_model(model_concat_16_8, test_url_ids, test_user_ids, test_labels)
```


```python
print([round(x, 3) for x in [score_dot, score_concat_16, score_concat_16_8]])
```

```
## [0.873, 0.899, 0.907]
```

Overall the models all performed very similar to one another, with the concatenated model with two dense layers doing slightly better than the others. This model performed about 4% better than the dot product model. The architecture allowed the model to learn some relationships between the 10 total latent variables for users and urls. Going forward, I am going to build another system using a list of key words that I have for each url. This means I can try to recommend urls based on topic similarity rather than the behavior of users, so it will be interesting to see which one performs better.
