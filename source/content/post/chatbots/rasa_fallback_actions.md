+++
date="2019-12-21"
title="对话系统Rasa - fallback actions [翻译]"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统rasa - fallback actions

有些时候，你想要返回到fallback action，比如，回复“Sorry, I didn't understand that”。你可以处理fallback案例，通过添加`FallbackPolicy`或者`TwoStageFallbackPolicy`到你的policy集合中。

## Fallback policy

`FallbackPolicy`有一个fallback action，如果意图识别的结果低于`nlu_threshold`，或者没有一个对话策略预测的action的置信度高于`core_threshold`，那么这个action就会被执行。

这个阈值和fallback action可以在policy配置文件中设置，如下：

```
policies:
  - name: "FallbackPolicy"
    nlu_threshold: 0.4
    core_threshold: 0.3
    fallback_action_name: "action_default_fallback"
```

`action_default_fallback`是Rasa Core中默认的action，用来发送`utter_default`模板的消息给用户。确保在你的domain文件中定义了`utter_default`。它还将恢复到导致回退的用户消息之前的会话状态，以便不会影响对未来操作的预测。您可以查看下面的操作源：

~ ~ ~

```python
class rasa.core.actions.action.ActionDefaultFallback
```

执行fallback action，回退到之前的对话状态。

你也可以创建你自己的action来作为fallback（见[custom actions](https://rasa.com/docs/rasa/core/actions/#custom-actions) )。如果这样做，请确保在你的policy配置文件中将自定义的fallback写入到`FallbackPolicy`。比如：

```
policies:
  - name: "FallbackPolicy"
    nlu_threshold: 0.4
    core_threshold: 0.3
    fallback_action_name: "my_fallback_action"
```

**注意**：如果你的自定义fallback action并没有返回`UserUtteranceReverted`事件，你的机器的下一个预测可能是不准确的，这很有可能是fallback action没有出现在你的stories。

如果你有个特别的意图，比如`out_of_scope`，这个意图总是触发fallback action，你需要添加类似下面的story：

```
## fallback story
* out_of_scope
  - action_default_fallback
```

## Two-stage Fallback Policy

`TwoStageFallbackPolicy` 通过用户输入消除歧义来处理多个阶段低的NLU置信度（在FallbackPolicy中对于低置信度的处理也是采用相同的方式）。

- 如果NLU预测得到一个低的置信度分数，用户会被要求对意图的分类结果进行确认（默认action：action_default_ask_affirmation）
  - 如果他们确认，story继续执行
  - 如果他们拒绝，用户会被要求对他们的消息进行调整
- 重新措词（默认action：action_default_ask_rephrase）
  - 如果重新措辞后的分类结果是合理的，story继续执行
  - 如果重新措辞之后的分类结果的分数不高，用户被要求对分类的意图进行确认。
- 第二次确认（默认action：action_default_ask_affirmation）
  - 如果用户确认意图，story继续
  - 如果用户拒绝，原始的意图被分类成`deny_suggestion_intent_name`，得到最终的fallback action `fallback_nlu_action_name`被触发（如，给人的东西）

rasa core提供了`action_default_ask_affirmation`和`action_default_ask_rephrase`的默认实现。`action_default_ask_rephrase`默认实现响应的是模板 `utter_ask_rephrase`，需要确认你的domain中间中有这个模板。两个action可以通过自定义action 的方式重写[custom actions](https://rasa.com/docs/rasa/core/actions/#custom-actions)。

可以在策略配置文件中将核心回退操作和最终NLU回退操作指定为TwoStageFallbackPolicy的参数。

```
policies:
  - name: TwoStageFallbackPolicy
    nlu_threshold: 0.3
    core_threshold: 0.3
    fallback_core_action_name: "action_default_fallback"
    fallback_nlu_action_name: "action_default_fallback"
    deny_suggestion_intent_name: "out_of_scope"
```

## 原文地址

https://rasa.com/docs/rasa/core/fallback-actions/