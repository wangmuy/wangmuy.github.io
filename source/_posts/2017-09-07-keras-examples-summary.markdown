---
layout: post
title: "keras-examples-summary"
date: 2017-09-07 15:55:18 +0800
comments: true
categories: 
keywords: 
description: 
#published: false
---

# NLP

## imdb 评论分类

目标: 划分 imdb 评论电影好坏

### imdb_bidirectional_lstm.py

* 方案

```
Dense
bidirectional lstm
Embedding
```

* 超参数

```
sentence len     = 100
vocab size       = 20000
embedding size   = 128
lstm unroll      = 64
lstm dropout     = 0.5
optimizer        = adam
loss = binary_crossentropy
```

* 指标

~0.8146 after 4 epochs

### imdb_cnn.py

* 方案

```
Dense
GlobalMaxPooling1D
Conv1D
Embedding
```

* 超参数

```
sentence len     = 400
vocab size       = 5000
embedding size   = 50
# filters        = 250
kernel size      = 3
activation       = relu
filter strides   = 1
optimizer        = adam
loss = binary_crossentropy
```

* 指标

~0.89 after 2 epochs

### imdb_cnn_lstm.py

* 方案

```
Dense
LSTM
Conv1D
Embedding
```

* 超参数

```
sentence len     = 100
vocab size       = 20000
embedding size   = 128
# filters        = 64
kernel size      = 5
activation       = relu
filter strides   = 1
lstm unroll      = 70
optimizer        = adam
loss = binary_crossentropy
```

* 指标

~0.8498 after 2epochs

### imdb_fasttext.py

* 方案
[fasttext](https://arxiv.org/abs/1607.01759)

```
Dense
GlobalAveragePooling1D
Embedding
```

* 超参数

```
sentence len     = 400
vocab size       = 20000
embedding size   = 50
if ngram_range > 1: add_ngram() 添加匹配的 ngram 到 x_train/x_test
```

* 指标

~0.8813 after 5epochs for Uni-gram

~0.9056 after 5 epochs for Bi-gram

### imdb_lstm.py

* 方案

```
Dense
LSTM
Embedding
```

* 超参数

```
sentence len     = 80
vocab size       = 20000
embedding size   = 128
optimizer        = adam
loss = binary_crossentropy
```

* 指标

test set: ~0.8084

### lstm_benchmark.py

* 方案

```
Dense
LSTM(不同实现)
Embedding
```

* 超参数

```
sentence len     = 80
vocab size       = 20000
embedding size   = 256
optimizer        = adam
loss = binary_crossentropy
for modes in [0, 1, 2]:
  lstm unroll         = 256
  lstm dropout        = 0.2
  lstm implementation = mode
```

* 指标

```
model=0: val_acc ~= 0.8108
model=1: val_acc ~= 0.8163
model=2: val_acc ~= 0.7946
```

## reuters 新闻分类

### reuters_mlp.py

* 方案

```
Dense(10, activation='softmax')
Dense(512, activation='relu', dropout=0.5)
```

* 超参数

```
setence len = 1000
optimizer   = adam
loss = categorical_crossentropy
```

* 指标

test set: ~0.7911

### reuters_mlp_relu_vs_selu.py

* 方案

```
Dense(num_classes, activation='softmax')
n_dense 个 Dense(dense_units, kernel_initializer, activation, dropoutImpl, droput_rate)
Dense(dense_units, kernel_initializer)
```

* 超参数

```
num_classes = 分类个数

network1:
  n_dense            = 6
  dense_units        = 16
  activation         = relu
  droputImpl         = Dropout
  droptout_rate      = 0.5
  kernel_initializer = glorot_uniform
  optimizer = sgd
  loss = categorical_crossentropy

network2:
  n_dense            = 6
  dense_units        = 16
  activation         = selu
  droputImpl         = AlphaDropout
  droptout_rate      = 0.1
  kernel_initializer = lecun_normal
  optimizer = sgd
  loss = categorical_crossentropy
```

* 指标

network1: test set ~0.5124

network2: test set ~0.6745

## 20 Newsgroup

目标: Newsgroup 文章分类

### pretrained_word_embeddings.py

* 方案

```
导入 glove embedding(不可训练)

Dense(128)
Flatten
MaxPooling1D(35)
Conv1D(128, 5, activation='relu')
MaxPooling1D(5)
Conv1D(128, 5, activation='relu')
MaxPooling1D(5)
Conv1D(128, 5, activation='relu')
Embedding
```

* 超参数

```
sentence len   = 1000
vocab size     = min(20000, dataset实际token数)
embedding size = 100
optimizer      = rmsprop
loss = categorical_crossentropy
```

* 指标

val_acc = ~0.7417

# Visual Recognition

## MNIST 数字识别

TBD
