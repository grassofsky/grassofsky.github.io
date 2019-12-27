+++
date="2019-12-26"
title="rasa problem - 意图中的use_entities选项"
categories=["chatbot","rasa"]
tags=["rasa-problem"]

+++

# rasa problem - 意图中的use_entities选项

## problem描述

### 准备工作

针对简单的天气查询demo，仅需要提供地点时间。

nlu.md如下：

```
## intent:search_weather
- [今天](date)[上海](location)的天气怎么样
- [北京](location)[明天](date)的天气怎么样
- [杭州](location)的天气
- 查询[北京](location)的天气
- [上海](location)的天气
- 查询天气
- 查询[明天](date)的天气
- 查询[今天](date)的天气
- [北京](location)的天气
- [宁波](location)的天气

## intent:greet
- 你好
- 您好
```

story.md如下：

```
## greet
* greet
	— utter_greet
	
## happy path
* search_weather
	- weather_form
	- form{"name": "weather_form"}
	- form{"name": null}
```

domain.yml如下：

```
intents:
- search_weather
- greet

entities:
- date
- location

slots:
  date:
    type: unfeaturized
  location:
    type: unfeaturized

forms:
- weather_form

actions:
- utter_greet

templates:
  utter_greet:
  - text: "您好"
  utter_ask_location:
  - text: "你要查哪个城市的天气"
  utter_ask_date:
  - text: "你要查哪天的天气"
```

config.yml如下：

```
language: zh

pipeline:
- name: JiebaTokenizer
- name: CRFEntityExtractor
- name: CountVectorsFeaturizer
- name: EmbeddingIntentClassifier

policies:
- name: FormPolicy
- name: MemoizationPolicy
- name: KerasPolicy
- name: MappingPolicy
```

actions.py如下：

```python
from typing import Any, Text, Dict, List

from rasa_sdk.forms import FormAction


class WeatherForm(FormAction):

    def name(self) -> Text:
        return "weather_form"

    @staticmethod
    def required_slots(tracker):
        return ["date", "location"]

    def slot_mapping(self):
        return {
            "date": self.from_entity(entity="date"),
            "location": self.from_entity(entity="location")
        }

    def submit(self, dispatcher, tracker, domain):
        date = tracker.get_slot('date')
        location = tracker.get_slot('location')

        result = "你需要查询{}{}的天气吗？".format(location, date)
        dispatcher.utter_message(result)
        return []

```

训练和执行：

```
rasa train
rasa shell
```

启动actions：

```
rasa run actions
```

试运行如下：

![](/image/rasa_use_entities_1.png)

### 问题

上述运行示例中为什么直接输入“查询北京的天气”没有如期执行对话流程呢？

## 问题排查

先试用debug模式查看情况：`rasa shell --debug`

![](/image/rasa_use_entities_2.png)

从日志中可以看出，意图识别没有问题，entity识别没有问题，出问题的地方是预测下一个action的时候出错了。预测action的策略有：MemoizationPolicy和KerasPolicy，对于小数据量的情况KerasPolicy预测的置信度会非常低，那么出问题的地方就是MemoizationPolicy没有正常运作。那么为什么呢？

翻找一下MemoizationPolicy源代码，位于，RASAPATH/rasa/core/policies/memoization.py。

`MemoizationPolicy->predict_action_probabilities->recall->_recall_states->self.lookup.get`。

通过一步一步的跟踪，发现最有可能出现问题的地方是在查找表中没有找到对应的action，导致的。为了进一步确定问题。需要将查找表进行输出。lookup是在训练的时候构建的。通过对ENABLE_FEATURE_STRING_COMPRESSION参数的设置，来设定是否需要对key进行字符串压缩。为了更便捷的查看lookup，暂时将该变量设置为false，同时，将lookup输出。在`MemoizationPolicy::train`函数中添加如下代码：

```python
def train(
	self,
	training_trackers: List[DialogueStateTracker],
	domain: Domain,
	**kwargs: Any,
) -> None:
    .....
    logger.debug("Current lookup is: {}".format(self.lookup))
    logger.debug("Memorized {} unique examples.".format(len(self.lookup)))
```

然后删除model下面的模型，进行重新训练`rasa train --debug`。可以看到对应的输出（对格式进行了手动调整）：

```
Current lookup is: {
'[null, null, null, null, {}]': 0, 
'[null, null, null, {}, {intent_search_weather: 1.0, prev_action_listen: 1.0}]': 9, '[null, null, {}, {intent_search_weather: 1.0, prev_action_listen: 1.0}, {intent_search_weather: 1.0, prev_weather_form: 1.0}]': 0, 
'[null, null, null, {}, {intent_greet: 1.0, prev_action_listen: 1.0}]': 8, 
'[null, null, {}, {intent_greet: 1.0, prev_action_listen: 1.0}, 
{intent_greet: 1.0, prev_utter_greet: 1.0}]': 0
}
```

接着输出查找的key，需要添加如下代码：

```python
def _recall_states(self, states: List[Dict[Text, float]]) -> Optional[int]:
    logger.debug("Search feature is: {}".format(self._create_feature_key(states)))
    return self.lookup.get(self._create_feature_key(states))
```

再执行`rasa shell --debug`。得到对应的输出为：

```
 Search feature is: [null, null, null, {}, {entity_location: 1.0, intent_search_weather: 1.0, prev_action_listen: 1.0}]
```

和之前的lookup对比，发现多出来一个entity_location，那么怎么可以将entity_location排除呢？

## 解决问题

此时需要使用use_entities。将domain改为如下：

```
intents:
- search_weather:
	use_entities: []
- greet

entities:
- date
- location

slots:
  date:
    type: unfeaturized
  location:
    type: unfeaturized

forms:
- weather_form

actions:
- utter_greet

templates:
  utter_greet:
  - text: "您好"
  utter_ask_location:
  - text: "你要查哪个城市的天气"
  utter_ask_date:
  - text: "你要查哪天的天气"
```

此时的运行结果如下：

![](/image/rasa_use_entities_3.png)