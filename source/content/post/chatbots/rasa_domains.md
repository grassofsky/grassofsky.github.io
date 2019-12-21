+++
date="2019-12-21"
title="Rasa - 领域 [翻译]"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# Rasa - 领域 [翻译]

Domain（领域）定义了助手可以操控的世界。它指定了机器需要了解的`intents`,`entities`,`slots`,`actions`。可选的，它还包括机器能够说的`templates`。

## 领域示例

`DefaultDomain`如下：

```
intents:
  - greet
  - goodbye
  - affirm
  - deny
  - mood_great
  - mood_unhappy
  - bot_challenge

actions:
- utter_greet
- utter_cheer_up
- utter_did_that_help
- utter_happy
- utter_goodbye
- utter_iamabot

templates:
  utter_greet:
  - text: "Hey! How are you?"

  utter_cheer_up:
  - text: "Here is something to cheer you up:"
    image: "https://i.imgur.com/nGF1K8f.jpg"

  utter_did_that_help:
  - text: "Did that help you?"

  utter_happy:
  - text: "Great, carry on!"

  utter_goodbye:
  - text: "Bye"

  utter_iamabot:
  - text: "I am a bot, powered by Rasa."
```

### 这是什么意思呢？

你的NLU模型将定义intents和entities。

[Slots](https://rasa.com/docs/rasa/core/slots/#slots)用来保存对话中你想要记录的踪迹。一个名叫`risk_level`的分类slot可以定义成如下形式：

```
slots:
	risk_level:
		type: categorical
		values:
		- low
		- medium
		- high
```

[Here](https://rasa.com/docs/rasa/core/slots/#slot-classes) 能够找到Rasa Core中定义的slot类型，以及如何在domain文件中定义。

[Actions](https://rasa.com/docs/rasa/core/actions/#actions)是机器实际会处理的事情。举个例子，一个动作可以是：

- 对用户的回答
- 外部API的调用
- 数据库查找
- 可以是其他任何事情

## 自定义Actions和Slots

为了在你的domain中引用slots，你需要以模块路径进行引用。要引用自定义action，可以使用他们的名字。比如，如果你有一个模块叫做my_actions，这个模块里面有一个类叫做MyAwesomeAction,还有一个模块my_slots，含有一个类MyAwesomeSlot，那么在domain文件中，你需要添加下面的内容：

```
actions:
	- my_custom_action
	...
	
slots:
	- my_slots.MyAwesomeSlot
```

这个例子中MyAwesomeAction类的name函数必要返回my_custom_action。更加详细的介绍可以见：[Custom Actions](https://rasa.com/docs/rasa/core/actions/#custom-actions)。

## 对话模板

对话模板带有机器返回给用户的消息。这里有两种方式使用这些模板：

1. 如果模板中的名字以`utter_`开头，那么该言论可以直接用作action。你将添加类似下面的模板到domain中：

   ```
   templates:
   	utter_greet:
   	- text: "Hey! How are you?"
   ```

   随后，在stories中可以这么使用：

   ```
   ## greet the user
   * intent_greet
     - utter_greet
   ```

   当`utter_greet`按照action运行的时候，它将从模板中发送消息给用户。

2. 你可以使用模板从自定义的actions中产生响应消息，具体用`dispatcher.utter_template("utter_greet", tracker)`。这个允许你将产生消息的逻辑从实际拷贝中移出。在你的自定义action代码中，你可以基于模板发送消息，如下：

   ```python
   from rasa_sdk.actions import Action
   
   class ActionGreet(Action):
       def name(self):
           return 'action_greet'
       
       def run(self, dispatcher, tracker, domain):
           dispatcher.utter_template("utter_greet", tracker)
           return []
   ```

## Images和Buttons

在domain定义文件中可以包含图像和按钮，如下：

```
tempaltes:
	utter_greete:
	- text: "Hey! How are you?"
	  buttons:
	  - title: "great"
	    payload: "great"
	  - title: "super sad"
	    payload: "super sad"
	utter_cheer_up:
	- text: "Here is something to cheer you up:"
	  image: "https://i.imgur.com/nGF1K8f.jpg"
```

注意：这个功能取决于输出平道是否支持button的显示。如命令行模式，会通过以输出选项的方式，替换图像和按钮的显示。

## 自定义输出

你可以使用custom将任意的响应输出到输出通道。

比如，尽管日期选择器在template模板中并没有定义成参数，但是slack date picker可以通过下面的方式发送：

```
templates:
  utter_take_bet:
  - custom:
      blocks:
      - type: section
        text:
          text: "Make a bet on when the world will end:"
          type: mrkdwn
        accessory:
          type: datepicker
          initial_data: '2019-05-21'
          placeholder:
            type: plain_text
            text: Select a date
```

## 特定频道的话语

如果你有某个话语只需要发送到特定频道，可以使用`channel`关键词进行限定。value需要和channel类(OutputChannel类)的name函数返回的名字相匹配。这个功能当你需要创建的自定义的输出只针对某个频道的时候就很有用。

```
templates:
  utter_ask_game:
  - text: "Which game would you like to play?"
    channel: "slack"
    custom:
      - # payload for Slack dropdown menu to choose a game
  - text: "Which game would you like to play?"
    buttons:
    - title: "Chess"
      payload: '/inform{"game": "chess"}'
    - title: "Checkers"
      payload: '/inform{"game": "checkers"}'
    - title: "Fortnite"
      payload: '/inform{"game": "fortnite"}'
```

每一次机器查找言论的时候，首先会确认是模板中是否有针对特定频道的言论。如果有，只会选择该言论。如果没有找到，将从没有定义channel的言论中挑选。因此，针对每个言论都定义一个没有channel的情况是一个好的习惯。

## 变量

在模板中也可以使用变量来表示对话中收集的内容。你可以在python代码中添加或使用自动slot填充机制。如有下面的模板：

```
templates:
  utter_greet:
  - text: "Hey, {name}. How are you?"
```

Rasa会自动的用叫做name的slot将变量进行填充。

在自定义代码中实现方式为：

```python
class ActionCustom(Action):
    def name(self):
        return "action_custom"
    
    def run(self, dispatcher, tracker, domain):
        # send utter default template to user
        dispatcher.utter_template("utter_default", tracker)
        # .. other code
        return []
```

如果模板包含`{my_variable}`的变量，你可以通过参数的形式传递给`utter_template`

```python
dispatcher.utter_template("utter_default", tracker, my_variable="my text")
```

## 可变性

如果你想要发送给用户的回复是随机的，那么可以将多个回复以列表的形式给出，Rasa会随机地进行选择，如：

```
templates:
  utter_greeting:
  - text: "Hey, {name}. How are you?"
  - text: "Hey, {name}. How is your day going?"
```

## 针对某种意图忽略实体（Ignoring entities for certain intents）

如果你想要忽略某个意图的所有实体，你可以添加`use_entities: []`，如下：

```
intents:
  - greet:
  	  use_entities: []
```

如果只需要忽略部分实体，如下：

```
intents:
- greet:
	use_entities:
	  - name
	  - first_name
	ignore_entities:
	  - location
	  - age
```

这意味着，这些意图中的被排除的实体是未经处理的，因此不会影响下一个动作的预测。当你有不关心一个意图中被挑选出来的实体的时候，这是有用的。如果您在没有这个参数的情况下将您的意图列为normal，那么实体将被特性化为normal。

注意：如果你确实想要这些实体不影响动作预测，建议使用具有相同名字，类型为unfeaturized的slots。

## 原文连接

https://rasa.com/docs/rasa/core/domains/#domains