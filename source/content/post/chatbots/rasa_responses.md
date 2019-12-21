+++
date="2019-12-21"
title="对话系统Rasa - 响应 [翻译]"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统Rasa - 响应 [翻译]

如果想要助手响应用户的消息，你需要管理这些响应。在训练数据，故事中，你定义了你的助手需要执行的动作。这些动作使用话语将消息发送给用户。

有三种方式管理这些话语：

- 将话语存储在domain中
- 检索的动作响应是训练数据的一部分，详细见：[retrieval actions](https://rasa.com/docs/rasa/core/retrieval-actions/#retrieval-actions)
- 创建自定义的NLG服务用语生成响应

## 将话语存储在domain中

默认的方式是将话语存储在domain文件中。这个文件可以包含自定义行为，一用的实体，slot和意图。

```
# all hashtags are comments :)
intents:
 - greet
 - default
 - goodbye
 - affirm
 - thank_you
 - change_bank_details
 - simple
 - hello
 - why
 - next_intent

entities:
 - name

slots:
  name:
    type: text

templates:
  utter_greet:
    - text: "hey there {name}!"  # {name} will be filled by slot (same name) or by custom action
  utter_channel:
    - text: "this is a default channel"
    - text: "you're talking to me on slack!"  # if you define channel-specific utterances, the bot will pick
      channel: "slack"                        # from those when talking on that specific channel
  utter_goodbye:
    - text: "goodbye 😢"   # multiple templates - bot will randomly pick one of them
    - text: "bye bye 😢"
  utter_default:   # utterance sent by action_default_fallback
    - text: "sorry, I didn't get that, can you rephrase it?"

actions:
  - utter_default
  - utter_greet
  - utter_goodbye
```

在domian文件中，templates包括了助手可以返回给用户消息的模板。

如果你想要改变这些文本，或响应的任何部分，你需要重新训练你的助手。

更详细的介绍可以参见：[Utterance templates](https://rasa.com/docs/rasa/core/domains/#utter-templates).

## 创建自定义的NLG服务用语生成响应

仅仅改变文本就需要进行重新训练，针对某些工作流可能不是最优的选择。这是为什么Core允许你将响应生成在外部处理与对话学习分离开来。

助手仍将学习根据过去的对话预测操作和对用户输入做出反应，但它发送回用户的响应是在rasa核心之外生成的。

如果助手发送消息给用户，它将以POST请求调用外部的HTTP服务。为了创建该endpoint，你需要创建endpoints.yml，并将它传递给`run`或者`server`脚本。`endpoints.yml`内容参见：

```
nlg:
  url: http://localhost:5055/nlg    # url of the nlg endpoint
  # you can also specify additional parameters, if you need them:
  # headers:
  #   my-custom-header: value
  # token: "my_authentication_token"    # will be passed as a get parameter
  # basic_auth:
  #   username: user
  #   password: pass
# example of redis external tracker store config
tracker_store:
  type: redis
  url: localhost
  port: 6379
  db: 0
  password: password
  record_exp: 30000
# example of mongoDB external tracker store config
#tracker_store:
  #type: mongod
  #url: mongodb://localhost:27017
  #db: rasa
  #user: username
  #password: password
```

调用`rasa run`命令的时候需要加上`enable-api`标签。

```
$ rasa run \
   --enable-api \
   -m examples/babi/models \
   --log-file out.log \
   --endpoints endpoints.yml
```

POST请求的示例如下：

```
{
  "tracker": {
    "latest_message": {
      "text": "/greet",
      "intent_ranking": [
        {
          "confidence": 1.0,
          "name": "greet"
        }
      ],
      "intent": {
        "confidence": 1.0,
        "name": "greet"
      },
      "entities": []
    },
    "sender_id": "22ae96a6-85cd-11e8-b1c3-f40f241f6547",
    "paused": false,
    "latest_event_time": 1531397673.293572,
    "slots": {
      "name": null
    },
    "events": [
      {
        "timestamp": 1531397673.291998,
        "event": "action",
        "name": "action_listen"
      },
      {
        "timestamp": 1531397673.293572,
        "parse_data": {
          "text": "/greet",
          "intent_ranking": [
            {
              "confidence": 1.0,
              "name": "greet"
            }
          ],
          "intent": {
            "confidence": 1.0,
            "name": "greet"
          },
          "entities": []
        },
        "event": "user",
        "text": "/greet"
      }
    ]
  },
  "arguments": {},
  "template": "utter_greet",
  "channel": {
    "name": "collector"
  }
}
```

endpoint将生成具体的响应：

```
{
    "text": "hey there",
    "buttons": [],
    "image": null,
    "elements": [],
    "attachments": []
}
```

接下来rasa将会把这个响应发回给用户。