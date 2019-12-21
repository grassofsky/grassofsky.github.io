+++
date="2019-12-21"
title="对话系统Rasa - 组件 [翻译]"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统Rasa - 组件 [翻译]

该文是Rasa NLU内建的每一个组件的介绍。如果你想要构建自定义组件，可以查看：[Custom NLU Components](https://rasa.com/docs/rasa/api/custom-nlu-components/#custom-nlu-components).内建的组件如下：

- word vector source，词向量源
  - MitieNLP
  - SpacyNLP
- Featurizers
  - MitieFeaturizer
  - SpacyFeaturizer
  - NGramFeaturizer
  - RegexFeaturizer
  - CountVectorsFeaturizer
- Intent Classifiers
  - KeywordIntentClassifier
  - MitieIntentClassifier
  - SklearnIntentClassifier
  - EmbeddingIntentClassifier
- Selectors
  - Response Selector
- Tokenizers
  - WhitespaceTokenizer
  - JiebaTokenizer
  - MitieTokenizer
  - SpacyTokenizer
- Entity Extractors
  - MitieEntityExtractor
  - SpacyEntityExtractor
  - EntitySynonymMapper
  - CRFEntityExtractor
  - DucklingHTTPExtractor

## Word Vector Sources 词向量源

### MitieNLP

简单说明：MITIE初始化，MITIE是一个MIT信息提取库。

输出：无

依赖：无

描述：初始化mitie结构。每个mitie组件都依赖与MitieNLP，因此它需要被放到pipeline中所有mitie组件之前。

配置：MITIE库需要语言模型文件，这个必须在配置文件中设定，更多的信息可以参考 [installing MITIE](https://rasa.com/docs/rasa/user-guide/installation/#install-mitie).

```
pipeline:
- name: "MitieNLP"
  # language model to load
  model: "data/total_word_feature_extractor.dat"
```

### SpacyNLP

简单说明：spacy语言初始化，spaCy是最流行的开源NLP开发包之一，它有极快的处理速度，并且预置了 词性标注、句法依存分析、命名实体识别等多个自然语言处理的必备模型。

输出：无

依赖：无

描述：初始化spacy数据结构。每个spacy组件都依赖于SpacyNLP，因此它需要被放到pipeline中所有spacy组件之前。

配置：语言模型，默认使用配置文件中的语言模型。如果被使用的spacy模型的名字和语言标签（“en”，“de”等）不一致，那么模型的名字可以配置到文件中。这个名字会被`spacy.load(name)`调用。

```
	
Language model, default will use the configured language. If the spacy model to be used has a name that is different from the language tag ("en", "de", etc.), the model name can be specified using this configuration variable. The name will be passed to spacy.load(name).

pipeline:
- name: "SpacyNLP"
  # language model to load
  model: "en_core_web_md"

  # when retrieving word vectors, this will decide if the casing
  # of the word is relevant. E.g. `hello` and `Hello` will
  # retrieve the same vector, if set to `false`. For some
  # applications and models it makes sense to differentiate
  # between these two words, therefore setting this to `true`.
  case_sensitive: false
```

## Featurizers 特化

### MitieFeaturizer

简单说明：MITIE意图特化

输出：无，作为需要意图特化的意图分类器的输入（如，SklearnIntentClassifier）

依赖：MitieNLP

描述：创建意图分类器需要的特征。注意：不能被MitieIntentClassifier组件使用。当前，只有SklearnIntentClassifier能够使用预计算的特征。

配置：

```
pipeline:
- name: "MitieFeaturizer"
```

### SpacyFeaturizer

简单说明：spacy意图特化

输出：无，作为需要意图特化的意图分类器的输入（如，SklearnIntentClassifier）

依赖：SpacyNLP

描述：创建意图分类器需要的特征。

### NGramFeaturizer

简单说明：将char-ngram特征附加到特征向量

输出：无，将该特征向量附加到其他意图特征器产生的特征向量上。

依赖：SpacyNLP

描述：这个特化器将char-ngram特征附加到特征向量上。当训练这个组件用来你查找最常见的字符序列的时候（如，app，ing）。添加的特征为一个布尔类型来标识字符序列是不是出现在单词序列中。注意：在pipeline中这个特化器之前还需要有别的特化器。

配置：

```
	
pipelinepipeline::
 --  namename::  "NGramFeaturizer""NGramFeaturiz 
  # Maximum number of ngrams to use when augmenting
  # feature vectors with character ngrams
  max_number_of_ngrams: 10
```

### RegexFeaturizer

简单说明：正则化特征创建，用来支持意图和实体的分类。

输出：`text_features`和`tokens.pattern`

依赖：无

描述：在训练过程中，正则化意图特征器根据训练数据中的定义创建了一系列的正则化表达式。针对每个正则化，当在输入中找到对应的匹配的时候，都会打上对应的标签，该标签会在后面意图分类器或者实体提取器中使用到。（假设这些分类器在训练阶段已经学习好了，并且这些特征表示某种意图。）用于实体提取的正则化特征创建目前仅支持`CRFEntityExtractor`组件。注意：pipeline中，这个组件必须放在分词器之后。

### CountVectorsFeaturizer

简单说明：将用户消息和标签（意图和回复）特征创建成词袋的形式。

输出：无，输入到需要词袋表述的意图分类器中，（如，`EmbeddingIntentClassifier`）

依赖：无

描述：使用sklearn中的CountVectorizer创建用户输入和标签特征的词袋表述。只带有数字的token将被分为相同的特征。**注意**：如果模型语言不能直接针对空格进行划分，那么在pipeline中针对语言的分词器必须给出配置（如针对中文需要JiebaTokenizer）

配置：关于配置参数的详细介绍可以参见： [sklearn’s CountVectorizer docs](http://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.CountVectorizer.html) 。该特化器是word还是character n-grams，可以通过`analyzer`配置参数进行配置。默认情况下是针对word。如果想要配置成针对character的可以将`analyzer`设置成`char`或`char_wb`。（**注意**：针对character n-grams不要 忘记设定`min_ngram`和`max_ngram`参数，否则字典只会包括单个字符。）对超出词典的单词的处理（这个只有在analyzer属性值为word的时候才适用）：由于训练的数据的词汇量是有限的，因此不能够保证算法不会遇到不认识的单词（单词在训练中没有碰到过的）。为了告诉算法如何对待不认识的单词，一些训练中的单词可以被`OOV_token`替代。这种情况，在预测的时候，所有为识别的单词会被识别成`OOV_token`。举个例子，也可以创建一个单独的意图，附加一些`OOV_tokens`或一些额外的通用的单词。然后实现一个算法会将带有未知单词的消息归为超出范围的意图。具体配置示例如下：

```
pipeline:
- name: "CountVectorsFeaturizer"
  # whether to use a shared vocab
  "use_shared_vocab": False,
  # whether to use word or character n-grams
  # 'char_wb' creates character n-grams only inside word boundaries
  # n-grams at the edges of words are padded with space.
  analyzer: 'word'  # use 'char' or 'char_wb' for character
  # the parameters are taken from
  # sklearn's CountVectorizer
  # regular expression for tokens
  token_pattern: r'(?u)\b\w\w+\b'
  # remove accents during the preprocessing step
  strip_accents: None  # {'ascii', 'unicode', None}
  # list of stop words
  stop_words: None  # string {'english'}, list, or None (default)
  # min document frequency of a word to add to vocabulary
  # float - the parameter represents a proportion of documents
  # integer - absolute counts
  min_df: 1  # float in range [0.0, 1.0] or int
  # max document frequency of a word to add to vocabulary
  # float - the parameter represents a proportion of documents
  # integer - absolute counts
  max_df: 1.0  # float in range [0.0, 1.0] or int
  # set ngram range
  min_ngram: 1  # int
  max_ngram: 1  # int
  # limit vocabulary size
  max_features: None  # int or None
  # if convert all characters to lowercase
  lowercase: true  # bool
  # handling Out-Of-Vacabulary (OOV) words
  # will be converted to lowercase if lowercase is true
  OOV_token: None  # string or None
  OOV_words: []  # list of strings
```

## Intent Classifiers 意图分类

### KeywordIntentClassifier

简单说明：简单关键词匹配的意图分类。不打算使用。

输出：`intent`

依赖：无

输出示例：

```
{
	"intent": {"name": "greet", "confidence": 0.98343}
}
```

描述：这个分类器更常被用作占位符。它能够通过检索传入消息中的关键词识别出`hello`和`goodbye`意图。

### MitieIntentClassifier

简单说明：MITIE意图分类器（使用 [text categorizer](https://github.com/mit-nlp/MITIE/blob/master/examples/python/text_categorizer_pure_model.py)）

输出：`intent`

依赖：`tokenizer`和`featurizer`

输出示例：

```
{
	"intent": {"name": "greet", "confidence": 0.98343}
}
```

描述：这个分类器使用MITIE执行意图分类。这个潜在的分类方法是使用了一个但稀疏线性核的多类别线性SVM（见：[MITIE trainer code](https://github.com/mit-nlp/MITIE/blob/master/mitielib/src/text_categorizer_trainer.cpp#L222)）

配置：

```
pipeline:
- name: "MitieIntentClassifier"
```

### SklearnIntentClassifier

简单说明：sklearn的意图分类器

输出：`intent`和`intent_ranking`

依赖：`featurizer`

输出示例：

```
{
    "intent": {"name": "greet", "confidence": 0.78343},
    "intent_ranking": [
        {
            "confidence": 0.1485910906220309,
            "name": "goodbye"
        },
        {
            "confidence": 0.08161531595656784,
            "name": "restaurant_search"
        }
    ]
}
```

描述：sklearn意图训练器使用grid search的方法训练线性SVM。与之前分类器不同的是，提供了没有胜出的intent。管道中在意图分类器之前需要有一个特征化器。这个特征化器会创建用于分类的特征。

配置：在训练SVM的时候，需要超参数查找来找到最佳的参数集。在配置文件中，可以给出的需要进行尝试的参数。

```
pipeline:
- name: "SklearnIntentClassifier"
  # Specifies the list of regularization values to
  # cross-validate over for C-SVM.
  # This is used with the ``kernel`` hyperparameter in GridSearchCV.
  C: [1, 2, 5, 10, 20, 100]
  # Specifies the kernel to use with C-SVM.
  # This is used with the ``C`` hyperparameter in GridSearchCV.
  kernels: ["linear"]
```

### EmbeddingIntentClassifier

简单说明：Embedding意图分类器

输出：`intent`和`intent_ranking`

依赖：`featurizer`

输出示例：

```
{
    "intent": {"name": "greet", "confidence": 0.8343},
    "intent_ranking": [
        {
            "confidence": 0.385910906220309,
            "name": "goodbye"
        },
        {
            "confidence": 0.28161531595656784,
            "name": "restaurant_search"
        }
    ]
}
```

描述：embedding意图分类器，将用户输入和意图标签输出到相同的空间中。通过最大化他们之间的相似性来训练监督式的embeddings。这个算法是基于[StarSpace](https://arxiv.org/abs/1709.03856). 但是，在实现损失函数的时候有些许的不同，带dropout的额外的隐藏层会被添加进来。这个算法同样提供了`intent_ranking`的结果。这个意图分类器在pipeline中需要放在featurizer之后，这个featurizer会创建embeddings需要的特征。推荐使用`CountVectorsFeaturizer`，它能够被`SpacyNLP`和`SpacyTokenizer`处理。

配置：这个算法也需要对一些超参数进行控制：

- 神经网络结构：
  - `hidden_layers_sizes_a`针对用户输入在embedding层之前设置的隐藏层的大小的列表，隐藏层的层数和列表的大小相等
  - `hidden_layers_sizes_b`针对意图标签在embedding层之前设置的隐藏层的大小的列表，隐藏层的层数和列表的大小相等
  - `share_hidden`，如果设置成True，在用户输入和意图标签上共享隐藏层
- 训练：
  - `batch_size`：训练样本的时候一次训练数据集的大小
  - `batch_strategy`：`sequence`或`balanced`
  - `epochs`：所有训练数据被训练的迭代次数
  - `random_seed`：针对相同的训练数据，是否需要获取有重复性的结果
- embedding:
  - `embed_dim`：设置词嵌入空间的维度
  - `num_neg`：设置不正确的标签数量，算法会在训练的时候最小化和输入的相似度
  - `similarity_type`：相似度计算类型设置，`auto`,`cosine`,`inner`。如果是`auto`，那么类型的选择基于损失函数的类型，针对`softmax`是`inner`，针对`margin`是`cosine`
  - `loss_type`：设置损失函数的类型，`softmax`或`margin`
  - `mu_pos`：控制算法应尝试为正确的意图标签生成嵌入向量的相似性，只当`loss_type`为`margin`的时候使用
  - `mu_neg`：控制错误意图的最大负相似性，只当`loss_type`为`margin`的时候使用
  - `use_max_sim_neg`：如果是true，那么算法仅最小化错误意图标签的最大相似度，只当`loss_type`为`margin`的时候使用
  - `scale_loss`：如果为true，算法会针对预测有高置信度的正确的标签缩小损失。只当`loss_type`为`softmax`的时候使用
- 正则化
  - `C2`，L2正则化的系数
  - `C_emb`，设置最小化不同意图标签的最大相似度的重要性
  - `droprate`，设置dropout率，在0和1之间，例如，droprate=0.1，那么会有10%的输入被忽略

相关参数的默认值定义在`EmbeddingIntentClassifier.defaults`中：

```
defaults = {
    # nn architecture
    # sizes of hidden layers before the embedding layer for input words
    # the number of hidden layers is thus equal to the length of this list
    "hidden_layers_sizes_a": [256, 128],
    # sizes of hidden layers before the embedding layer for intent labels
    # the number of hidden layers is thus equal to the length of this list
    "hidden_layers_sizes_b": [],
    # Whether to share the hidden layer weights between input words and labels
    "share_hidden_layers": False,
    # training parameters
    # initial and final batch sizes - batch size will be
    # linearly increased for each epoch
    "batch_size": [64, 256],
    # how to create batches
    "batch_strategy": "balanced",  # string 'sequence' or 'balanced'
    # number of epochs
    "epochs": 300,
    # set random seed to any int to get reproducible results
    "random_seed": None,
    # embedding parameters
    # dimension size of embedding vectors
    "embed_dim": 20,
    # the type of the similarity
    "num_neg": 20,
    # flag if minimize only maximum similarity over incorrect actions
    "similarity_type": "auto",  # string 'auto' or 'cosine' or 'inner'
    # the type of the loss function
    "loss_type": "softmax",  # string 'softmax' or 'margin'
    # how similar the algorithm should try
    # to make embedding vectors for correct labels
    "mu_pos": 0.8,  # should be 0.0 < ... < 1.0 for 'cosine'
    # maximum negative similarity for incorrect labels
    "mu_neg": -0.4,  # should be -1.0 < ... < 1.0 for 'cosine'
    # flag: if true, only minimize the maximum similarity for incorrect labels
    "use_max_sim_neg": True,
    # scale loss inverse proportionally to confidence of correct prediction
    "scale_loss": True,
    # regularization parameters
    # the scale of L2 regularization
    "C2": 0.002,
    # the scale of how critical the algorithm should be of minimizing the
    # maximum similarity between embeddings of different labels
    "C_emb": 0.8,
    # dropout rate for rnn
    "droprate": 0.2,
    # visualization of accuracy
    # how often to calculate training accuracy
    "evaluate_every_num_epochs": 20,  # small values may hurt performance
    # how many examples to use for calculation of training accuracy
    "evaluate_on_num_examples": 0,  # large values may hurt performance
}
```

## Selectors 选择器

### Response Selector

简单说明：回复选择器

输出：key是`direct_response_inent`，数值是`response,ranking`的字典

依赖：featurizer

输出示例：

```
{
    "text": "What is the recommend python version to install?",
    "entities": [],
    "intent": {"confidence": 0.6485910906220309, "name": "faq"},
    "intent_ranking": [
        {"confidence": 0.6485910906220309, "name": "faq"},
        {"confidence": 0.1416153159565678, "name": "greet"}
    ],
    "response_selector": {
      "faq": {
        "response": {"confidence": 0.7356462617, "name": "Supports 3.5, 3.6 and 3.7, recommended version is 3.6"},
        "ranking": [
            {"confidence": 0.7356462617, "name": "Supports 3.5, 3.6 and 3.7, recommended version is 3.6"},
            {"confidence": 0.2134543431, "name": "You can ask me about how to get started"}
        ]
      }
    }
}
```

描述：回复选择组件可以用来构建回复选择模型，直接从一系列候选的回复中预测出对应的回复。这个模型的预测结果被 [Retrieval Actions](https://rasa.com/docs/rasa/core/retrieval-actions/#retrieval-actions)使用。它将用户输入和响应标签映射到相同的空间中，并且使用相同的神经网络结构和优化EmbeddingIntentClassifier。pipeline中response selector之前需要有个featurizer。推荐使用`CountVectorsFeaturizer`，因为它能够被`SpacyNLP`使用。

配置：该算法包括了`EmbeddingIntentClassifier` 使用的所有参数。此外，还有参数可以用来配置针对特定的取回的意图进行训练。`retrieval_intent`：设置模型训练针对的意图的名字。默认为None.具体配置如下：

```
defaults = {
    # nn architecture
    # sizes of hidden layers before the embedding layer for input words
    # the number of hidden layers is thus equal to the length of this list
    "hidden_layers_sizes_a": [256, 128],
    # sizes of hidden layers before the embedding layer for intent labels
    # the number of hidden layers is thus equal to the length of this list
    "hidden_layers_sizes_b": [256, 128],
    # Whether to share the hidden layer weights between input words and intent labels
    "share_hidden_layers": False,
    # training parameters
    # initial and final batch sizes - batch size will be
    # linearly increased for each epoch
    "batch_size": [64, 256],
    # how to create batches
    "batch_strategy": "balanced",  # string 'sequence' or 'balanced'
    # number of epochs
    "epochs": 300,
    # set random seed to any int to get reproducible results
    "random_seed": None,
    # embedding parameters
    # dimension size of embedding vectors
    "embed_dim": 20,
    # the type of the similarity
    "num_neg": 20,
    # flag if minimize only maximum similarity over incorrect actions
    "similarity_type": "auto",  # string 'auto' or 'cosine' or 'inner'
    # the type of the loss function
    "loss_type": "softmax",  # string 'softmax' or 'margin'
    # how similar the algorithm should try
    # to make embedding vectors for correct intent labels
    "mu_pos": 0.8,  # should be 0.0 < ... < 1.0 for 'cosine'
    # maximum negative similarity for incorrect intent labels
    "mu_neg": -0.4,  # should be -1.0 < ... < 1.0 for 'cosine'
    # flag: if true, only minimize the maximum similarity for
    # incorrect intent labels
    "use_max_sim_neg": True,
    # scale loss inverse proportionally to confidence of correct prediction
    "scale_loss": True,
    # regularization parameters
    # the scale of L2 regularization
    "C2": 0.002,
    # the scale of how critical the algorithm should be of minimizing the
    # maximum similarity between embeddings of different intent labels
    "C_emb": 0.8,
    # dropout rate for rnn
    "droprate": 0.2,
    # visualization of accuracy
    # how often to calculate training accuracy
    "evaluate_every_num_epochs": 20,  # small values may hurt performance
    # how many examples to use for calculation of training accuracy
    "evaluate_on_num_examples": 0,  # large values may hurt performance,
    # selector config
    # name of the intent for which this response selector is to be trained
    "retrieval_intent": None,
}
```

## 分词器

### WhitespaceTokenizer 空格分词器

简单说明：使用空格作为分隔符的分词器

输出：无

依赖：无

描述：为每个空格分隔的字符串序列创建一个token。可以被用作MITIE实体提取器的分词。

配置：如果你想要将意图划分为多个标签，如，预测多个意图，或建模意图的层次结构，可以使用这些flag：`intent_split_symbol`设定意图和响应标签之间的分隔符，默认是空格。如果需要设定的tokenizer大小写不敏感，可以通过设置`case_sensitive: false`。该属性的默认值为`case_sensitive: true`。

```
pipeline:
- name: "WhitespaceTokenizer"
  case_sensitive: false
```

### JiebaTokenizer Jieba分词器

简单说明：用于中文分词的结巴分词器

输出：无

依赖：无

描述：针对中文使用结巴分词器提取token。针对非中文的语言，结巴分词器的执行效果和`WhitespaceTokenizer`一样。提取的token可以用于MITIE实体提取。要使用该分词器，需要保证已经安装了jieba，`pip install jieba`。

配置：自定义字典文件可以通过`dictionary_path`进行设定。

```
pipeline:
- name: "JiebaTokenizer"
  dictionary_path: "path/to/custom/dictionary/dir"
```

如果`dictionary_path`为空（默认情况），那么没有自定义的字典被使用。

### MitieTokenizer Mitie分词器

简单说明：MITIE分词器

输出：无

依赖：MitieNLP

描述：使用MITIE分词器，创建token。提取的token可以用于MITIE实体提取。

配置：

```
pipeline:
- name: "MitieTokenizer"
```

### SpacyTokenizer spacy分词器

简单说明：spacy分词器

输出：无

依赖：SpacyNLP

描述：使用spacy分词器，创建token。提取的token可以用于MITIE实体提取。

## 实体提取

### MitieEntityExtractor

简单说明：MITIE实体提取（使用 [MITIE NER trainer](https://github.com/mit-nlp/MITIE/blob/master/mitielib/src/ner_trainer.cpp)）

输出：附加`entities`

依赖：[MitieNLP](https://rasa.com/docs/rasa/nlu/components/#mitienlp)

输出示例：

```
{
    "entities": [{"value": "New York City",
                  "start": 20,
                  "end": 33,
                  "confidence": null,
                  "entity": "city",
                  "extractor": "MitieEntityExtractor"}]
}
```

描述：使用MITIE实体提取从消息中提取实体。内部分类器使用了带稀疏线性核和自定义特征的多分类线性SVM。MITIE组件不会提供实体confidence值。

配置：

```
pipeline:
- name: "MitieEntityExtractor"
```

### SpacyEntityExtractor

简单说明：spacy实体提取

输出：附加`entities`

依赖：SpacyNLP

输出示例：

```
{
    "entities": [{"value": "New York City",
                  "start": 20,
                  "end": 33,
                  "entity": "city",
                  "confidence": null,
                  "extractor": "SpacyEntityExtractor"}]
}
```

描述：使用spacy预测消息中的实体。spacy使用statistical BILOU transition模型。到目前为止，这个组件只能够用于内置的spacy实体提取模型，并且不能被再训练。这个提取器没有提供任何confidence值。

配置：可以定义那个维度被提取出来，如，实体类型。支持的维度可以参见：[spaCy documentation](https://spacy.io/api/annotation#section-named-entities). 如果没有给出定义，那么会提取所有的维度。

```
pipeline:
- name: "SpacyEntityExtractor"
  # dimensions to extract
  dimensions: ["PERSON", "LOC", "ORG", "PRODUCT"]
```

### EntitySynonymMapper

简单说明：将同义词实体值映射到其他值。

输出：对之前实体提取器提取出来的实体的进行更改。

依赖：无

描述：如果训练数据包括同义词（使用`value`属性）。那么这个组件就知道检测出来的实体值需要被映射到一些值。如，如果你训练下面的示例：

```
[{
  "text": "I moved to New York City",
  "intent": "inform_relocation",
  "entities": [{"value": "nyc",
                "start": 11,
                "end": 24,
                "entity": "city",
               }]
},
{
  "text": "I got a new flat in NYC.",
  "intent": "inform_relocation",
  "entities": [{"value": "nyc",
                "start": 20,
                "end": 23,
                "entity": "city",
               }]
}]
```

那么这个组件会将实体`New York City`和`NYC`映射到`nyc`。这个实体提取器会返回`nyc`。当该组件改变了存在的实体，会将自身附加到此实体的处理列表中。

### CRFEntityExtractor

简单说明：条件随机场实体提取

输出：附加`entities`

依赖：tokenizer

输出示例：

```
{
    "entities": [{"value":"New York City",
                  "start": 20,
                  "end": 33,
                  "entity": "city",
                  "confidence": 0.874,
                  "extractor": "CRFEntityExtractor"}]
}
```

描述：这个组件实现了条件随机场用于命名的实体识别。CRFs可以被认为是无序的马尔科夫链，其中的时间步是单词，状态是实体类。单词的特征（capitalisation, POS tagging, etc）给出了某个实体类的概率，可以用来在相近实体标签之间转换：计算和返回最可能的标签集。如果使用了POS特征（pos或pos2），必须安装spacy。（译者注：这个不是很理解）

配置：

```
pipeline:
- name: "CRFEntityExtractor"
  # The features are a ``[before, word, after]`` array with
  # before, word, after holding keys about which
  # features to use for each word, for example, ``"title"``
  # in array before will have the feature
  # "is the preceding word in title case?".
  # Available features are:
  # ``low``, ``title``, ``suffix5``, ``suffix3``, ``suffix2``,
  # ``suffix1``, ``pos``, ``pos2``, ``prefix5``, ``prefix2``,
  # ``bias``, ``upper``, ``digit`` and ``pattern``
  features: [["low", "title"], ["bias", "suffix3"], ["upper", "pos", "pos2"]]

  # The flag determines whether to use BILOU tagging or not. BILOU
  # tagging is more rigorous however
  # requires more examples per entity. Rule of thumb: use only
  # if more than 100 examples per entity.
  BILOU_flag: true

  # This is the value given to sklearn_crfcuite.CRF tagger before training.
  max_iterations: 50

  # This is the value given to sklearn_crfcuite.CRF tagger before training.
  # Specifies the L1 regularization coefficient.
  L1_c: 0.1

  # This is the value given to sklearn_crfcuite.CRF tagger before training.
  # Specifies the L2 regularization coefficient.
  L2_c: 0.1
```

### DucklingHTTPExtractor

简单说明：Duckling可以用来提取常见的实体，如日期，金钱额度，距离等。

输出：附加`entities`

依赖：无

输出示例：

```
{
    "entities": [{"end": 53,
                  "entity": "time",
                  "start": 48,
                  "value": "2017-04-10T00:00:00.000+02:00",
                  "confidence": 1.0,
                  "extractor": "DucklingHTTPExtractor"}]
}
```

描述：为了使用该组件，需要运行duckling服务。最简单的方式是：`docker run -p 8000:8000 rasa/duckling`。其他的可以参见： [install duckling directly on your machine](https://github.com/facebook/duckling#quickstart)，然后启动服务。Duckling可以用来提取常见的实体，如日期，金钱额度，距离等。需要注意的是，duckling会尽可能多的提取实体类型，并且不提供ranking。比如，维度定义中有`nunber`和`time`，那么针对文本`I will be there in 10 minutes`，组件会提取两个实体`10`和`in 10 minutes`。这种情况下，你的应用需要确定哪个实体类型是正确的。这个提取器的confidence值总是返回1.0，因为它是基于规则的系统。

配置：配置duckling组件需要处理哪些维度，如实体类型。完整的维度可以参见：[duckling documentation](https://duckling.wit.ai/).如果没有定义维度，那么所有可能的维度都会被提取出来。

```
pipeline:
- name: "DucklingHTTPExtractor"
  # url of the running duckling server
  url: "http://localhost:8000"
  # dimensions to extract
  dimensions: ["time", "number", "amount-of-money", "distance"]
  # allows you to configure the locale, by default the language is
  # used
  locale: "de_DE"
  # if not set the default timezone of Duckling is going to be used
  # needed to calculate dates from relative expressions like "tomorrow"
  timezone: "Europe/Berlin"
```