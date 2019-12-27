+++
date="2019-12-21"
title="对话系统rasa - Knowledge Base Actions （翻译）"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统rasa - Knowledge Base Actions （翻译）

## rasa文集

[grassofsky：rasa文章导引（用于收藏）](https://zhuanlan.zhihu.com/p/88112269)

**警告**：这个功能处于试验阶段。我们引入试验的功能，为了从我们的社区中过去反馈，因此我们建议你进行尝试。但是，这个功能可能在未来的版本中被改变或去除。有任何的反馈，请在[forum](https://forum.rasa.com/)进行分享。

基于knowledge的action使得你能够处理类似的对话：

![](/image/knowledge-base-example.png)

对话AI中的一个常见问题是，用户并不是总是通过名字来指向特定的对象，还经常使用如“第一个”或者“它”等指代来表示对象。我们需要记录这些信息，用来获取指代具体指向的内容。

此外，在对话过程中，用户可能想要获取额外的信息，比如，一个餐馆的外面是否有座位，或者价格怎样。为了回答用户的这些问题，需要有餐饮领域的知识。由于这些信息通常是变动的，因此对这些信息进行硬编码是不合适的。

为了解决上面的挑战，Rasa能够集成知识库。为了使用这个集成，你可以在`ActionQueryKnowledgeBase`的基础上创建自定义的action，这是一个预先写好的自定义action，包含用来从知识库检索对象和属性的逻辑。

你可以在[knowledge base bot](https://github.com/RasaHQ/rasa/blob/master/examples/knowledgebasebot/)看到完整的示例。下面介绍实现该自定义的action。

## 使用`ActionQueryKnowledgeBase`

### 创建知识库

用于回答用户问题的数据会被存储到知识库中。知识库可以用来存储复杂的数据结构。我们建议你从使用`InMemoryKnowledgeBase`开始。一旦你想要使用大量的数据的时候，你可以选择创建自定义知识库。

为了初始化`InMemoryKnowledgeBase`，你需要以json的格式提供数据。下面的例子中包含了restaurants和hotels的数据。json结构应该包含每个对象类型的关键字，如restaurant和hotel。每个对象类型与一系列的对象对应 - 这里有3个restaurants和3个hotels。

```json
{
    "restaurant": [
        {
            "id": 0,
            "name": "Donath",
            "cuisine": "Italian",
            "outside-seating": true,
            "price-range": "mid-range"
        },
        {
            "id": 1,
            "name": "Berlin Burrito Company",
            "cuisine": "Mexican",
            "outside-seating": false,
            "price-range": "cheap"
        },
        {
            "id": 2,
            "name": "I due forni",
            "cuisine": "Italian",
            "outside-seating": true,
            "price-range": "mid-range"
        }
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
        {
            "id": 1,
            "name": "Hilton",
            "price-range": "expensive",
            "breakfast-included": true,
            "city": "Frankfurt am Main",
            "free-wifi": true,
            "star-rating": 4,
            "swimming-pool": false
        },
        {
            "id": 2,
            "name": "B&B",
            "price-range": "mid-range",
            "breakfast-included": false,
            "city": "Berlin",
            "free-wifi": false,
            "star-rating": 1,
            "swimming-pool": false
        },
    ]
}
```

一旦数据定义到json文件中，如，data.json，你将能够使用这个数据文件创建`InMemoryKnowledgeBase`，它可以被传递给action，用来检索知识库。

想要使用默认实现，你的知识库中的每个对象治好有name和id两个属性。如果没有，你需要自定义你的`InMemoryKnowledgeBase`。

### 定义NLU数据

这一节：

- 我们会引入新的意图，`query_knowledge_base`
- 我们将标记`mention`实体，使得我们的模型能够检测出对象的间接指代，如“the first one”
- 我们将使用[synonyms](https://rasa.com/docs/rasa/nlu/training-data-format/#entity-synonyms)扩展

为了让机器理解用户想要从知识库中检索知识，你需要定义新的意图。我们这里称之为`query_knowledge_base`。

我们可以将ActionQueryKnowledgeBase可以处理的请求分为两类：（1）用户想要获取特定类型的对象列表，或者（2）用户想要知道对象的某个属性。意图需要包含这些请求的不同变体：

```
## intent:query_knowledge_base
- what [restaurants](object_type:restaurant) can you recommend?
- list some [restaurants](object_type:restaurant)
- can you name some [restaurants](object_type:restaurant) please?
- can you show me some [restaurant](object_type:restaurant) options
- list [German](cuisine) [restaurants](object_type:restaurant)
- do you have any [mexican](cuisine) [restaurants](object_type:restaurant)?
- do you know the [price range](attribute:price-range) of [that one](mention)?
- what [cuisine](attribute) is [it](mention)?
- do you know what [cuisine](attribute) the [last one](mention:LAST) has?
- does the [first one](mention:1) have [outside seating](attribute:outside-seating)?
- what is the [price range](attribute:price-range) of [Berlin Burrito Company](restaurant)?
- what about [I due forni](restaurant)?
- can you tell me the [price range](attribute) of [that restaurant](mention)?
- what [cuisine](attribute) do [they](mention) have?
...
```

上面的例子仅显示除了与restaurant领域相关的示例。你需要对你的知识库中出现的每个对象类型添加示例到`query_knowledge_base`意图。

除了针对每个查询类型，要添加多样的训练示例之外，你还需要标注示例中的实体：

- `object_type`：每当训练示例引用知识库中的特定对象类型时，应将该对象类型标记为实体。使用[synonyms](https://rasa.com/docs/rasa/nlu/training-data-format/#entity-synonyms) 将`restaurants`映射到`restaurant`，正确的对象类型作为知识库中的key列举出来。
- `mention`：如果用户通过“the first one”，“that one”或者“it”指代一个对象，你应该将它们标记为"mention"。我们也使用synonyms将一些指代映射到同义词。后面会介绍消除指代。
- `attribute`：定义在你的知识库中的所有的属性值应该在nlu data中被标记为`attribute`。同样，使用同义词将属性名的变体映射到知识库中使用的属性名。

记住在你的domain文件中添加这些实体（entities和slots）：

```
entities:
  - object_type
  - mention
  - attribute

slots:
  object_type:
    type: unfeaturized
  mention:
    type: unfeaturized
  attribute:
    type: unfeaturized
```

### 创建一个Action用来查询你的知识库

为了创建你自己的基于知识库的action，你需要继承`ActionQueryKnowledgeBase`，并将知识库传递给`ActionQueryKnowledgeBase`：

```python
class MyKnowledgeBaseAction(ActionQueryKnowledgeBase):
    def __init__(self):
        knowledge_base = InMemoryKnowledgeBase("data.json")
        super().__init__(knowledge_base)
```

一旦你创建`ActionQueryKnowledgeBase`，你需要将`KnowledgeBase` 传递给构造函数。这个可以是 `InMemoryKnowledgeBase` 或者你自己实现的`KnowledgeBase`。你只能从一个知识库中提取信息，因为不支持同时使用多个知识库。

这就是完整的action的代码，这个action的名字是`action_query_knowledge_base`。不要忘了在domain中添加：

```
actions:
- action_query_knowledge_base
```

**注意**：如果你重写了默认的action名字action_query_knowledge_base。你需要在你的domain文件中添加三个unfeaturized slots：`knowledge_base_objects`, `knowledge_base_last_object`, `knowledge_base_last_object_type`。这些slots在`ActionQueryKnowledgeBase`内部会被使用到。如果你保持默认的名字，这些slots会被自动添加。

你也需要确保stories中包含`query_knowledge_base`意图和 `action_query_knowledge_base`行为。如：

```
## Happy Path
* greet
  - utter_greet
* query_knowledge_base
  - action_query_knowledge_base
* goodbye
  - utter_goodbye
```

最后一件需要处理的事情是，在你的domain文件中定义 `utter_ask_rephrase`。如果这个action不知道如何处理用户的请求，它将使用模板来要求用户进行重新描述。比如，domain中的模板如下：

```
utter_ask_rephrase:
- text: "Sorry, I'm not sure I understand. Could you rephrase it?"
- text: "Could you please rephrase your message? I didn't quite get that."
```

当添加完所有相关的内容后，该action可以用来查询知识库了。

## 它是如何工作的

`ActionQueryKnowledgeBase` 会对当前请求中提取出来的实体进行检索，也会依据之前设定的slots来决定需要检索什么。

### 从知识库中检索对象

为了从知识库中检索任何类型的对象，用户的请求中需要包含对象类型。如：

```
Can you please name some restaurants?
```

这个问题包含了关心的对象类型是：“restaurant”。bot会挑选这个实体构建查询语句，否则action将不知道用户关心什么对象。

当用户说：

```
What Italian restaurant options in Berlin do I have?
```

用户想要检索包括以下信息的restaurant，1）有Italian口味，2）位于Berlin。如果NER从用户的请求中检测出了这些属性，action将用这些属性对知识库中的restaurant进行过滤。

为了让bot检测出这些属性，你需要将Italian和Berlin在NLU数据中标记为实体：

```
What [Italian](cuisine) [restaurant](object_type) options in [Berlin](city) do I have?.
```

这些属性的名字，‘cuisine’和‘city’，应该和知识库中的内容一致。你也需要在domain文件中添加对应的实体和slots。

### 从知识库中检索对象的属性

如果用户想要获取对象的特定信息，请求将包含对象和感兴趣的属性，如：

```
What is the cuisine of Berlin Burrito Company?
```

用户想要获取restaurant（Berlin Burrito Company）的cuisine（感兴趣的属性）。

在NLU数据中，这些属性和对象的标记如下：

```
What is the [cuisine](attribute) of [Berlin Burrito Company](restaurant)?
```

确保添加restaurant作为entity和slot到你的domain文件中。

### 消除指代

根据上面的例子，用户可能并不总是用自己的名字来指代餐馆。用户可以通过其名称引用感兴趣的对象，例如“Berlin Burrito Company”（对象的表示字符串），也可以通过提及引用先前列出的对象，例如：

```
What is the cuisine of the second restaurant you mentioned?
```

我们的action能够将这些指代解析成对应知识库中的对象。更具体的说，它能够解决两种指代类型：（1）序列提及，如“the first one”，和（2）如it和that one形式的指代。

**ordinal mentions**

当用户用列表中的位置指代一个对象的时候，他被称为ordinal mentions。比如：

- User: What restaurants in Berlin do you know?
- Bot: Found the following objects of type ‘restaurant’: 1: I due forni 2: PastaBar 3: Berlin Burrito Company
- User: Does the first one have outside seating?

用户使用the first one指代“I due forni”，使用“the second one”指代PastaBar，也可能还会使用“3”，“any”。

Ordinal mentions通常会在将对象列表返回给用户进行选的时候用到。为了知道对应的实际对象，我们使用了ordinal mention 映射。默认的映射如下：

```python
{
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
```

由于ordinal mention映射没有包括“the first one”一类的指代，此处就需要用到[Entity Synonyms](https://rasa.com/docs/rasa/nlu/training-data-format/#entity-synonyms) 将“the first one”映射到“1”：

```
Does the [first one](mention:1) have [outside seating](attribute:outside-seating)?
```

NER检测出first one为mention实体，但是在mention slot中设置了1。因此我们的action能够采用mention slot和ordinal mention映射将first one解析为实际对象“I due forni”。

你可以在实现自己的`KnowledgeBase` 时候调用函数`set_ordinal_mention_mapping()`来重写ordinal mention映射。

**其他指代**

让我们来看一下下面的对话：

- User: What is the cuisine of PastaBar?
- Bot: PastaBar has an Italian cuisine.
- User: Does it have wifi?
- Bot: Yes.
- User: Can you give me an address?

问题“Does it have wifi?”,中it指代“PastaBar”。如果NER识别出it为mention实体，知识库action能够将其解析为对话中最近一次提到的对象，“PastaBar”.

在下一个输入中，用户没有直接指出“PastaBar”对象。知识库的action将检测出用户想要获取特定的属性，此处是，address。如果没有指代，或对象没有被NER识别出来，action假设用户指代的是最近提到的对象，“PastaBar”。

你可以禁用这种行为，通过在初始化action的时候将 `use_last_object_mention`设定为`False`。


## 定制化

### 定制化`ActionQueryKnowledgeBase`

如果你想要定义bot会对用户说什么，你可以重写 `ActionQueryKnowledgeBase`的以下两个函数：

- `utter_objects()`
- `utter_attribute_value()`

`utter_objects()`用于当用户请求一系列的对象的时候。一旦bot从知识库中获取了对象，它将用默认的消息返回给用户，如：

```
Found the following objects of type ‘restaurant’: 1: I due forni 2: PastaBar 3: Berlin Burrito Company
```

如果没有找到对象：

```
I could not find any objects of type ‘restaurant’.
```

如果你想要改变对话格式，你可以在你的action中重写`utter_objects()`。

函数`utter_attribute_value()`决定当用户询问对象特定的属性的时候bot会返回的消息。

如果找到了感兴趣的属性，回复为：

```
‘Berlin Burrito Company’ has the value ‘Mexican’ for attribute ‘cuisine’.
```

否则为：

```
Did not find a valid value for attribute ‘cuisine’ for object ‘Berlin Burrito Company’.
```

如果你想要改变对话格式，你可以在你的action中重写 `utter_attribute_value()`.。

**注意**：此处给出了如何使用自定义知识库的blog。[rasa blog - 将Rasa和知识库进行集成](https://zhuanlan.zhihu.com/p/85911378)

### 创建你自己的基于知识库的Actions

`ActionQueryKnowledgeBase` 能够让你很方便的将知识库集成到你的action。但是，这个action只能够处理两种用户请求：

- 从知识库中获取对象列表
- 从知识库中获取特定对象的属性

action不能够比较对象，或考虑对象之间的关系。此外，总是将其他的指代解释为交谈中最近一次提到的对象并不总是合理的。

如果你想处理更加复杂的用例，你可以自定义action。我们在 `rasa_sdk.knowledge_base.utils`添加了一些helper函数 ([link to code](https://github.com/RasaHQ/rasa-sdk/tree/master/rasa_sdk/knowledge_base/) ) 帮助你实现你自己的方案。我们建议你使用`KnowledgeBase` 的接口，这样你仍然可以在你的action中使用`ActionQueryKnowledgeBase`。

如果你写基于knowledge的action用来处理上面提到过的或新的场景，可以通过 [forum](https://forum.rasa.com/)告诉我们。

### 定制化`InMemoryKnowledgeBase`

 `InMemoryKnowledgeBase`类继承自`KnowledgeBase`.你可以通过重写下面函数自定义`InMemoryKnowledgeBase`：

~~ `get_key_attribute_of_object()`：为了跟踪用户上次谈论的对象，我们将key属性的值存储在一个特定的槽中。每个对象都应该有一个唯一的键属性，类似于关系数据库中的主键。默认情况下，每个对象类型的key属性的名称都设置为id。你可以通过调用 `set_key_attribute_of_object()`重写特定对象的key属性的名字。

~~ `get_representation_function_of_object()`:让我们关注下面的restaurant。

```json
{
    "id": 0,
    "name": "Donath",
    "cuisine": "Italian",
    "outside-seating": true,
    "price-range": "mid-range"
}
```

当用户要求bot列出任意Italian restaurant，它不需要restaurant的详细信息。相反，你需要提供一个有意义的名称来标识餐厅——在大多数情况下，对象的名称将起作用。函数 `get_representation_function_of_object()`返回lambda函数，将上述的restaurant对象映射到他的名字。

```python
lambda obj: obj["name"]
```

每当bot讨论到特定的对象的时候，这个函数都会被调用，因此用户需要对对象提供一个有意义的名字。

默认情况下，lambda函数返回对象的“name”属性。如果你的对象没有“name”属性，或者对象的“name”属性含糊不清的，你应该调用 `set_representation_function_of_object()`针对这个对象类型设置性的lambda函数。

~~ `set_ordinal_mention_mapping()`，ordinal mention 映射用来解决ordinal mention，比如“second one”。默认情况下，这个映射是这样的：

```python
{
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
```

你可以通过调用函数`set_ordinal_mention_mapping()`重写它。如果你想要进一步了解如何使用此映射，请继续查看之前的**消除指代**的内容。

请参阅[example bot](https://github.com/RasaHQ/rasa/blob/master/examples/knowledgebasebot/actions.py)获取InMemoryKnowledgeBase的示例实现，该示例使用了方法`set_representation_function_of_object（）`覆盖对象类型“hotel”的默认表示。InMemoryKnowledgeBase本身的实现可以在[rasa-sdk](https://github.com/RasaHQ/rasa-sdk/tree/master/rasa_sdk/knowledge_base/) 包中找到。

### 创建你自己的知识库

如果你有更多的数据，或你想要使用更加复杂的数据结构，比如，包含不同对象之间的关系，你可以创建你自己的知识库实现。只要集成`KnowledgeBase`类，和实现`get_objects()`，`get_object()`，``get_attributes_of_object()`方法。[knowledge base code](https://github.com/RasaHQ/rasa-sdk/tree/master/rasa_sdk/knowledge_base/)提供了这些函数应该做什么的详细介绍。

你可以进一步的自定义你的知识库，通过**定制化`InMemoryKnowledgeBase`**一节中提到的方法。

**注意**：相关的blog，[rasa blog - 为Rasa设置知识库来编码领域知识](https://zhuanlan.zhihu.com/p/85925978)





