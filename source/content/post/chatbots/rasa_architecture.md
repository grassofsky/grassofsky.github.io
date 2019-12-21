+++
date="2019-12-21"
title="对话系统rasa - 架构（翻译）"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统Rasa - 架构 [翻译]

## 消息处理

下图中描述了Rasa构建的助手在处理消息时的基本步骤：

![](https://rasa.com/docs/rasa/_images/rasa-message-processing.png)

这些步骤是：

1. 消息传入后被Interpreter接收，这个模块能够将消息转换成字典，包括原始的文本，意图，发现的实体。这部分叫做自然语言理解（NLU）。
2. Tracker用来追踪记录对话状态的对象。
3. policy接收tracker的当前状态。
4. policy选择下一个应该是什么动作。
5. 选择的动作会被记录到tracker中。
6. 结果返回给用户。

注意：输入的消息可以是用户输入的话语，也可以是按钮点击一类。

## 原文链接

https://rasa.com/docs/rasa/user-guide/architecture/