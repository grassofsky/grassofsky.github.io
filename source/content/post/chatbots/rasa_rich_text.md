+++
date="2019-12-21"
title="rasa如何支持富文本"
categories=["chatbot","rasa"]
tags=["rasa-in-actions"]
+++

# rasa如何支持富文本

## 问题描述

对于FAQs机器人针对问题的答案可能需要很长的篇幅进行描述，那么在实现过程中，一般会先提供简单的回复描述，在附带上详细描述的链接，通过点击详细链接，给出答案的详细描述。该详细描述可以是之前通过富文本编辑的形式存储于数据库中的内容。

## 问题解决

从`rasa-sdk/CollectingDispatcher/executor.py`中可以看到，message的格式如下：

```json
{
    "text": text,
    "buttons": buttons,
    "elements": elements,
    "custom": json_message,
    "template": template,
    "image": image,
    "attachment": attachment,
}
```

支持在返回消息中，添加text，添加button，添加elements，针对定义的模板来返回消息，添加image，添加附件。

那么关于富文本的接口调用方式可以以`json_mseeage`的形式提供，

如：

```
{
  "custom": {
    "rich_text_api": ip:port/webhook/api/,
    "parameters": param
  }
}
```

然后在前端对消息进行组装。