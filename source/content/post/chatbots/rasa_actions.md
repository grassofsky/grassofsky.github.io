+++
date="2019-12-21"
title="对话系统Rasa - Actions [翻译]"
categories=["chatbot","rasa"]
tags=["rasa-doc"]
+++

# 对话系统Rasa - Actions [翻译]

Actions是机器针对用户输入的响应。Rasa中有四种actions：

- Utterance actions：以`utter_`开头，用来发送特定的消息给用户。
- Retrieval actions：以`respond_`开头，用来发送检索模型挑选的消息。
- Custom actions：运行任意代码，发送任意数量的消息。
- Default actions：如，`action_listen`,`action_restart`,`action_default_fallback`。

## Utterance action

为了定义utterance action（`ActionUtterTemplate`），需要在domain文件中添加utterance template，以`utter_`开头：

```
templates:
  utter_my_message:
    - "this is what I want my action to say!"
```

utterance action以`utter_`开头是一种惯例。如果，没有这个前缀，你仍然可在你的自定义行为中使用模板，但是模板不能直接被预测为自身的行为。详细见：[Responses](https://rasa.com/docs/rasa/core/responses/#responses) 。

如果你是用外部的NLG服务，你没有必要在domain文件中定义模板，但是你仍然可以在actions列表中添加utternace名字。

## Retrieval Actions

retrieval actions是的对大量相似意图的对话的实现变得简单（如闲聊和FAQs）。详细见：[Retrieval Actions](https://rasa.com/docs/rasa/core/retrieval-actions/#retrieval-actions)。

## Custom Actions

自定义action可以运行任何你想要执行的代码。自定义action可以开灯，可以给日历添加事件，可以检查用户的银行账单，或者任何你可以想到的事情。

当custom action被预测的时候，rasa将调用你指定的一个终端。这个端点应该是一个web服务器，它对这个调用作出反应，运行代码，并有选择地返回信息来修改对话状态。

为了指定终端，你的action服务使用`endpoints.yml`：

```
action_endpoint:
  url: "http://localhost:5055/webhook"
```

在命令行中可以用`--endpoints endpoints.yml`进行指定。

你可以使用node.js，.Net，Java或者任何其他语言来创建action server，以及定义你的actions，但是我们提供了简单的Python sdk，使得action的创建相当简单。

**注意**：rasa使用了ticket lock mechanism来确保来自同一个ID的消息不会和其他消息串扰，保证能够以正确的顺序进行处理。如果你的自定义行为可能需要超过60s的时间执行，请将环境变量`TICKET_LOCK_LIFETIME`设置成你期望的值。

### 用python实现Custom Actions

我们提供了sdk，方便使用python实现actions。

action服务唯一要做的事情是安装`rasa-sdk`：`pip install rasa-sdk`

**注意**：你没有必要在action服务器上安装rasa。建议在docker环境中运行rasa。并为你的action服务器创建单独的容器。在这个单独的容器里面，你只需要安装`rasa-sdk`。

这个带有你自定义actions的文件应该是`actions.py`。

如果你安装了rasa，那么可以通过下面的命令运行你的action服务：

```
rasa run actions
```

否则，可以使用下面的命令：

```
python -m rasa_sdk --actions actions
```

对于餐厅机器人，如果一个用户说“show me a Mexican restaurant”，你的机器人可以执行`ActionCheckRestaurants`，实现可能是这样的：

```python
from rasa_sdk import Action
from rasa_sdk.events import SlotSet

class ActionCheckRestaurants(Action):
   def name(self) -> Text:
      return "action_check_restaurants"

   def run(self,
           dispatcher: CollectingDispatcher,
           tracker: Tracker,
           domain: Dict[Text, Any]) -> List[Dict[Text, Any]]:

      cuisine = tracker.get_slot('cuisine')
      q = "select * from restaurants where cuisine='{0}' limit 1".format(cuisine)
      result = db.query(q)

      return [SlotSet("matches", result if result is not None else [])]
```

在你的domain文件中，应该添加名为`action_check_restaurants`的action。这个actions的run方法接收三个参数。你可以使用tracker对象访问slots的值，和用于最近发送的消息。，然后通过dispatcher对象将消息发送给用户，具体的调用是`dispatcher.utter_template, dispatcher.utter_message`或者任意其他的`rasa_sdk.executor.CollectingDispatcher`方法。

`run()`方法的消息介绍：

名称：`Action.run(dispatcher, tracker, domain)`

参数介绍：

- dispatcher (CollectionDispatcher) - dispatcher用来将消息发送给用户。使用`dispatcher.utter_message`或任何其他的`rasa_sdk.executor.CollectiongDispatcher`方法。
- tracker (Tracker) - 当前用户的状态记录。你可通过`tracker.get_slot(slot_name)`获取slot的值，使用`tracker.latest_message.text`获取最近的用户消息，以及其他的`rasa_sdk.Tracker`属性。
- domain (Dict[Text, Any]) - 机器的domain

返回值：通过终端返回的`rasa_sdk.events.Event`示例的字典。

返回类型：List[Dict[Text, Any]]

所有可能的Events见，https://rasa.com/docs/rasa/api/events/#events

## 在其他的代码中执行Action

Rasa会发送post请求给运行action的服务。此外，这个请求会包含所有的对话信息。[Action Server](https://rasa.com/docs/rasa/api/action-server/#action-server)给出了详细的API说明。

对于action请求的回复，可以对tracker进行修改，如，设置slots，给用户发送回复。所有的更改都是用events。详细的事件类型见：[Events](https://rasa.com/docs/rasa/api/events/#events).

## 是用action主动联系用户

你也许需要主动联系用户，如显示在后台运行很长时间的输出，或者通知用户外部事件。

为了实现这点，你可以往终端发送POST请求，指定需要针对特定用户执行的action。使用`output_channel`查询参数，指定哪个输出通道被用来助手对用户的响应。如果你的消息是静态的，你可以在domain文件中通过模板定义`utter_` action。任何自定义行为指派的响应将会提交到指定的输出通道。

主动联系用户依赖于通道的可用性，因此并不被所有的通道支持。如果你的通道不支持，可以考虑使用`CallbackInput`通道将消息发给`webhook`。

**注意**：在对话中运行一个action会改变对话历史，会影响助手下一次的预测。如果你不想这件事情发生，确保你的响应将自己回退，可以通过将`ActionReverted`事件添加到对话记录中实现。

## 默认Actions

默认的actions有：

- `action_listen`：停止预测更多actions，等待用户输入。
- `action_restart`：重置整个对话，如果 [Mapping Policy](https://rasa.com/docs/rasa/core/policies/#mapping-policy)包含在policy配置中，那么可以在对话中通过输入`/restart`触发。
- `action_default_fallback`：回退上一次用户的消息（就好像用户没有发送，机器也没有做出反应），和说出一个机器不理解的消息。参见： [Fallback Actions](https://rasa.com/docs/rasa/core/fallback-actions/#fallback-actions).
- `action_deactivate_form`：停止active form，并重置请求的slot。参见：[Handling unhappy paths](https://rasa.com/docs/rasa/core/forms/#section-unhappy).
- `action_revert_fallback_events`：当有TwoStageFallbackPolicy时，还原发生的事件，参见：[Fallback Actions](https://rasa.com/docs/rasa/core/fallback-actions/#fallback-actions).
- `action_default_ask_affirmation`：询问用户确认他们的意图。建议重写这个默认行为，获得更加有意义的提示。
- `action_default_ask_rephrase`：询问用户重述他们的意图。
- `action_back`：回退上一次用户的消息（就好像用户没有发送，机器也没有做出反应）。如果 [Mapping Policy](https://rasa.com/docs/rasa/core/policies/#mapping-policy)包含在policy配置中，可以在对话中被`/back`触发。

所有的默认action都可以被重写。只要将action name添加到domain中：

```
actions:
- action_default_ask_affirmation
```

rasa然后会调用你的action endpoint，并将其视为自定义action。

## 原文链接

https://rasa.com/docs/rasa/core/actions/#