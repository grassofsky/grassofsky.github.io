+++
date="2019-12-21"
title="对话系统Rasa - Events"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统rasa - Events

Rasa中的对话系统表示成一系列的events。这一页会对Rasa Core中定义的event类型进行介绍。

**注意** 如果你使用Rasa SDK，用python来写自定义actions。你需要从`rasa_sdk.events`导入events，而不是从`rasa.core.events`。如果你使用其他的编程语言进行实现，你的events应该格式化成类似这篇介绍中给出的JSON对象。

## 目录

- 通用目的的Events
  - 设置槽位
  - 重启对话
  - 重置槽位
  - 安排提醒
  - 暂停对话
  - 继续对话
  - 强制采取后续行动
- 自动追踪events
  - 用户发送消息
  - 机器回复消息
  - 撤销用户消息
  - 撤销action
  - 记录一个执行的action

## 通用目的的Events

### 设置槽位

**简介**： 用来设置slot到tracker上的event

**JSON**：`evt = {"event": "slot", "name": "departure_airport", "value": "BER"}`

**Class**：

`class rasa.core.events.SlotSet(key, value=None, timestamp=None)`

用户针对slot的值可以设定首选项。

每一个slot都有一个name和一个value。这个event可以用来设置对话中slot的值。

作为边际效果，`Tracker`的slot将被更新，如`tracker.slots[key]=value`。

**Effect**：

当添加到tracker，下面的代码会用来更新tracker：

```python
def apply_to(self, tracker):
    tracker._set_slot(self.key, self.value)
```

### 重启对话

**简介**：重置记录在tracker中的所有内容

**JSON**：`evt = {"event": "restart"}`

**Class**：

`class rasa.core.events.Restarted(timestamp=None)`

谈话应该重新开始，历史应该被抹去。

除了删除所有的events，这个event可以被用来重置trackers的状态（如，忽略任何过去的用户消息并重置所有插槽）。

**Effect**：

当添加到tracker的时候，下面的代码可以用来更新tracker：

```python
def apply_to(self, tracker):
    from rasa.core.actions.action import ( # pytype: disable=pyi-error
    	ACTION_LISTEN_NAME,
    )
    tracker._reset()
    tracker.trigger_followup_action(ACTION_LISTEN_NAME)
```

### 重置槽位

**简介**：

重置对话的所有槽位。

**JSON**：`evt = {"event": "reset_slots"}`

**Class**：

`class rasa.core.events.AllSlotsReset(timestamp=None)`

所有的槽位都被重置成初始化值。

如果你想要保留对话历史，仅想要重置槽位，那么可以使用这个event将槽位都重置成他们的初始值。

**Effect**：

当添加到tracker的时候，这个代码需要被用来更新tracker：

```python
def apply_to(self, tracker):
    tracker._reset_slots()
```

### 安排提醒

**简介**：安排一个action在未来执行

**JSON**：

```json
evt={
      "event": "reminder",
      "action": "my_action",
      "date_time": "2018-09-03T11:41:10.128172",
      "name": "my_reminder",
      "kill_on_user_msg": True,
    }
```

**Class**：

`class rasa.core.events.ReminderScheduled(action_name,trigger_date_time,name=None,kill_on_user_message=True,timestamp=None)`

允许异步安排很多的action执行。

作为边际效果，消息处理器将安排一个action在触发时间执行。

**Effect**：当添加到tracker是，core将安排action在未来的某个时间执行。

### 暂停对话

**简介**：停止机器人对消息的响应。action预测将停止，直到恢复

**JSON**：`evt = {"event": "pause"}`

**Class**：

`class rasa.core.events.ConversationPaused(timestamp=None)`

忽略来自用户的消息，让人接管。

作为边际效果，Tracker的paused属性将会被设置为True。

**Effect**：

当添加到tracker的时候，下面的代码被用来更新tracker：

```python
def apply_to(self, tracker):
    tracker._paused = True
```

### 继续对话

**简介**：继续之前暂停的对话。机器人将再开始预测actions。

**JSON**：`evt = {"event": "resume"}`

**Class**：

`class rasa.core.events.ConversationResumed(timestamp=None)`

bot将接管对话。

功效与`PauseConversation`相反。作为边际效果，Tracker的paused属性将被设置成False。

**Effect**：

当添加到tracker的时候，下面的代码会被用来更新tracker：

```python
def apply_to(self, tracker):
    tracker._paused = False
```

### 强制采取后续行动

**简介**：与预测下一个action相反，强制执行下一个action

**JSON**：`evt = {"event": "followup", "name": "my_action"}`

**Class**：

`class rasa.core.events.FollowupAction(name, timestamp=None)`

安排后续行动。

**Effect**：

当添加到tracker的时候，下面的代码会被用来更新tracker：

```python
def apply_to(self, tracker: "DialogueStateTracker") -> None:
    tracker.trigger_followup_action(self.action_name)
```

## 自动追踪events

### 用户发送消息

**简介**：用户发送给机器的消息

**JSON**：

```json
evt={
      "event": "user",
      "text": "Hey",
      "parse_data": {
        "intent": {
          "name": "greet",
          "confidence": 0.9
        },
        "entities": []
      },
      "metadata": {},
    }
```

**Class**：

`class rasa.core.events.UserUttered(text=None, intent=None, entities=None, parse_data=None, timestamp=None, input_channel=None, message_id=None, metadata=None)`

用户向机器说了些内容。

作为边际效果，Tracker中会创建新的Turn。

**Effect**：

当添加到tracker的时候，下面的代码会被用来更新tracker：

```python
def apply_to(self, tracker: "DialogueStateTracker") -> None:
    tracker.latest_message = self
    tracker.clear_followup_action()
```

### 机器回复消息

**简介**：机器发送给用户的消息

**JSON**：`evt = {"event": "bot", "text": "Hey there!", "data": {}}`

**Class**：

`class rasa.core.events.BotUttered(text=None,data=None,metadata=None,timestamp=None)`

bot向用户说了一些内容。

因为这个类包含在ActionExecuted类，它不用于story训练。作为Tracker的一个条目。

**Effect**：

当添加到tracker的时候，下面的代码会被用来更新tracker：

```python
def apply_to(self, tracker: "DialogueStateTracker") -> None:
    tracker.latest_bot_utterance = self
```

### 撤销用户消息

**简介**：撤消上次用户消息（包括消息的用户事件）之后发生的所有副作用。

**JSON**：`evt = {"event": "rewind"}`

**Class**：

`class rasa.core.events.UserUtteranceReverted(timestamp=None)`

bot会将所有内容还原到最新用户消息之前。

机器会还原最近UserUttered之后的所有events。这也意味着，tracker上的最后一个event通常是`action_listen`，bot正在等待新的用户消息。

**Effect**：

当添加到tracker的时候，下面的代码会被用来更新tracker：

```python
def apply_to(self, tracker: "DialogueStateTracker") -> None:
    tracker._reset()
    tracker.replay_events()
```

### 撤销action

**简介**：撤销上次action之后发送能能的所有副作用（包括action的action事件）。

**JSON**：`evt = {"event": "undo"}`

**Class**：

`class rasa.core.events.ActionReverted(timestamp=None)`

bot撤销上次的action。

机器人会将所有内容还原到最近一次操作之前。这包括操作本身，以及操作创建的任何事件，如设置槽事件-机器人现在将使用最新操作之前的状态预测新操作。

**Effect**：

当添加到tracker的时候，下面的代码会被用来更新tracker：

```python
def apply_to(self, tracker: "DialogueStateTracker") -> None:
    tracker._reset()
    tracker.replay_events()
```

### 记录一个执行的action

**简介**：将机器人执行的操作记录到对话中。创建的操作将单独记录事件。

**JSON**：`evt = {"event": "action", "name": "my_action"}`

**Class**：

`class rasa.core.events.ActionExecuted(action_name, policy=None, confidence=None, timestamp=None)`

用来描述一个action执行和结果的操作。

它包括一个动作和一系列事件。操作将附加到tracker.turns中的最新Turn。

**Effect**：

当添加到tracker的时候，下面的代码会被用来更新tracker：

```python
def apply_to(self, tracker: "DialogueStateTracker") -> None:
    tracker.set_latest_action_name(self.action_name)
    tracker.clear_followup_action()
```



