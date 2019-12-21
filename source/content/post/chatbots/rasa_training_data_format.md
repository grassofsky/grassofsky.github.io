+++
date="2019-12-21"
title="对话系统Rasa-训练数据格式 [翻译]"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统Rasa-训练数据格式 [翻译]

## 数据格式

支持Markdown和Json两种格式，通常情况下Markdonw更合适（Json格式的介绍省略）。

### Markdown格式

Markdown是用户阅读和书写Rasa NLU最方便的格式。例子给出了使用无序列表的格式，如`-`,`*`,`+`。例子通过意图进行组合，实体和实体的名字通过markdown的链接形式给出，如`[entity](entity name)`

```
## intent:check_balance
- what is my balance <!-- no entity -->
- how much do I have on my [savings](source_account) <!-- entity "source_account" has value "savings" -->
- how much do I have on my [savings account](source_account:savings) <!-- synonyms, method 1-->
- Could I pay in [yen](currency)?  <!-- entity matched by lookup table -->

## intent:greet
- hey
- hello

## synonym:savings   <!-- synonyms, method 2 -->
- pink pig

## regex:zipcode
- [0-9]{5}

## lookup:currencies   <!-- lookup table list -->
- Yen
- USD
- Euro

## lookup:additional_currencies  <!-- no list to specify lookup table file -->
path/to/currencies.txt
```

用于RasaNLU训练的数据被结构化成如下部分：

- 通常的例子（common example）
- 同义词（synonyms）
- 正则化特征（regex features）
- 查找表（lookup tables）

虽然常见的例子是唯一必填的部分，但是其他数据结构有助于减少NLU模型训练需要的例子，并帮助NLU模型对于预测的结果更加有信心。

同义词可以将提取出来的实体映射到相同的名字，如例子中将"my savings account"映射到"savings"。但是，这个仅仅发生在实体被提取出来之后，因此需要提供含有同义词表述的实例，使得Rasa能够通过学习将实体提取出来。

查找表可以以list或txt文件（以newline分隔）的形式提供。当加载训练数据的时候，这些文件被用来生成大小写不敏感的正则化模式（基于正则化特征）。举个例子，货币列表中的名字很容易被当做实体挑选出来。

## 提高意图分类和实体识别

### 通用的例子（common examples）

通用例子的组成主要包括三个部分：`text`，`intent`，`entities`。前两个是字符串，最后一个是数组。

- text是用户消息，必填
- intent是text对应的意图，选填
- entities是text中需要被标识出来的部分，选填

Entities带有开始和结束值，组成python格式的范围选择，比如，`text="show me chinese restaurants"`，那么`text[8:15]=='chinese'`。实体可以跨单词，实际上，`value`属性并不直接与子串相关。那样可以映射同义词，或误拼写的词，到相同的一个`value`。

```
## intent:restaurant_search
- show me [chinese](cuisine) restaurants
```

### 正则化特征（Regular Expression Features）

正则化表达式可以有助于支持意图分类和实体提取。举个例子，如果你的实体具有固定的结构（如邮编，或email地址），那么可以使用正则化表达式提取这些实体。如邮编可以用下面的表达式：

```
## regex:zipcode
- [0-9]{5}

## regex:greet
- hey[^\\s]*
```

这个名字并没有定义实体或意图，这个仅仅是用户可读的描述，用来帮助记住这些正则表达式的作用，同时是相关模式特征的标题。正如上面的例子中，可以使用正则化特征提高意图分类的性能。

尽量的使用更加紧缩的匹配形式，如使用`hey[^\s]*`替换`hey.*`。由于后一个表达式会匹配到更多无效的信息。

正则化特征目前仅支持`CRFEntityExtractor`组件。其他的实体提取器，类似`MitieEntityExtractor`或`SpacyEntityExtractor`当前并不能使用正则化特征。当前，所有的意图分类器可以使用所有的正则化特征。

注意：正则化特征并没有定义实体或意图。他们仅仅提供了模式用来帮助分类器识别实体和相关的意图。因此，你还是需要提供意图和实体的例子。

### 查找表（lookup tables）

查找表可以包括在训练数据中。外部提供的数据必须要以newline进行分隔。比如`plates.txt`可以包含：

```
tacos
beef
mapo tofu
burrito
lettuce wrap
```

训练文件中对应是：

```
## lookup:plates
plates.txt
```

利用list的实现是：

```
## lookup:plates
- tacos
- beef
- mapo tofu
- burrito
- lettuce wrap
```

注意：为了查找表能够有效的被使用，你的训练数据中必须要有一些示例被匹配上。否则，不会使用查找表。

警告：当使用查找表的时候，如果查找表中存在噪声，这个会损害性能。因此，确保你的查找表中的数据都是干净的。

## 数据标准化（Normalizing Data）

### 实体同义词

如果你定义的实体具有相同的名字，那么他们会被解析成同义词。如下：

```
## intent:search
- in the center of [NYC](city:New York City)
- in the center of [New York City](city)
```

正如上面所述，在两个例子中，city的值是`New York City`，尽管第一个例子中的txt是NYC。无论相同的text被找到，这个值还是会使用同义词替代消息中确切的文本。

为了在训练数据中使用同义词，需要pipeline中包含EntitySynonmMapper组件。见[Components](https://rasa.com/docs/rasa/nlu/components/#components)。

另一种的同义词的定义方式如下：

```
## synonym:New York City
- NYC
- nyc
- the big apple
```

需要注意的是：同义词的添加，并不会提高模型对实体的识别能力。**实体必须在使用同义词替换之前识别出来**。

## 生成更多的实体例子

有一些工具可以帮助生成大量的实体示例。如，[Chatito](https://rodrigopivi.github.io/Chatito/)。但是，创建人造示例，可能会出导致过拟合，更好的方式是使用查找表，而不是大量的实体值。

## 原文连接

https://rasa.com/docs/rasa/nlu/training-data-format/#training-data-format