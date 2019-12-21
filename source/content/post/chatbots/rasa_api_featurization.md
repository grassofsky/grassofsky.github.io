+++
date="2019-12-21"
title="对话系统rasa - Featurization （翻译）"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统rasa - Featurization （翻译）

为了将机器学习算法应用到对话AI，我们需要构建对话的词向量表示方式。

每个故事都对应一个tracker，该tracker由每次action前的对话状态组成。

## State Featurizers

trackers历史中的每个event都会创建一个新的状态（如，运行bot action，接收用户消息，设置槽位值）。特征化tracker中的单个状态，有以下几个步骤：

### 1. Tracker提供的一系列的 active features，有：

- 表示intents和entitiies的feature，如果这是一个Turn中的第一个状态，如，这是解析用户消息后我们将采取的第一个操作。（如，`[intent_restaurant_search, entity_cuisine]`）。
- 表示当前定义的slots的feature，如，如果用户之前提到了他们要查找的restaurants的位置，`slot_location`。
- 表示存储在插槽中的任何api调用结果的特性，如，slot_matches
- 表示最后一个action是什么的特征，如，`pre_action_listen`

### 2. 将Feature转换成数值数组：

对于监督式学习我们使用常见的标记，X和y。X的形状是`(num_data_points, time_dimension, num_input_features)`和y的形状是`(num_data_points, num_bot_features)`或者`(num_data_points, time_dimension, num_bot_features)`包含被编码为one-hot向量的目标类别标签。

目标标签与bot将采取的action相关。将features转换成向量形式，这里有不同的featurizers：

- `BinarySingleStateFeaturizer`创建二进制one-hot编码：向量X,y表示某个意图、实体、先前的action或slot，如，`[0 0 1 0 0 1 ...]`.
- `LabelTokenizerSingleStateFeaturizer` 基于feature label创建一个向量：所有的active feature标签（如，pre_action_listen）被划分成token，并且被表示成词袋的形式。如，actions `utter_explane_details_hotel`和`utter_explain_details_restaurant`通常将有3个features，不同的是一个单独的feature表示一个domain。用于用户输入的labels（intents，entities）和机器的action被单独的进行featurized。两个类别的标签被特殊的字符（`split_symbol`）进行分词（如，`action_search_restaurant = {action, search, restaurant}`），创建两个词汇表。使用适合的词汇表，针对每个label会创建一个词袋的表示形式。slot被featurized成二进制向量，表示他们在对话的每个步骤中出现了还是没有出现。

**注意**：如果领域中定义了可能的`actions`，`[ActionGreet, ActionGoodbye]`，`4`额外的默认的actions会添加：`[ActionListen(), ActionRestart(), ActionDefaultFallback(), ActionDeactivateForm()]`。因此，label 0表示action listen，1表示restart，2表示greeting，3表示goodbye。

## Tracker Featurizers

在预测一个动作时，包含比当前状态多一点的历史记录通常是有用的。TrackerFeaturizer遍历跟踪程序状态，并未每个状态调用一个SingleStateFeaturizer。有两种不同的tracker featurizers：

### 1. Full Dialogue

`FullDialogueTrackerFeaturizer` 创建故事的数值表示，以馈送到一个递归神经网络，其中整个对话馈送到一个网络，并且梯度从所有时间步反向传播。因此X数组的形状是`(num_stories, max_dialogue_length, num_input_features)`，y的形状是`(num_stories, max_dialogues_length, num_bot_features)`。对于所有功能，较小的对话都用-1填充，表示没有策略值。

### 2. Max History

`MaxHistoryTrackerFeaturizer` 为每个bot操作或语句创建一个以前的跟踪器状态数组，参数max_history定义X中每行的状态数。执行重复数据消除，是为了根据他们之前的状态过滤掉重复的turns（bot actions或者bot utterances）。因此X的形状是：`(num_unique_turns, max_history, num_input_features)`，y的形状是`(num_unique_turns, num_bot_features)`。

针对一些算法，需要一个flat feature向量，因此X可以reshape成`(num_unique_turns,max_history * num_input_features)`。如果需要数字目标类标签而不是一个热向量，可以使用`y.argmax(axis=-1)`。