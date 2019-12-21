+++
date="2019-12-21"
title="rasa - http api测试"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# rasa - http api测试

此处主要介绍如何对http api进行测试。关于http api的详细介绍可以参见帮助文档：https://rasa.com/docs/rasa/api/http-api/

对于http接口进行测试可以使用postman进行，该软件的下载地址如下：http://www.downza.cn/soft/205171.html。关于该工具的使用可以参见：https://blog.csdn.net/fxbin123/article/details/80428216。

下面介绍一下针对rasa的一个例子（[对话系统rasa示例简析 - Knowledge Base Bot](https://zhuanlan.zhihu.com/p/84405045)）说明如何利用http api进行简单对话(仅仅为了说明http api如何进行调用以及测试)。

首先执行`rasa train`进行训练，接着执行`rasa run actions`运行actions服务，最后执行`rasa run --enable-api`运行rasa服务。

在shell中直接输入对话，就会返回结果，在这里并没有直接的api，需要分几个api进行写作调用。

## 1. post请求，添加messages

请求地址：`localhost:5005/conversations/0/messages`

请求内容：

```json
{
    "text": "Hello!",
    "sender": "user"
}
```

响应：

```json
{
    "sender_id": "0",
    "slots": {
        "attribute": null,
        "city": null,
        "cuisine": null,
        "hotel": null,
        "knowledge_base_last_object": null,
        "knowledge_base_last_object_type": null,
        "knowledge_base_listed_objects": null,
        "mention": null,
        "object_type": null,
        "restaurant": null
    },
    "latest_message": {
        "intent": {
            "name": "greet",
            "confidence": 0.9873532057
        },
        "entities": [],
        "intent_ranking": [
            {
                "name": "greet",
                "confidence": 0.9873532057
            },
            {
                "name": "goodbye",
                "confidence": 0.0096256845
            },
            {
                "name": "query_knowledge_base",
                "confidence": 0.0015526779
            },
            {
                "name": "bot_challenge",
                "confidence": 0.0014684227
            }
        ],
        "text": "Hello!"
    },
    "latest_event_time": 1571646907.7877626419,
    "followup_action": null,
    "paused": false,
    "events": [
        {
            "event": "action",
            "timestamp": 1571646907.634185791,
            "name": "action_listen",
            "policy": null,
            "confidence": null
        },
        {
            "event": "user",
            "timestamp": 1571646907.7877626419,
            "text": "Hello!",
            "parse_data": {
                "intent": {
                    "name": "greet",
                    "confidence": 0.9873532057
                },
                "entities": [],
                "intent_ranking": [
                    {
                        "name": "greet",
                        "confidence": 0.9873532057
                    },
                    {
                        "name": "goodbye",
                        "confidence": 0.0096256845
                    },
                    {
                        "name": "query_knowledge_base",
                        "confidence": 0.0015526779
                    },
                    {
                        "name": "bot_challenge",
                        "confidence": 0.0014684227
                    }
                ],
                "text": "Hello!"
            },
            "input_channel": null,
            "message_id": "cefc86543bbc410fa6197f67618bccbe",
            "metadata": null
        }
    ],
    "latest_input_channel": null,
    "active_form": {},
    "latest_action_name": "action_listen"
}
```

## 2. post请求，预测action

请求地址：`localhost:5005/conversations/0/predict`

响应：

```json
{
    "scores": [
        {
            "action": "utter_greet",
            "score": 1
        },
        {
            "action": "action_back",
            "score": 0
        },
        {
            "action": "action_deactivate_form",
            "score": 0
        },
        {
            "action": "action_default_ask_affirmation",
            "score": 0
        },
        {
            "action": "action_default_ask_rephrase",
            "score": 0
        },
        {
            "action": "action_default_fallback",
            "score": 0
        },
        {
            "action": "action_listen",
            "score": 0
        },
        {
            "action": "action_query_knowledge_base",
            "score": 0
        },
        {
            "action": "action_restart",
            "score": 0
        },
        {
            "action": "action_revert_fallback_events",
            "score": 0
        },
        {
            "action": "utter_ask_rephrase",
            "score": 0
        },
        {
            "action": "utter_goodbye",
            "score": 0
        },
        {
            "action": "utter_iamabot",
            "score": 0
        }
    ],
    "policy": "policy_0_MemoizationPolicy",
    "confidence": 1,
    "tracker": {
        "sender_id": "0",
        "slots": {
            "attribute": null,
            "city": null,
            "cuisine": null,
            "hotel": null,
            "knowledge_base_last_object": null,
            "knowledge_base_last_object_type": null,
            "knowledge_base_listed_objects": null,
            "mention": null,
            "object_type": null,
            "restaurant": null
        },
        "latest_message": {
            "intent": {
                "name": "greet",
                "confidence": 0.9873532057
            },
            "entities": [],
            "intent_ranking": [
                {
                    "name": "greet",
                    "confidence": 0.9873532057
                },
                {
                    "name": "goodbye",
                    "confidence": 0.0096256845
                },
                {
                    "name": "query_knowledge_base",
                    "confidence": 0.0015526779
                },
                {
                    "name": "bot_challenge",
                    "confidence": 0.0014684227
                }
            ],
            "text": "Hello!"
        },
        "latest_event_time": 1571646907.7877626419,
        "followup_action": null,
        "paused": false,
        "events": [
            {
                "event": "action",
                "timestamp": 1571646907.634185791,
                "name": "action_listen",
                "policy": null,
                "confidence": null
            },
            {
                "event": "user",
                "timestamp": 1571646907.7877626419,
                "text": "Hello!",
                "parse_data": {
                    "intent": {
                        "name": "greet",
                        "confidence": 0.9873532057
                    },
                    "entities": [],
                    "intent_ranking": [
                        {
                            "name": "greet",
                            "confidence": 0.9873532057
                        },
                        {
                            "name": "goodbye",
                            "confidence": 0.0096256845
                        },
                        {
                            "name": "query_knowledge_base",
                            "confidence": 0.0015526779
                        },
                        {
                            "name": "bot_challenge",
                            "confidence": 0.0014684227
                        }
                    ],
                    "text": "Hello!"
                },
                "input_channel": null,
                "message_id": "cefc86543bbc410fa6197f67618bccbe",
                "metadata": null
            }
        ],
        "latest_input_channel": null,
        "active_form": {},
        "latest_action_name": "action_listen"
    }
}
```

## 3. 执行action

请求地址：`localhost:5005/conversations/0/execute`

请求参数：

```json
{
    "name": "utter_greet"
}
```

响应：

```json
{
    "tracker": {
        "sender_id": "0",
        "slots": {
            "attribute": null,
            "city": null,
            "cuisine": null,
            "hotel": null,
            "knowledge_base_last_object": null,
            "knowledge_base_last_object_type": null,
            "knowledge_base_listed_objects": null,
            "mention": null,
            "object_type": null,
            "restaurant": null
        },
        "latest_message": {
            "intent": {
                "name": "greet",
                "confidence": 0.9873532057
            },
            "entities": [],
            "intent_ranking": [
                {
                    "name": "greet",
                    "confidence": 0.9873532057
                },
                {
                    "name": "goodbye",
                    "confidence": 0.0096256845
                },
                {
                    "name": "query_knowledge_base",
                    "confidence": 0.0015526779
                },
                {
                    "name": "bot_challenge",
                    "confidence": 0.0014684227
                }
            ],
            "text": "Hello!"
        },
        "latest_event_time": 1571647564.262816906,
        "followup_action": null,
        "paused": false,
        "events": [
            {
                "event": "action",
                "timestamp": 1571646907.634185791,
                "name": "action_listen",
                "policy": null,
                "confidence": null
            },
            {
                "event": "user",
                "timestamp": 1571646907.7877626419,
                "text": "Hello!",
                "parse_data": {
                    "intent": {
                        "name": "greet",
                        "confidence": 0.9873532057
                    },
                    "entities": [],
                    "intent_ranking": [
                        {
                            "name": "greet",
                            "confidence": 0.9873532057
                        },
                        {
                            "name": "goodbye",
                            "confidence": 0.0096256845
                        },
                        {
                            "name": "query_knowledge_base",
                            "confidence": 0.0015526779
                        },
                        {
                            "name": "bot_challenge",
                            "confidence": 0.0014684227
                        }
                    ],
                    "text": "Hello!"
                },
                "input_channel": null,
                "message_id": "cefc86543bbc410fa6197f67618bccbe",
                "metadata": null
            },
            {
                "event": "action",
                "timestamp": 1571647564.2628035545,
                "name": "utter_greet",
                "policy": null,
                "confidence": null
            },
            {
                "event": "bot",
                "timestamp": 1571647564.262816906,
                "text": "Hey!",
                "data": {
                    "elements": null,
                    "quick_replies": null,
                    "buttons": null,
                    "attachment": null,
                    "image": null,
                    "custom": null
                },
                "metadata": {}
            }
        ],
        "latest_input_channel": null,
        "active_form": {},
        "latest_action_name": "utter_greet"
    },
    "messages": [
        {
            "recipient_id": "0",
            "text": "Hey!"
        }
    ]
}
```

返回的结果为`Hey!`