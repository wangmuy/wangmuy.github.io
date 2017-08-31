---
layout: post
title: "tensorflow-tutorial-seq2seq-rtfsc"
date: 2017-08-30 14:51:58 +0800
comments: true
categories: 
keywords: 
description: 
published: false
---

# seq2seq 中的 rnn embedding 是 trainable 的

```
legacy_seq2seq
embedding_attention_seq2seq

core_rnn_cell.py:line111: EmbeddingWrapper.call(self, inputs, state) inputs仍是list(token id)
  embedding = vs.get_variable("embedding", [self._embedding_classes, self._embedding_size], initializer=initializer, dtype=data_type)
-> line1135 variable_scope.py: get_variable() # custom_getter=None, use_resource=None, Trainable=True
-> line991 variable_scope.py: VariableScope.get_variable(
    _get_default_variable_store(), name, shape=shape, dtype=dtype, initializer=initializer, regularizer=regularizer,
    trainable=trainable, collections=collections, caching_device=caching_device, partitioner=partitioner, valid_shape=valid_shape,
    use_resource=use_resource, custom_getter=custom_getter, constraint=constraint)
    return var_store.get_variable(full_name, ...) # full_name = self.name + "/" + name
-> line225 _VariableStore.get_variable(name, ...)
   custom_getter == None: return _true_getter()
-> line329 _VariableScope.get_variable._true_getter()
-> line663 _VariableScope._get_single_variable()
-> variables.Variable(trainable=True)
```