+++
date="2019-12-21"
title="对话系统Rasa - choosing a pipeline [翻译]"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统Rasa - choosing a pipeline [翻译]

选择一个NLU pipeline允许你自定义你的模型并针对你的数据进行微调。

## 简单的回答

如果你的训练示例小于1000条，并且针对你的语言有spaCy模型，使用`pretrained_embeddings_spacy`管道：

```
language: "en"
pipeline: "pretrained_embeddings_spacy"
```

如果你带标签的言语超过1000条，使用`supervised_embeddings`管道。

```
language: "en"
pipeline: "supervised_embeddings"
```

## 详细的回答

两个最重要的pipeline是`supervised_embeddings`和`pretrained_embeddings_spacy`。两者之间最大的区别是`pretrained_embeddings_spacy`使用由GloVe或fastText生成的预训练的词向量。`supervised_embeddings`管道不适用预训练的词向量，但是会针对性适应你的数据集。

### pretrained_embeddings_spacy

使用`pretrained_embeddings_spacy`管道的优点是如果你有个训练例子如：“I want to buy apples”，Rasa被询问要预测“get pear”的意图，你的模型已经知道apple和pears两个词是非常接近的。这个在你并没有很多的训练数据的时候很有用。

### supervised_embeddings

使用`supervised_embeddings`管道的优点是你的词向量将很好的和你的domain匹配。举个例子，在英语中，单词balance和symmetry非常接近，但是和cash相差很远。在银行domain中，balance和cash非常接近，你希望你的模型能够知道这点。这个管道不会使用语言相关的模型，因此它能够适用于任何可以被分词的语言（使用空格分词器或其他分词器）。

关于这个主题更详细的内容可以见：[here](https://medium.com/rasa-blog/supervised-word-vectors-from-scratch-in-rasa-nlu-6daf794efcd8) .

### MITIE

你也可以使用MITE在你的管道中作为词向量的来源，见[MITIE](https://rasa.com/docs/rasa/nlu/choosing-a-pipeline/#section-mitie-pipeline). MITIE后端对于小的数据集表现很好，但是如果你有几百的示例，那么训练会花费很长。

但是，我们不建议使用它，因为在后续的发布版本中可能会废弃它。

### 针对你的数据比较不同的pipeline

Rasa提供了用于比较不同管道的性能，参见：[Comparing NLU Pipelines](https://rasa.com/docs/rasa/user-guide/evaluating-models/#comparing-nlu-pipelines).

**注意**：意图识别依赖于实体提取。因此，有些时候NLU获取得到的实体是错误的，而得到的意图是正确的，或者相反。你需要针对意图和实体提供足够的数据。

## 类不平衡

当有很大的类不平衡的时候（即不同类别的数据量差别很大），分类算法表现的通常不好，比如，针对一些意图有很多数据，但是针对另一些意图的数据却很少。为了缓解这个问题，rasa的`supervised_embeddings`管道使用了`balanced`批处理策略。这个算法确保所有类别在所有批次中，或至少在尽可能多的批次中出现，这样还是模拟了一些类出现的频率会更大的事实。Balanced batching默认开启。为了关闭它，可以使用一下配置：`batch_strategy: sequence`。

```
language: "en"

pipeline:
- name: "CountVectorsFeaturizer"
- name: "EmbeddingIntentClassifier"
  batch_strategy: sequence
```

## 多意图

如果你想要把意图划分成多个labels，如，预测多意图，或对层级意图结构进行建模，你只能够使用supervised embeddings管道。为了这么做，在`Whitespace Tokenizer`中使用`intent_split_symbol`标签，这是分割意图标签使用的分隔符，默认是`_`。

[Here](https://blog.rasa.com/how-to-handle-multiple-intents-per-input-using-rasa-nlu-tensorflow-pipeline/) 介绍了如何在Rasa Core和NLU中使用多意图。

下面是配置示例：

```
language: "en"

pipeline:
- name: "WhitespaceTokenizer"
  intent_split_symbol: "_"
- name: "CountVectorsFeaturizer"
- name: "EmbeddingIntentClassifier"
```

## 理解Rasa NLU pipeline

在Rasa NLU中，输入的消息会被一些列的组件处理。这些组件是按照先后顺序一个一个执行的，被称为管道。有很多组件用于实体提取，意图分类，响应选择，预处理等。如果你想要添加你自己的组件，比如，对示例做拼写检查或者情感分析，参见： [Custom NLU Components](https://rasa.com/docs/rasa/api/custom-nlu-components/#custom-nlu-components).

每个组件处理输入并产生输出。这个输出会被管道中定义在之前组件后面的组件使用。有一些组件只能够生成信息被其他组件使用，还有一些组件可以产生Output，可以在执行结束之后返回。如，对于句子“I am looking for Chinese food”，的Output是：

```
{
    "text": "I am looking for Chinese food",
    "entities": [
        {"start": 8, "end": 15, "value": "chinese", "entity": "cuisine", "extractor": "CRFEntityExtractor", "confidence": 0.864}
    ],
    "intent": {"confidence": 0.6485910906220309, "name": "restaurant_search"},
    "intent_ranking": [
        {"confidence": 0.6485910906220309, "name": "restaurant_search"},
        {"confidence": 0.1416153159565678, "name": "affirm"}
    ]
}
```

这个是在`pretrained_embeddings_spacy`中定义的不同组件产生的结果。如，`entities`属性是`CRFEntityExtractor`组件创建的。

## 组件的生命周期

每个组件可以继承基类`Component`并实现一些方法；在pipeline中，不同的方法会以特定的顺序调用。我们假设在pipeline中添加了如下配置：`"pipeline": ["Component A", "Component B", "Last Component"]`。下图显示了训练这个管道的时候的调用顺序：

![](https://rasa.com/docs/rasa/_images/component_lifecycle.png)

在第一个组件创建之前，会创建context（仅仅是python的dict）。这个context用来在组件之间传递信息。举个例子，一个组件能够计算训练数据特征向量，并存储到context中，另一个组件可以获取这个特征向量，做意图分类。

初始的context装满了所有配置值，图中的箭头表示调用的顺序，也是context传递的路径。当所有的组件都被训练和持久化hou最后的context字典被用来预测模型的metadata。

## entity对象解释

在语法分析之后，实体以字典的形式返回。有两个字段显示了管道如何影响实体的返回：extractor字段告诉你哪个实体提取器找到了这个实体，processor字段包含了更改这个实体的组件。

同义词的使用会导致value字段和text字段的内容不匹配。相反，它会返回训练后的同义词：

```
{
  "text": "show me chinese restaurants",
  "intent": "restaurant_search",
  "entities": [
    {
      "start": 8,
      "end": 15,
      "value": "chinese",
      "entity": "cuisine",
      "extractor": "CRFEntityExtractor",
      "confidence": 0.854,
      "processors": []
    }
  ]
}
```

注意：confidence将被CRFEntityExtractor设定。duckling entity extractor返回的confidence总是1,。SpacyEntityExtractor不会提供confidence的值，为null。

## 预定义的pipelines

提供了完整组件列表的简单描述模板。如，下面两种配置是相同的。

```
language: "en"

pipeline: "pretrained_embeddings_spacy"
```

```
language: "en"

pipeline:
- name: "SpacyNLP"
- name: "SpacyTokenizer"
- name: "SpacyFeaturizer"
- name: "RegexFeaturizer"
- name: "CRFEntityExtractor"
- name: "EntitySynonymMapper"
- name: "SklearnIntentClassifier"
```

下面给出了所有预定义的管道模板。

### supervised_embeddings

为了针对你的语言训练Rasa模型，在`config.yml`中定义`supervised_embeddings`管道，如下：

```
language: "en"

pipeline: "supervised_embeddings"
```

`supervised_embeddings`支持任何能够被分词的语言。默认使用空格进行分词。你可以通过增加或改变组件来设定这个pipeline。下面是`supervised_embeddings`默认组件：

```
language: "en"

pipeline:
- name: "WhitespaceTokenizer"
- name: "RegexFeaturizer"
- name: "CRFEntityExtractor"
- name: "EntitySynonymMapper"
- name: "CountVectorsFeaturizer"
- name: "CountVectorsFeaturizer"
  analyzer: "char_wb"
  min_ngram: 1
  max_ngram: 4
- name: "EmbeddingIntentClassifier"
```

因此为了举例，如果你选择了不支持空格分词的语言，你可以替换WhitespaceTokenizer。我们支持一些不同的分词器[tokenizers](https://rasa.com/docs/rasa/nlu/components/#tokenizers),你也可以自己创建[create your own](https://rasa.com/docs/rasa/api/custom-nlu-components/#custom-nlu-components).

这个管道使用了两个CountVectorFeaturizer。第一个是基于单词进行特征化。第二个是保留单词边界，基于字符n-grams进行特征化。从经验上，我们发现第二个特征化非常有用，但是我们决定保留第一个特征化，每个使得特征化功能更加的健壮。

### pretrained_embeddings_spacy

为了使用该模板，配置如下：

```
language: "en"

pipeline: "pretrained_embeddings_spacy"
```

参见[Pre-trained Word Vectors](https://rasa.com/docs/rasa/nlu/language-support/#pretrained-word-vectors) 查看关于加载spacy语言模型更多的信息。这个模板的展开如下：

```
language: "en"

pipeline:
- name: "SpacyNLP"
- name: "SpacyTokenizer"
- name: "SpacyFeaturizer"
- name: "RegexFeaturizer"
- name: "CRFEntityExtractor"
- name: "EntitySynonymMapper"
- name: "SklearnIntentClassifier"
```

### MITIE

为了使用MITIE管道，你将必须从语料中训练词向量。相关的指导可以见：[here](https://rasa.com/docs/rasa/nlu/language-support/#mitie). 

```
language: "en"

pipeline:
- name: "MitieNLP"
  model: "data/total_word_feature_extractor.dat"
- name: "MitieTokenizer"
- name: "MitieEntityExtractor"
- name: "EntitySynonymMapper"
- name: "RegexFeaturizer"
- name: "MitieFeaturizer"
- name: "SklearnIntentClassifier"
```

该管道的另一种方式是使用MITIE进行特征化，名使用多类分类器。训练会非常慢，所以针对大数据集不建议这么做。

```
language: "en"

pipeline:
- name: "MitieNLP"
  model: "data/total_word_feature_extractor.dat"
- name: "MitieTokenizer"
- name: "MitieEntityExtractor"
- name: "EntitySynonymMapper"
- name: "RegexFeaturizer"
- name: "MitieIntentClassifier"
```

## 自定义pipeline

当然，你没有必要一定使用模板，你可以运行自定义的管道，通过简单的将你想用的组件名称添加到配置文件即可。

```
pipeline:
- name: "SpacyNLP"
- name: "CRFEntityExtractor"
- name: "EntitySynonymMapper"
```

这个管道智能做实体识别，没有意图分类。因此Rasa NLU不会预测任何意图。关于组件的详细信息参见：[对话系统Rasa - 组件 [翻译\]](https://zhuanlan.zhihu.com/p/83566179)。

如果你想使用自定义的组件，可以参见： [Custom NLU Components](https://rasa.com/docs/rasa/api/custom-nlu-components/#custom-nlu-components).

## 原文链接

https://rasa.com/docs/rasa/nlu/choosing-a-pipeline/

