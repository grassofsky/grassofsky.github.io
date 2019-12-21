+++
date="2019-12-21"
title="对话系统rasa示例简析 - formbot"
categories=["chatbot","rasa"]
tags=["rasa-example"]
+++

# 对话系统rasa示例简析 - formbot

[TOC]

## 链接

https://github.com/RasaHQ/rasa/tree/master/examples/formbot

## 示例说明

详细见：https://github.com/RasaHQ/rasa/blob/master/examples/formbot/README.md

## domain描述

关于domain，可以参见：[对话系统Rasa - 领域 翻译](https://zhuanlan.zhihu.com/p/83134588)

`restaurant_form`是`form action`的名字，在该`action`定义的时候，通常需要定义三个方法：`name`，`required_slots`，`submit`。关于forms，可以参见：[对话系统rasa - forms (翻译)](https://zhuanlan.zhihu.com/p/84441651)

```
intents:
  - request_restaurant:
      use_entities: []          <!-- 忽略该意图消息对应的实体 -->
  - chitchat:
      use_entities: []
  - inform
  - affirm
  - deny
  - stop
  - thankyou
  - greet
  - bot_challenge

entities:
  - cuisine       <!-- 风味 -->
  - num_people    <!-- 人数 -->
  - number        <!-- 数量 -->
  - feedback      <!-- 反馈 -->
  - seating       <!-- 座位 -->

slots:
  cuisine:
    type: unfeaturized    <!-- 不影响对话流程 -->
    auto_fill: false      <!-- 停用自动赋值的功能 -->
  num_people:
    type: unfeaturized
    auto_fill: false
  outdoor_seating:
    type: unfeaturized
    auto_fill: false
  preferences:
    type: unfeaturized
    auto_fill: false
  feedback:
    type: unfeaturized
    auto_fill: false
  requested_slot:
    type: unfeaturized

templates:
  utter_ask_cuisine:
    - text: "what cuisine?"
  utter_ask_num_people:
    - text: "how many people?"
  utter_ask_outdoor_seating:
    - text: "do you want to seat outside?"
  utter_ask_preferences:
    - text: "please provide additional preferences"
  utter_ask_feedback:
    - text: "please give your feedback on your experience so far"
  utter_submit:
    - text: "All done!"
  utter_slots_values:
    - text: "I am going to run a restaurant search using the following parameters:\n
             - cuisine: {cuisine}\n     <!-- 会自动使用slot中的cuisine变量值进行填充 -->
             - num_people: {num_people}\n
             - outdoor_seating: {outdoor_seating}\n
             - preferences: {preferences}\n
             - feedback: {feedback}"
  utter_noworries:
    - text: "you are welcome :)"
  utter_chitchat:
    - text: "chitchat"
  utter_ask_continue:
    - text: "do you want to continue?"
  utter_wrong_cuisine:
    - text: "cuisine type is not in the database, please try again"
  utter_wrong_num_people:
    - text: "number of people should be a positive integer, please try again"
  utter_wrong_outdoor_seating:
    - text: "could not convert input to boolean value, please try again"
  utter_default:
    - text: "sorry, I didn't understand you, please try input something else"
  utter_greet:
    - text: "Hello! I am restaurant search assistant! How can I help?"
  utter_iamabot:
    - text: "I am a bot, powered by Rasa."

actions:
  - utter_slots_values
  - utter_noworries
  - utter_chitchat
  - utter_ask_continue
  - utter_greet
  
  - utter_iamabot

forms:
  - restaurant_form
```

## nlu描述

参见[对话系统Rasa-1-训练数据格式 翻译](https://zhuanlan.zhihu.com/p/83023959)。

关于`intent:request_restaurant`可以做出如下分类：

- 直接询问restaurant，如`im looking for a restaurant` ，`i need to find a restaurant`
- 根据口味进行查找，如`can i get [swedish](cuisine) food in any area`，`a restaurant that serves [caribbean](cuisine) food`
- 根据座位数量查找，如`i need a table for [4](num_people)`，`can you please book a table for [5](num_people)?`
- 座位和口味相结合，如`Can I get a table for [four](num_people:4) at the place which server [greek](cuisine) food?`

关于`intent:inform`同样主要针对口味和人数。

```
## intent:greet
- Hi
<!-- 一系列的问候语句，省略 -->

## intent:request_restaurant <!-- 为了更明显的查看，对内容进行了排序 -->
- im looking for a restaurant <!-- 直接询问restaurant -->
- id like a restaurant
- i need to find a restaurant
- restaurant please
- can i get [swedish](cuisine) food in any area       <!-- 根据风味进行查找 -->
- a restaurant that serves [caribbean](cuisine) food
- im looking for a restaurant that serves [mediterranean](cuisine) food
- can i find a restaurant that serves [chinese](cuisine)
- i am looking for any place that serves [indonesian](cuisine) food for [three](num_people:3)
- uh im looking for a restaurant that serves [kosher](cuisine) food
- uh can i find a restaurant and it should serve [brazilian](cuisine) food
- im looking for a restaurant serving [italian](cuisine) food
- i'd like to book a table for [two](num_people:2) with [spanish](cuisine) cuisine
- i need a table for [4](num_people)
- book me a table for [three](num_people:3) at the [italian](cuisine) restaurant
- can you please book a table for [5](num_people)?
- I would like to book a table for [2](num_people)
- looking for a table at the [mexican](cuisine) restaurant for [five](num_people:5)
- find me a table for [7](num_people) people
- Can I get a table for [four](num_people:4) at the place which server [greek](cuisine) food?

## intent:affirm
- yeah a cheap restaurant serving international food
<!-- 一系列的确认语句，省略 -->

## intent:deny
- no
<!-- 一系列的拒绝语句，省略 -->

## intent:inform
- [afghan](cuisine) food
- how bout [asian oriental](cuisine)
- what about [indian](cuisine) food
- uh how about [turkish](cuisine) type of food
- um [english](cuisine)
- im looking for [tuscan](cuisine) food
- id like [moroccan](cuisine) food
- [seafood](cuisine)
- [french](cuisine) food
- serves [british](cuisine) food
- id like [canapes](cuisine)
- serving [jamaican](cuisine) food
- um what about [italian](cuisine) food
- im looking for [corsica](cuisine) food
- im looking for [world](cuisine) food
-  serves [french](cuisine) food
- how about [indian](cuisine) food
- can i get [chinese](cuisine) food
- [irish](cuisine) food
- [english](cuisine) food
- [spanish](cuisine) food
- how bout one that serves [portuguese](cuisine) food and is cheap
- [german](cuisine)
- [korean](cuisine) food
- im looking for [romanian](cuisine) food
-  serves [canapes](cuisine) food
- [gastropub](cuisine)
- i want [french](cuisine) food
- how about [modern european](cuisine) type of food
- it should serve [scandinavian](cuisine) food
- how [european](cuisine)
- how about [european](cuisine) food
- serves [traditional](cuisine) food
- [indonesian](cuisine) food
- [modern european](cuisine)
- serves [brazilian](cuisine)
- i would like [modern european](cuisine) food
- looking for [lebanese](cuisine) food
- [portuguese](cuisine)
- [european](cuisine)
- i want [polish](cuisine) food
- id like [thai](cuisine)
- i want to find [moroccan](cuisine) food
- [afghan](cuisine)
- [scottish](cuisine) food
- how about [vietnamese](cuisine)
- hi im looking for [mexican](cuisine) food
- how about [indian](cuisine) type of food
- [polynesian](cuisine) food
- [mexican](cuisine)
- instead could it be for [four](num_people:4) people
- any [japanese](cuisine) food
- what about [thai](cuisine) food
- how about [asian oriental](cuisine) food
- im looking for [japanese](cuisine) food
- im looking for [belgian](cuisine) food
- im looking for [turkish](cuisine) food
- serving [corsica](cuisine) food
- serving [gastro pub](cuisine:gastropub)
- is there [british](cuisine) food
- [world](cuisine) food
- im looking for something serves [japanese](cuisine) food
- id like a [greek](cuisine)
- im looking for [malaysian](cuisine) food
- i want to find [world](cuisine) food
- serves [pan asian](cuisine:asian) food
- looking for [afghan](cuisine) food
- that serves [portuguese](cuisine) food
- [asian oriental](cuisine:asian) food
- [russian](cuisine) food
- [corsica](cuisine)
- [asian oriental](cuisine:asian)
- serving [basque](cuisine) food
- how about [italian](cuisine)
- looking for [spanish](cuisine) food in the center of town
- it should serve [gastropub](cuisine) food
- [welsh](cuisine) food
- i want [vegetarian](cuisine) food
- im looking for [swedish](cuisine) food
- um how about [chinese](cuisine) food
- [world](cuisine) food
- can i have a [seafood](cuisine) please
- how about [italian](cuisine) food
- how about [korean](cuisine)
- [corsica](cuisine) food
- [scandinavian](cuisine)
- [vegetarian](cuisine) food
- what about [italian](cuisine)
- how about [portuguese](cuisine) food
- serving [french](cuisine) food
- [tuscan](cuisine) food
- how about uh [gastropub](cuisine)
- im looking for [creative](cuisine) food
- im looking for [malaysian](cuisine) food
- im looking for [unusual](cuisine) food
- [danish](cuisine) food
- how about [spanish](cuisine) food
- im looking for [vietnamese](cuisine) food
- [spanish](cuisine)
- a restaurant serving [romanian](cuisine) food
- im looking for [lebanese](cuisine) food
- [italian](cuisine) food
- a restaurant with [afghan](cuisine) food
- im looking for [traditional](cuisine) food
- uh i want [cantonese](cuisine) food
- im looking for [thai](cuisine)
- i want to seat [outside](seating)
- i want to seat [inside](seating)
- i want to seat [outdoor](seating)
- i want to seat [indoor](seating)
- let's go [inside](seating)
- [inside](seating)
- [outdoor](seating)
- prefer sitting [indoors](seating)
- I would like to seat [inside](seating) please
- I prefer sitting [outside](seating)
- my feedback is [good](feedback)
- my feedback is [great](feedback)
- it was [terrible](feedback)
- i consider it [success](feedback)
- you are [awful](feedback)
- for [ten](num_people:10) people
- [2](num_people) people
- for [three](num_people:3) people
- just [one](num_people:1) person
- book for [seven](num_people:7) people
- 2[num_people] please
- [nine](num_people:9) people

## intent:thankyou
- um thank you good bye
<!-- 一系列的感谢语句，省略 -->

## intent:chitchat
- can you share your boss with me?
<!-- 一系列的闲聊语句，省略 -->

## intent:stop
- ok then you cant help me
<!-- 一系列的stop语句，省略 -->

## intent:bot_challenge
- are you a bot?
<!-- 省略 -->
```

## stories描述

故事集合包含以下故事：

- happy path，常规的路径，先问候，然后询问restaurant，通过form获取需要的slot值，确认查询结果
- unhappy path，针对出现一次闲聊的处理路径
- very unhappy path，针对出现多次闲聊的处理路径
- stop but continue path，出现中断意图，然后又继续的路径
- stop and really stop path，出现中断意图，真的中断的路径
- chitchat stop but continue path，闲聊后出现中断意图，然后继续
- stop but continue and chitchat path，中断意图出现后有继续，但是出现闲聊
- chitchat stop but continue and chitchat path，闲聊中断后继续，但是出现闲聊
- chitchat, stop and really stop path，闲聊后中断然后确实中断的路径
- Generated Story 3490283781720101690 (example from interactive learning, "form: " will be excluded from training)；通过 interactive learning创建的故事
- bot challenge

```
## happy path
* greet
    - utter_greet
* request_restaurant
    - restaurant_form
    - form{"name": "restaurant_form"}   <!-- 启用form -->
    - form{"name": null}                <!-- 停用form -->
    - utter_slots_values                <!-- 输出获取的slot值 -->
* thankyou
    - utter_noworries

## unhappy path
* greet
    - utter_greet
* request_restaurant
    - restaurant_form
    - form{"name": "restaurant_form"}
* chitchat
    - utter_chitchat      <!-- 针对闲聊回复 -->
    - restaurant_form
    - form{"name": null}  <!-- 停用form -->
    - utter_slots_values  <!-- 输出当前获取的slot值 -->
* thankyou
    - utter_noworries

## very unhappy path
* greet
    - utter_greet
* request_restaurant
    - restaurant_form
    - form{"name": "restaurant_form"}
* chitchat  <!-- 针对多次闲聊的现象 -->
    - utter_chitchat
    - restaurant_form
* chitchat
    - utter_chitchat
    - restaurant_form
* chitchat
    - utter_chitchat
    - restaurant_form
    - form{"name": null}
    - utter_slots_values
* thankyou
    - utter_noworries

## stop but continue path
* greet
    - utter_greet
* request_restaurant
    - restaurant_form
    - form{"name": "restaurant_form"}
* stop
    - utter_ask_continue
* affirm
    - restaurant_form
    - form{"name": null}
    - utter_slots_values
* thankyou
    - utter_noworries

## stop and really stop path
* greet
    - utter_greet
* request_restaurant
    - restaurant_form
    - form{"name": "restaurant_form"}
* stop
    - utter_ask_continue
* deny
    - action_deactivate_form
    - form{"name": null}

## chitchat stop but continue path
* request_restaurant
    - restaurant_form
    - form{"name": "restaurant_form"}
* chitchat
    - utter_chitchat
    - restaurant_form
* stop
    - utter_ask_continue
* affirm
    - restaurant_form
    - form{"name": null}
    - utter_slots_values
* thankyou
    - utter_noworries

## stop but continue and chitchat path
* greet
    - utter_greet
* request_restaurant
    - restaurant_form
    - form{"name": "restaurant_form"}
* stop
    - utter_ask_continue
* affirm
    - restaurant_form
* chitchat
    - utter_chitchat
    - restaurant_form
    - form{"name": null}
    - utter_slots_values
* thankyou
    - utter_noworries

## chitchat stop but continue and chitchat path
* greet
    - utter_greet
* request_restaurant
    - restaurant_form
    - form{"name": "restaurant_form"}
* chitchat
    - utter_chitchat
    - restaurant_form
* stop
    - utter_ask_continue
* affirm
    - restaurant_form
* chitchat
    - utter_chitchat
    - restaurant_form
    - form{"name": null}
    - utter_slots_values
* thankyou
    - utter_noworries

## chitchat, stop and really stop path
* greet
    - utter_greet
* request_restaurant
    - restaurant_form
    - form{"name": "restaurant_form"}
* chitchat
    - utter_chitchat
    - restaurant_form
* stop
    - utter_ask_continue
* deny
    - action_deactivate_form
    - form{"name": null}

## Generated Story 3490283781720101690 (example from interactive learning, "form: " will be excluded from training)
* greet
    - utter_greet
* request_restaurant
    - restaurant_form
    - form{"name": "restaurant_form"}
    - slot{"requested_slot": "cuisine"}
* chitchat
    - utter_chitchat  <!-- restaurant_form was predicted by FormPolicy and rejected, other policy predicted utter_chitchat -->
    - restaurant_form
    - slot{"requested_slot": "cuisine"}
* form: inform{"cuisine": "mexican"}
    - slot{"cuisine": "mexican"}
    - form: restaurant_form
    - slot{"cuisine": "mexican"}
    - slot{"requested_slot": "num_people"}
* form: inform{"number": "2"}
    - form: restaurant_form
    - slot{"num_people": "2"}
    - slot{"requested_slot": "outdoor_seating"}
* chitchat
    - utter_chitchat
    - restaurant_form
    - slot{"requested_slot": "outdoor_seating"}
* stop
    - utter_ask_continue
* affirm
    - restaurant_form  <!-- FormPolicy predicted FormValidation(False), other policy predicted restaurant_form -->
    - slot{"requested_slot": "outdoor_seating"}
* form: affirm
    - form: restaurant_form
    - slot{"outdoor_seating": true}
    - slot{"requested_slot": "preferences"}
* form: inform
    - form: restaurant_form
    - slot{"preferences": "/inform"}
    - slot{"requested_slot": "feedback"}
* form: inform{"feedback": "great"}
    - slot{"feedback": "great"}
    - form: restaurant_form
    - slot{"feedback": "great"}
    - form{"name": null}
    - slot{"requested_slot": null}
    - utter_slots_values
* thankyou
    - utter_noworries

## bot challenge
* bot_challenge
  - utter_iamabot
```

`rasa visualize`结果如下：

![](./image/Rasa-Core-Visualisation.jpg)

## action描述

```python
# -*- coding: utf-8 -*-
from typing import Dict, Text, Any, List, Union, Optional

from rasa_sdk import Tracker
from rasa_sdk.executor import CollectingDispatcher
from rasa_sdk.forms import FormAction


class RestaurantForm(FormAction):
    """Example of a custom form action"""

    def name(self) -> Text:
        """Unique identifier of the form"""
        
        return "restaurant_form"

    @staticmethod
    def required_slots(tracker: Tracker) -> List[Text]:
        """A list of required slots that the form has to fill"""
        # 和这个form相关的slot值

        return ["cuisine", "num_people", "outdoor_seating", "preferences", "feedback"]

    def slot_mappings(self) -> Dict[Text, Union[Dict, List[Dict]]]:
        """A dictionary to map required slots to
            - an extracted entity
            - intent: value pairs
            - a whole message
            or a list of them, where a first match will be picked"""

        # cuisine: 不是chitchat意图的消息中的cuisine实体
        # num_people: inform或request_restaurant意图消息中的num_people实体，其他消息中的nunber实体
        #             但在这个例子中并没有number实体
        # outdoor_seating: 如果针对问题返回affirm的意图，那么值为True
        #                  如果针对问题返回deny的意图，那么值为False
        #                  提取到的seating值。
        # preferences: 如果是拒绝意图，那么值是no additional preferences
        #              否则从非affirm意图消息中获取文本
        # feedback: feedback实体，或空
        return {
            "cuisine": self.from_entity(entity="cuisine", not_intent="chitchat"),
            "num_people": [
                self.from_entity(
                    entity="num_people", intent=["inform", "request_restaurant"]
                ),
                self.from_entity(entity="number"),
            ],
            "outdoor_seating": [
                self.from_entity(entity="seating"),
                self.from_intent(intent="affirm", value=True),
                self.from_intent(intent="deny", value=False),
            ],
            "preferences": [
                self.from_intent(intent="deny", value="no additional preferences"),
                self.from_text(not_intent="affirm"),
            ],
            "feedback": [self.from_entity(entity="feedback"), self.from_text()],
        }

    # USED FOR DOCS: do not rename without updating in docs
    @staticmethod
    def cuisine_db() -> List[Text]:
        """Database of supported cuisines"""

        return [
            "caribbean",
            "chinese",
            "french",
            "greek",
            "indian",
            "italian",
            "mexican",
        ]

    @staticmethod
    def is_int(string: Text) -> bool:
        """Check if a string is an integer"""

        try:
            int(string)
            return True
        except ValueError:
            return False

    # USED FOR DOCS: do not rename without updating in docs
    def validate_cuisine(
        self,
        value: Text,
        dispatcher: CollectingDispatcher,
        tracker: Tracker,
        domain: Dict[Text, Any],
    ) -> Dict[Text, Any]:
        """Validate cuisine value."""

        if value.lower() in self.cuisine_db():
            # validation succeeded, set the value of the "cuisine" slot to value
            return {"cuisine": value}
        else:
            dispatcher.utter_template("utter_wrong_cuisine", tracker)
            # validation failed, set this slot to None, meaning the
            # user will be asked for the slot again
            return {"cuisine": None}

    def validate_num_people(
        self,
        value: Text,
        dispatcher: CollectingDispatcher,
        tracker: Tracker,
        domain: Dict[Text, Any],
    ) -> Dict[Text, Any]:
        """Validate num_people value."""

        if self.is_int(value) and int(value) > 0:
            return {"num_people": value}
        else:
            dispatcher.utter_template("utter_wrong_num_people", tracker)
            # validation failed, set slot to None
            return {"num_people": None}

    def validate_outdoor_seating(
        self,
        value: Text,
        dispatcher: CollectingDispatcher,
        tracker: Tracker,
        domain: Dict[Text, Any],
    ) -> Dict[Text, Any]:
        """Validate outdoor_seating value."""

        if isinstance(value, str):
            if "out" in value:
                # convert "out..." to True
                return {"outdoor_seating": True}
            elif "in" in value:
                # convert "in..." to False
                return {"outdoor_seating": False}
            else:
                dispatcher.utter_template("utter_wrong_outdoor_seating", tracker)
                # validation failed, set slot to None
                return {"outdoor_seating": None}

        else:
            # affirm/deny was picked up as T/F
            return {"outdoor_seating": value}

    def submit(
        self,
        dispatcher: CollectingDispatcher,
        tracker: Tracker,
        domain: Dict[Text, Any],
    ) -> List[Dict]:
        """Define what the form has to do
            after all required slots are filled"""

        # utter submit template
        dispatcher.utter_template("utter_submit", tracker)
        return []
```

## endpoints描述

```
action_endpoint:
    url: http://localhost:5055/webhook
```

## config描述

关于pipeline各个组件的介绍可以参见：https://zhuanlan.zhihu.com/p/83566179

```
language: en

pipeline:
  - name: WhitespaceTokenizer
  - name: CRFEntityExtractor
  - name: EntitySynonymMapper
  - name: CountVectorsFeaturizer
    token_pattern: (?u)\b\w+\b
  - name: EmbeddingIntentClassifier
  - name: DucklingHTTPExtractor
    url: http://localhost:8000
    dimensions:
      - number

policies:
  - name: FallbackPolicy
  - name: MemoizationPolicy
  - name: FormPolicy
  - name: MappingPolicy
```

## 对话记录

![](./image/formbot_chat.png)

