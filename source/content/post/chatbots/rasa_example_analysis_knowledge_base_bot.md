+++
date="2019-12-21"
title="对话系统rasa示例简析 - Knowledge Base Bot"
categories=["chatbot","rasa"]
tags=["rasa-example"]
+++

# 对话系统rasa示例简析 - Knowledge Base Bot

## 链接

https://github.com/RasaHQ/rasa/tree/master/examples/knowledgebasebot

## 示例说明

此处不赘述，参见：https://github.com/RasaHQ/rasa/tree/master/examples/knowledgebasebot/README.md

## domain描述

考虑到篇幅长度，对部分内容进行了省略。关于slot可以参见：[对话系统rasa - slots 翻译](https://zhuanlan.zhihu.com/p/83586780)

```
intents:           <!-- 定义意图 -->
  - greet
  - goodbye
  <!-- 省略 -->

entities:         <!-- 定义实体 -->
  - object_type
  - mention
  <!-- 省略 -->

slots:           <!-- unfeaturized类型的slot，仅用来存储信息，对对话工作流没有影响 -->   
  object_type:
    type: unfeaturized
  mention:
    type: unfeaturized
  <!-- 省略 -->

actions:
- utter_greet
- utter_goodbye
- utter_ask_rephrase
- utter_iamabot
- action_query_knowledge_base    <!-- 自定义的action -->

templates:
  utter_greet:   <!-- 返回的对话，提供多选 -->
  - text: "Hey!"
  - text: "Hello! How can I help you?"

  utter_goodbye:
  - text: "Bye"
  - text: "Goodbye. See you soon."

  <!-- 省略 -->
```

## nlu描述

考虑到篇幅长度，对部分内容进行了省略。训练数据格式可以参见：[对话系统Rasa-1-训练数据格式 翻译](https://zhuanlan.zhihu.com/p/83023959)。

这里简单针对实体进行简单介绍：`what [restaurants](object_type:restaurant) can you recommend?`这里的格式是：`[实体](实体名称：具体值)`，如果没有具体值的话，这个值就是实体本身 。

```
## intent:greet <!-- great意图相关的语句 -->
- hey
- hello
<!-- 省略 -->

## intent:goodbye  <!-- goodbye意图相关的语句 -->
- bye
- goodbye
<!-- 省略 -->

## intent:query_knowledge_base <!-- 检索意图相关的语句 -->
- what [restaurants](object_type:restaurant) can you recommend?
- list some [restaurants](object_type:restaurant)
- can you name some [restaurants](object_type:restaurant) please?
- can you show me some [restaurant](object_type:restaurant) options
- list [German](cuisine) [restaurants](object_type:restaurant)
- do you have any [mexican](cuisine) [restaurants](object_type:restaurant)?
- do you know the [price range](attribute:price-range) of [that one](mention)?
<!-- 省略 -->

## lookup:restaurant
- Donath
- Berlin Burrito Company
<!-- 省略 -->

## lookup:hotel
- Hilton
- B&B
<!-- 省略 -->

## intent:bot_challenge
- are you a bot?
- are you a human?
<!-- 省略 -->
```

## stories描述

```
## Happy path 1   <!-- 第一个故事包括了三个对话，问候，查询，再见 -->
* greet
  - utter_greet
* query_knowledge_base
  - action_query_knowledge_base
* goodbye
  - utter_goodbye

## Happy path 2  <!-- 第二个故事包括了四个对话，问候，查询，查询，再见 -->
* greet
  - utter_greet
* query_knowledge_base
  - action_query_knowledge_base <!-- 针对复杂的查询，自定义了实现逻辑 -->
* query_knowledge_base
  - action_query_knowledge_base
* goodbye
  - utter_goodbye

## Hello <!-- 第三个故事是简单的问候 -->
* greet
- utter_greet

## Query Knowledge Base  <!-- 第四个故事是简单的查询 -->
* query_knowledge_base
- action_query_knowledge_base

## Bye <!-- 第五个故事是简单的再见 -->
* goodbye
- utter_goodbye

## bot challenge <!-- 第六个故事是对bot的质疑 -->
* bot_challenge
  - utter_iamabot
```

`rasa visualize`运行结果如下：

![](E:/Workspace/data_science/TaskTracker/xiewei.zhong/images/knowledgebasebot_stories.png)

## action描述

```python
from rasa_sdk.knowledge_base.storage import InMemoryKnowledgeBase
from rasa_sdk.knowledge_base.actions import ActionQueryKnowledgeBase


class ActionMyKB(ActionQueryKnowledgeBase):
    def __init__(self):
        # load knowledge base with data from the given file
        knowledge_base = InMemoryKnowledgeBase("knowledge_base_data.json")

        # overwrite the representation function of the hotel object
        # by default the representation function is just the name of the object
        knowledge_base.set_representation_function_of_object(
            "hotel", lambda obj: obj["name"] + " (" + obj["city"] + ")"
        )

        super().__init__(knowledge_base)
```

从定义中可见主要的实现逻辑位于ActionQueryKnowledgeBase中，查看https://github.com/RasaHQ/rasa-sdk/blob/master/rasa_sdk/knowledge_base/actions.py文件内部的run函数，具体如下：

```python
def run(self, dispatcher, tracker, domain):
    # ...
    object_type = tracker.get_slot(SLOT_OBJECT_TYPE) # object_type
    last_object_type = tracker.get_slot(SLOT_LAST_OBJECT_TYPE) # knowledge_base_last_object
    attribute = tracker.get_slot(SLOT_ATTRIBUTE) # attribute

    # 笔者添加，用来观察变量
    print("fun:run, object_type: ", object_type)
    print("fun:run, last_object_type: ", last_object_type)
    print("fun:run, attribtue: ", attribute)
    
    new_request = object_type != last_object_type

    # 如果没有object_type的slot被匹配上，那么直接返回
    if not object_type:
        # object type always needs to be set as this is needed to query the
        # knowledge base
        dispatcher.utter_template("utter_ask_rephrase", tracker)
        return []

    # 如果没有属性，或者是一个新的请求，查询object_type的实体
    if not attribute or new_request:
        return self._query_objects(dispatcher, tracker)
    elif attribute: # 如果属性不为空，需要查找到对应的属性
        return self._query_attribute(dispatcher, tracker)

    dispatcher.utter_template("utter_ask_rephrase", tracker)
    return []
```

该函数主要调用了两个函数`_query_objects`和`_query_attribute`，`_query_objects`如下。

```python
def _query_objects(self, dispatcher, tracker):
    # type: (CollectingDispatcher, Tracker) -> List[Dict]
    """
    Queries the knowledge base for objects of the requested object type and
    outputs those to the user. The objects are filtered by any attribute the
    user mentioned in the request.

    Args:
    dispatcher: the dispatcher
    tracker: the tracker

    Returns: list of slots
    """
    # 如果object_type为：restaurant
    # 那么object_attributes为：
    #   ['id', 'name', 'cuisine', 'outside-seating', 'price-range']
    object_type = tracker.get_slot(SLOT_OBJECT_TYPE)
    object_attributes = self.knowledge_base.get_attributes_of_object(object_type)

    # get all set attribute slots of the object type to be able to filter the
    # list of objects
    # 获取对象对应的属性，比如用户说'What Italian restaurants do you know?'.
    # 那么NER会检测出'Italian'是'cuisine'，我们知道cuisine是restaurant的一个属性。
    # 因此, 这个函数返回 [{'name': 'cuisine', 'value': 'Italian'}] 
    attributes = get_attribute_slots(tracker, object_attributes)
    # query the knowledge base
    # 根据属性获取对应的restaurant，如果属性为空，则从所有的restaurant中查询，默认返回长度为5
    objects = self.knowledge_base.get_objects(object_type, attributes)

    # 针对查找结果构建对话
    self.utter_objects(dispatcher, object_type, objects)

    if not objects:
        # 重置当前对象类型的所有属性slot
        # 如果用户说Show me all restaurants with Italian cuisine
        # NER会识别出restaurant为object_type,Italian为cuisine对象。
        # 那么我们会从知识库中提取出cuisine=Italian的restaurant。
        # 当列出对象的时候，我们check NER检测出了什么属性。我们拿到了所有set
        # 的属性，如cuisine = Italian。如果我们在请求之后没有重置attribute slots，
        # 当用户的下一个对话是List all restaurants that have wifi，我们将
        # 有两个属性slots，wifi和cuisine。因此，我们查找restaurant的时候需要
        # 校验这两个属性。但是用户的表达仅仅针对wifi。
    	return reset_attribute_slots(tracker, object_attributes)

    # 获取主键
    key_attribute = self.knowledge_base.get_key_attribute_of_object(object_type)

    # 针对objects小于等于1的情况下才有last_object
    last_object = None if len(objects) > 1 else objects[0][key_attribute]

    # 笔者添加用来观察变量
    print("fun:_query_objects object_type: ", object_type)
    print("fun:_query_objects object_attributes: ", object_attributes)
    print("fun:_query_objects attributes: ", attributes)
    print("fun:_query_objects objects: ", objects)
    print("fun:_query_objects key_attribute: ", key_attribute)
    print("fun:_query_objects last_object: ", last_object)
    
    # slot赋值
    slots = [
        SlotSet(SLOT_OBJECT_TYPE, object_type),
        SlotSet(SLOT_MENTION, None),
        SlotSet(SLOT_ATTRIBUTE, None),
        SlotSet(SLOT_LAST_OBJECT, last_object),
        SlotSet(SLOT_LAST_OBJECT_TYPE, object_type),
        SlotSet(
            SLOT_LISTED_OBJECTS, list(map(lambda e: e[key_attribute], objects))
        ),
    ]

    return slots + reset_attribute_slots(tracker, object_attributes)    
```

`_query_attribute`如下：

```python
def _query_attribute(self, dispatcher, tracker):
	# type: (CollectingDispatcher, Tracker) -> List[Dict]
	"""
	Queries the knowledge base for the value of the requested attribute of the
	mentioned object and outputs it to the user.

	Args:
		dispatcher: the dispatcher
		tracker: the tracker

	Returns: list of slots
	"""
    # 针对:
    # list restaurants
    # do you know what cuisine the last one has
    # 两个问题，当第二个问题处理的时候，进入到这个函数中：
    # 此时object_type为restaurants
    # attribute为cuisine
	object_type = tracker.get_slot(SLOT_OBJECT_TYPE)
	attribute = tracker.get_slot(SLOT_ATTRIBUTE)

    # 获取用户指代的对象，返回对应的id
	object_name = get_object_name(
		tracker,
		self.knowledge_base.ordinal_mention_mapping,
		self.use_last_object_mention,
	)
    
    # 笔者添加用来观察变量
    print("fun:_query_attribute object_type: ", object_type)
    print("fun:_query_attribute attribute: ", attribute)
    print("fun:_query_attribute object_name: ", object_name)

	if not object_name or not attribute:
		dispatcher.utter_template("utter_ask_rephrase", tracker)
		return [SlotSet(SLOT_MENTION, None)]

    # 根据id和对应的对象类型，获取object
    # {'id':2,'name':'I due forni', 'cuisine': 'Italian', 'outside-seating': True, 'price-range': 'mid-range'}
	object_of_interest = self.knowledge_base.get_object(object_type, object_name)
    print("fun:_query_attribute object_of_interest: ", object_of_interest)

	if not object_of_interest or attribute not in object_of_interest:
		dispatcher.utter_template("utter_ask_rephrase", tracker)
		return [SlotSet(SLOT_MENTION, None)]

    # 获取属性对应的值，Italian
	value = object_of_interest[attribute]
	repr_function = self.knowledge_base.get_representation_function_of_object(
		object_type
	)
    # object_representation = 'I due forni'
	object_representation = repr_function(object_of_interest)
	key_attribute = self.knowledge_base.get_key_attribute_of_object(object_type)
	object_identifier = object_of_interest[key_attribute]
    
    print("fun:_query_attribute value: ", value)
    print("fun:_query_attribute object_representation: ", object_representation)
    print("fun:_query_attribute key_attribute: ", key_attribute)
    print("fun:_query_attribute object_identifier: ", object_identifier)

    # 用来给用户反馈消息
	self.utter_attribute_value(dispatcher, object_representation, attribute, value)

	slots = [
		SlotSet(SLOT_OBJECT_TYPE, object_type),   # 存储object_type实体值
		SlotSet(SLOT_ATTRIBUTE, None),            # 将attribute重置None
		SlotSet(SLOT_MENTION, None),              # 将mention重置为None
		SlotSet(SLOT_LAST_OBJECT, object_identifier), # 上一次object的id
		SlotSet(SLOT_LAST_OBJECT_TYPE, object_type),  # 记录为上一次的object_type实体值
	]

	return slots
```

## endpoints描述

自定义的action 的服务：

```
action_endpoint:
  url: "http://localhost:5055/webhook"
```

## config描述

pipeline使用模板`supervised_embeddings`。

Policy：

- KerasPolicy用Keras中实现的神经网络来选择下一个action；
- MemoizationPolicy用来记忆你的训练数据中的对话，当在训练数据中能够找到对应的对话，那么confidence为1，否则confidence为0；

```
language: en
pipeline: supervised_embeddings

policies:
  - name: MemoizationPolicy
  - name: KerasPolicy
```

## knowledge_base_data.json

酒店和餐馆数据集。

```
{
    "restaurant": [
        {
            "id": 0,
            "name": "Donath",
            "cuisine": "Italian",
            "outside-seating": true,
            "price-range": "mid-range"
        },
        ......
    ],
    "hotel": [
        {
            "id": 0,
            "name": "Hilton",
            "price-range": "expensive",
            "breakfast-included": true,
            "city": "Berlin",
            "free-wifi": true,
            "star-rating": 5,
            "swimming-pool": true
        },
        ......
    ]
}
```

## 对话记录

![](./image/knowledgebasebot_chat.png)

对应的变量输出为：

![](./image/knowledgebasebot_chat_action_log.png)

从上面可以观察到，they和it的指代并没有被识别出来，在获取`object_name`的时候返回了Null。经过代码分析，原因出在如下代码，在对指代的映射中，并没有they和it：

```python
# rasa_sdk/knowledge_base/storage.py

# ...
		self.ordinal_mention_mapping = {
            "1": lambda l: l[0],
            "2": lambda l: l[1],
            "3": lambda l: l[2],
            "4": lambda l: l[3],
            "5": lambda l: l[4],
            "6": lambda l: l[5],
            "7": lambda l: l[6],
            "8": lambda l: l[7],
            "9": lambda l: l[8],
            "10": lambda l: l[9],
            "ANY": lambda l: random.choice(list),
            "LAST": lambda l: l[-1],
        }
# ...
```

