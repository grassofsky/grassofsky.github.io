+++
date="2019-12-21"
title="对话系统rasa - retrieval actions"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统rasa - retrieval actions

**警告**：该功能还处于试验阶段。这个功能在未来的版本中可能被移除或改变。可以通过 [forum](https://forum.rasa.com/)进行反馈。

**注意**：[here](https://blog.rasa.com/response-retrieval-models/)这里有详细的博客，关于如何利用retrieval actions 处理单轮交互。

## About

Retrieval actions设计用于更好的处理短对话（[Small Talk](https://rasa.com/docs/rasa/dialogue-elements/small-talk/#small-talk)）和简单问答的场景（[Simple Questions](https://rasa.com/docs/rasa/dialogue-elements/completing-tasks/#simple-questions)）。比如，如果你的助手需要处理100个FAQs和50个不同的small talk意图，你可以使用单个retrieval action来覆盖所有的这些场景。从对话的视角看，想这些单轮对话的模式是一致的，因此可以简化你的stories。

没有retrieval actions的时候，你可能会这样设计你的stories：

```
## weather
* ask_weather
  - utter_ask_weather
  
## introduction
* ask_name
  - utter_introduce_myself
  
...
```

你可以使用单个story包括上面所有的意图：

```
## chitchat
* chitchat
  - respond_chitchat
```

retrieval action使用了NLU中的[Response Selector](https://rasa.com/docs/rasa/nlu/components/#response-selector) 组件，该组件用来学习如何根据用户的输入从一系列候选的回复中选出合适的回复。

## Training Data

retrieval action学习如何根据用户的输入从一系列候选的回复中选出合适的回复。和其他的NLU数据类似，你需要在NLU文件中包括对应的示例，如下：

```
## intent: chitchat/ask_name
- what's your name
- who are you?
- what are you called?

## intent: chitchat/ask_weather
- how's weather?
- is it sunny where you are?
```

首先，所有这些示例会被合并成单个`chitchat`的retrieval意图。所有检索意图都添加了一个后缀，用于标识助手的特定响应文本，上面的例子中是`ask_name`和`ask_weather`。这些后缀通过`/`和意图的名字分离。

接着，将所有的retrieval意图的回复消息写入到单个训练数据中，名为`responses.md`：

```
## ask name
* chitchat/ask_name
  - my name is Sara, Rasa's documentation bot!
  
## ask weather
* chitchat/ask_weather
  - it's always sunny where I live
  
```

检索模型作为NLU训练管道的一部分单独训练，以选择正确的响应。需要记住的一件重要事情是，检索模型使用响应消息的文本来选择正确的文本。如果更改这些响应的文本，则必须重新训练检索模型！这是域文件中响应模板的一个关键区别。

**注意**：含有回复文本的文件必须单独存在于训练数据目录中。并且该文件中的训练数据不能包含其他组件使用的数据。

**注意**：正如上面的例子显示，`/`符号被用来作为回复文本的分隔符。确保在其他的意图名字中使用它。

## Config File

在你的配置文件中需要加上`ResponseSelector`组件。这个组件之前应该进行分词，特征化，和意图识别。如下：

```
language: "en"

pipeline:
- name: "WhitespaceTokenizer"
  intent_split_symbol: "_"
- name: "CountVectorsFeaturizer"
- name: "EmbeddingIntentClassifier"
- name: "ResponseSelector"
```

## Domain

rasa使用命名约定将诸如chitchat/ask_name之类的意图名称与检索操作相匹配。正确的action名字是`respond_chitchat`。`respond_`前缀强制被用来标记retrieval action。另一个示例，对于`faq/ask-policy`正确的action名字是`respond_faq`，如下：

```
actions:
  ...
  - respond_chitchat
  - respond_faq
```

确保检索操作在chitchat意图之后被预测的一个简单方法是使用映射策略( [Mapping Policy](https://rasa.com/docs/rasa/core/policies/#mapping-policy))。但是，您也可以在您的故事中包含此操作。例如，如果您想在处理chitchat后重复一个问题（请参见不愉快的路径[Unhappy Paths](https://rasa.com/docs/rasa/dialogue-elements/completing-tasks/#unhappy-paths)）。

```
## interruption
* search_restaurant
   - utter_ask_cuisine
* chitchat
   - respond_chitchat
   - utter_ask_cuisine
```

## Multiple Retrieval Actions

如果你的助手包括FAQs和chitchat，可以将其拆分为各自独立的retrieval actions，比如`chitchat/ask_weather`和`faq/returns_policy`。rasa支持添加多个RetrievalActions，比如`respond_chitchat`和`respond_returns_policy`。为了支持分别训练，你需要在config文件中做如下修改：

```
language: "en"

pipeline:
- name: "WhitespaceTokenizer"
  intent_split_symbol: "_"
- name: "CountVectorsFeaturizer"
- name: "EmbeddingIntentClassifier"
- name: "ResponseSelector"
  retrieval_intent: chitchat
- name: "ResponseSelector"
  retrieval_intent: faq
```

当然你可以公用相同的retrieval模型。这样，只需要将`retrieval_intent`设定为默认值（None）。

```
language: "en"

pipeline:
- name: "WhitespaceTokenizer"
  intent_split_symbol: "_"
- name: "CountVectorsFeaturizer"
- name: "EmbeddingIntentClassifier"
- name: "ResponseSelector"
```

在这种情况下，响应选择器将接受来自chitchat/{x}和faq/{x}的示例的训练，并通过名称default（NLU解析的输出）来标识。

在当前的进展，使用分离的模型和使用单个模型对于retrieval action的准确度上并没有任何区别。为了便捷，我们建议你使用单个retrieval模型。如果你有不同的答案，请让我们知道[forum](https://forum.rasa.com/) !

## Parsing Response Selector Output

通过NLU解析出来的输出有一个`response_selector`属性，用来包含每个response selector的输出。每一个response selector由该response selector的参数retrieval_intent标识，并存储两个属性。

- response：预测出来的回复文本和置信度
- ranking：置信度评分前10的候选回复

示例结果如下：

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

如果特定响应选择器的retrieval_intent参数保留为其默认值，则相应的响应选择器将在返回的输出中标识为default。

## 原文链接

https://rasa.com/docs/rasa/core/retrieval-actions

