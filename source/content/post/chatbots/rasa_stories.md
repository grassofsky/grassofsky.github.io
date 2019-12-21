+++
date="2019-12-21"
title="对话系统Rasa - Stories [翻译]"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# Rasa - Stories [翻译]

Rasa故事集是用来训练Rasa对话管理模型的训练数据集格式。

一个故事表示用户和AI助手之间的一次对话，其中该对话中用户的输入已经转换成对应的意图（有必要的话还带有实体），而回复是助手表达的行为名称。

Rasa核心对话系统中的一个训练示例被叫做一个故事（story）。这个是故事数据格式的指导。

注意：你可以将故事集划分成多个文件。

## 格式

这里是Rasa故事格式的一个例子：

```
## greet + location/price + cuisine + num people    <!-- name of the story - just for debugging -->
* greet
   - action_ask_howcanhelp
* inform{"location": "rome", "price": "cheap"}  <!-- user utterance, in format intent{entities} -->
   - action_on_it
   - action_ask_cuisine
* inform{"cuisine": "spanish"}
   - action_ask_numpeople        <!-- action that the bot should execute -->
* inform{"people": "six"}
   - action_ack_dosearch
```

### 什么组成了一个故事

- 一个故事以一个名字开始，如`## greet + location/price + cuisine + num people`。可以将名字命名成任何想要的文字，但是一个合理的名字有助于后期调试。
- 一个故事的结尾是newline，然后另一个故事会以`##`开始。
- 用户输入的消息以`*`打头，后面可以跟上实体的名称和值，如`intent{"entity1": "value", "entity2":"value"}`。
- 机器人执行的响应以`-`打头，后面跟着动作的名称。
- 动作返回的事件，在该动作之后，如，如果一个动作返回一个SlotSet事件，那么写成`slot{"slot_name": "value"}`。

### 用户消息

当写故事的时候，你没有必要处理用户发的消息中特殊的内容。相反，你可以利用NLU管道的输出，它允许你使用意图和实体的组合来讲用户的可以发送的所有可能的消息映射到相同的内容。

这里引入实体也是很重要的，因为用来预测下一个行为的策略是基于意图和实体的组合。（你可以使用[use_entities](https://rasa.com/docs/rasa/core/domains/#use-entities)属性改变这个设定）。

### 动作

在写故事的时候，会遇到两种类型的动作：对话和自定义的动作。对话是机器能够处理的硬编码的消息。自定义行为，是添加了自定义的执行代码。

所有的动作都是以`-`开头的。

所有的对话动作都是以`utter_`开头的，并且必须和domain中定义的模板匹配。

针对自定义动作，动作的名字可以从自定义动作类的name方法返回。尽管，对于这个命名没有规则上的限制，但是最好的实践方式是以`action_`开头。

### 事件

像插槽或表单激活/去激活的事件，必须在故事中显式的定义出来。当一个动作看上去是多余的时候，必须将返回的事件写出来。由于Rasa在训练的时候并不能确定这个现象，这一步是必须的。

关于Event的更多介绍可以见：[here](https://rasa.com/docs/rasa/user-guide/evaluating-models/#end-to-end-evaluation).

#### 插槽事件（Slot Events）

插槽事件写的样子是这样的`- slot{"slot_name": "value"}`。如何这个slot是设置在自定义的行为中，那么它需要写到自定义动作的后面。如果你的自定义动作重置了slot值，相关的事件是`-slot{"slot_name":null}`。

#### 表单事件（Form Events）

当处理故事中的表单的时候，需要记住三种类型的事件：

- 一个表单响应事件（如，`- restaurant_form`）在第一次出现一个表单之前被使用，并且当表单已经激活后，会恢复这个表单行为。
- 一个表单激活事件（如，`- form{"name":"restaurant_form"}`）在第一次表单响应事件之后被调用。
- 一个表单去激活事件（如，`- form{"name":null}`），用来去激活表单。

注意：为了避免忘记添加事件的陷阱，建议使用交互式学习（[interactive learning](https://rasa.com/docs/rasa/core/interactive-learning/#interactive-learning).）来写故事。

## 写少而简短的故事

### checkpoints

你可以使用`> checkpoints`来模块化和简化你的训练数据。checkpoints可以是有用的，但是不能过度使用他们。使用大量的checkpoints会很快的让你的故事变的难以理解。当一个故事块经常在不同的故事中重复出现，那么使用checkpoints是有用的，但是没有检查点的故事更加容易读和写。下面是包含checkpoints的示例：

```
## first story
* greet
   - action_ask_user_question
> check_asked_question

## user affirms question
> check_asked_question
* affirm
  - action_handle_affirmation
> check_handled_affirmation

## user denies question
> check_asked_question
* deny
  - action_handle_denial
> check_handled_denial

## user leaves
> check_handled_denial
> check_handled_affirmation
* goodbye
  - utter_goodbye
```

注意：不像常规的故事，checkpoints不限于从用户的输入开始，只要在故事的正确位置插入，第一个时间可以是动作或是话语。

## OR

写简短故事的另一方法是使用`OR`。如：

```
## story
...
  - utter_ask_confirm
* affirm OR thankyou
  - action_handle_affirmation
```

和checkpoints类似，出现过多的使用OR，更好的方式是重构你的领域或者意图。

警告：过度使用checkpoints和OR，会减慢训练速度。

## 端到端的故事评估格式

端到端故事格式是一种将NLU和核心培训数据合并到一个文件中进行评估的格式。你可以在这里读到更多[here](https://rasa.com/docs/rasa/user-guide/evaluating-models/#end-to-end-evaluation)。

警告：这个格式智能用来端到端的评估，不能被用来训练。

## 原文连接

https://rasa.com/docs/rasa/core/stories/#stories