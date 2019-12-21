+++
date="2019-12-21"
title="对话系统rasa示例简析 - concertbot"
categories=["chatbot","rasa"]
tags=["rasa-example"]
+++

# 对话系统rasa示例简析 - concertbot

## 链接

https://github.com/RasaHQ/rasa/tree/master/examples/concertbot

## 示例说明

此处不赘述，参见：https://sourcegraph.com/github.com/RasaHQ/rasa/-/blob/examples/concertbot/README.md。

## domain描述

```
slots:      <!-- 定义了两个slot，concerts和venues，他们的类型都是list -->
  concerts:
    type: list
  venues:
    type: list

intents:   <!-- 定义意图 -->
 - greet
 - thankyou
 - goodbye
 - search_concerts
 - search_venues
 - compare_reviews
 - bot_challenge

entities:  <!-- 定义实体 -->
 - name

templates:                       <!-- 对话模板，用于简单的对话选择 -->
  utter_greet:
    - text: "hey there!"
  utter_goodbye:
    - text: "goodbye :("
  utter_default:
    - text: "default message"
  utter_youarewelcome:
    - text: "you're very welcome"
  utter_iamabot:
    - text: "I am a bot, powered by Rasa."

actions:                        <!-- 动作名称 -->
  - utter_default
  - utter_greet
  - utter_goodbye
  - utter_youarewelcome
  - action_search_concerts
  - action_search_venues
  - action_show_concert_reviews
  - action_show_venue_reviews
  - utter_iamabot
```

## 故事描述

故事用来设定意图和响应是怎么对应起来的。故事内容，以及对应的注释如下：

```
## greet <!-- 常规的故事描述，一个意图对应一个对话响应， ## 开头的是描述，可以是任何内容 -->
* greet  <!-- *开头的是意图 -->
    - utter_greet <!-- -开头的是action -->

## happy
* thankyou
    - utter_youarewelcome

## goodbye
* goodbye
    - utter_goodbye

## venue_search
* search_venues
    - action_search_venues <!-- 自定义的action -->
    - slot{"venues": [{"name": "Big Arena", "reviews": 4.5}]} <!-- slot对应的默认值 -->

## concert_search
* search_concerts
    - action_search_concerts
    - slot{"concerts": [{"artist": "Foo Fighters", "reviews": 4.5}]}

## compare_reviews_venues
* search_venues
    - action_search_venues
    - slot{"venues": [{"name": "Big Arena", "reviews": 4.5}]}
* compare_reviews <!-- search_venues， compare_reviews顺序执行时，会匹配到下面定义的action -->
    - action_show_venue_reviews

## compare_reviews_concerts
* search_concerts
    - action_search_concerts
    - slot{"concerts": [{"artist": "Foo Fighters", "reviews": 4.5}]}
* compare_reviews
    - action_show_concert_reviews

## bot challenge
* bot_challenge
  - utter_iamabot
```

`rasa visualize`运行结果如下：

![](E:/Workspace/data_science/TaskTracker/xiewei.zhong/images/concertbot.png)

## Action描述

actions.py中给出了自定义的action，可以参见：[对话系统Rasa - Actions 翻译](https://zhuanlan.zhihu.com/p/83600363)

```python
from rasa_sdk import Action
from rasa_sdk.events import SlotSet

# 继承Action实现自定义action，通常需要实现两个函数，
# name，返回的是该action的名字，需要和stories.md和domain.yml中保持一致，
# run，具体实现的代码
class ActionSearchConcerts(Action):
    def name(self):
        return "action_search_concerts"

    def run(self, dispatcher, tracker, domain):
        concerts = [
            {"artist": "Foo Fighters", "reviews": 4.5},
            {"artist": "Katy Perry", "reviews": 5.0},
        ]
        description = ", ".join([c["artist"] for c in concerts])
        dispatcher.utter_message("{}".format(description)) # 返回对话内容
        return [SlotSet("concerts", concerts)] # 将值设定到slot中


class ActionSearchVenues(Action):
    def name(self):
        return "action_search_venues"

    def run(self, dispatcher, tracker, domain):
        venues = [
            {"name": "Big Arena", "reviews": 4.5},
            {"name": "Rock Cellar", "reviews": 5.0},
        ]
        dispatcher.utter_message("here are some venues I found")
        description = ", ".join([c["name"] for c in venues])
        dispatcher.utter_message("{}".format(description))
        return [SlotSet("venues", venues)]


class ActionShowConcertReviews(Action):
    def name(self):
        return "action_show_concert_reviews"

    def run(self, dispatcher, tracker, domain):
        concerts = tracker.get_slot("concerts") # 从tracker中，根据slot名字，获取slot的值
        dispatcher.utter_message("concerts from slots: {}".format(concerts))
        return []


class ActionShowVenueReviews(Action):
    def name(self):
        return "action_show_venue_reviews"

    def run(self, dispatcher, tracker, domain):
        venues = tracker.get_slot("venues")
        dispatcher.utter_message("venues from slots: {}".format(venues))
        return []
```

## endpoints描述

```
action_endpoint: <!-- 自定义action 的通信通道 -->
  url: http://localhost:5055/webhook
```

## config描述

关于pipeline可以参见：[对话系统Rasa - choosing a pipeline 翻译](https://zhuanlan.zhihu.com/p/83753179)。简单描述pipeline定义了自然语言理解过程中的各个步骤，以`supervised_embeddings`为例，其先后包括了：分词（WhitespaceTokenizer），正则化特征化（RegexFeaturizer），实体提取（CRFEntityExtractor），实体同义词映射（EntitySynonymMapper），词袋特征化（CountVectorsFeaturizer），意图分类（EmbeddingIntentClassifier）。

通过pipeline我们可以将输入转换成意图，那么针对意图如何选择对应的action呢，policies定义了行为选择的策略。其中，

- KerasPolicy用Keras中实现的神经网络来选择下一个action；
- FallbackPolicy，当满足以下条件中的其中一个时候会触发回退，1. 意图识别的confidence的值低于`nlu_threshold`，2. 识别出来的意图中，confidence最高的和第二高的之间的差距小于`ambiguity_threshold`，3. 没有一个对话策略预测的行为的confidence值高于`core_threshold`；
- MemoizationPolicy用来记忆你的训练数据中的对话，当在训练数据中能够找到对应的对话，那么confidence为1，否则confidence为0；
- MappingPolicy：能够将意图映射到action。

```
language: en

pipeline: supervised_embeddings

policies:
  - name: KerasPolicy
    epochs: 200
    batch_size: 50
    max_training_samples: 300
  - name: FallbackPolicy
  - name: MemoizationPolicy
  - name: MappingPolicy
```

## 对话记录

![](E:/Workspace/data_science/TaskTracker/xiewei.zhong/images/concertbot_chattest.png)

需要注意的是这个例子中没有提供NLU数据，因此在对话中要直接输入意图。而没有合适结果的对话，都会被导向default message回复。

## 