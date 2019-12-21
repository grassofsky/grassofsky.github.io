+++
date="2019-12-21"
title="对话系统Rasa - slots [翻译]"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统rasa - slots [翻译]

## 什么是slots

**slots是机器的记忆**。他们以key-value的形式进行存储，存储的信息包括用户提供的，以及从外面世界收集的（如数据库检索的结果）。

大多数情况，你希望slots能够影响对话的进度。针对不同的行为有不同的slot类型。

举个例子，如果用户提供了他们所住的城市，那么你可能带有一个`text`类型的slot，叫做`home_city`。如果用户询问天气，而你不知道他们所住的城市，那么你回提问，让用户告诉你他们所住的城市。一个`text`slot仅告诉rasa core，这个slot是否有值。`text`slot的值并没有任何的区别。

如果这个值本身也很重要，那么使用`categorical`或`bool`slot。rasa还提供了`float`，`list`slot。如果你只想存储一些数据，但是不想要它影响对话流程，那么使用`unfeaturized`slot。

## Rasa怎么使用slots

Policy没法访问你的slots的值。它接受特征化的表述。正如上面提到的，对于`text`slot的值是无关紧要的。policy只能够看到1或0，依赖于是否被这是过。

**你需要小心地选择slot的类型**

## slots怎么进行Get Set

你可以在domain文件中提供slot的初始值：

```
slot:
  name:
    type: txt
    initial_value: "human"
```

这里有多个方法在对话中设置slots：

### 从NLU中设置Slots

如果你的NLU模型挑选了一个实体，并且你的domain文件中包含有相同名字的slot，那么这个slot会被自动设置。如：

```
# story_01
* greet{"name": "Ali"}
  - slot{"name": "Ali"}
  - utter_greet
```

在这个示例中，你没有必要在story中包括 `-slot{}`部分，因为他会被自动选择。

某些特殊的slot不想要这样的自动设值的功能，那么可以显式的取消掉，如下：

```
slots:
  name:
    type: text
    auto_fill: False
```

### 通过点击button设置slots

你可以把button当做shortcut使用。rasa core将会发送以`/`开头的消息给`RegexInterpreter`，它期望NLU输入和故事文件有相同的格式，如，`/intent{entities}`。举个例子，如果你想让用户通过点击的方式选择颜色，这个button可能是`/choose{"color": "blue"}`和`/choose{"color":"red"}`。

你可以在domain文件中给出定义：

```
utter_ask_color:
- text: "what color would you like?"
  buttons:
  - title: "blue"
    payload: '/choose{"color": "blue"}'
  - title: "red"
    payload: '/choose{"color": "red"}'
```

### 在actions设定slots

另一个可选的实现方式是，可以在自定义actions（[custom actions](https://rasa.com/docs/rasa/core/actions/#custom-actions)）返回的事件中设定slots。这种情况，你的story需要包括slots。举个例子，你有一个自定义action获取用户的个人信息，同时你有`categorical`类型的slot，名字为`account_type`。当`fetch_profile`动作执行的时候，它会返回`rasa.core.events.SlotSet`事件：

```
slots:
  account_type:
    type: categorical
    values:
    - premium
    - basic
```

```python
from rasa_sdk.actions import Action
from rasa_sdk.events import SlotSet
import requests

class FetchProfileAction(Action):
    def name(self):
        return "fetch_profile"
    
    def run(self, dispatcher, tracker, domain):
        url = "http://myprofileurl.com"
        data = requests.get(url).json
        return [SlotSet("account_type", data["account_type"])]
```

```
# story_01
* greet
  - action_fetch_profile
  - slot{"account_type": "premium"}
  - utter_welcome_premium
  
# story_02
* greet
  - action_fetch_profile
  - slot{"account_type": "basic"}
  - utter_welcome_basic
```

在这个例子中，你必须在故事中包含`- slot{}`部分。Rasa Core将学习通过这些信息决定采用正确的动作（如这里的，utter_welcome_premium，utter_welcome_basic）

**注意**，通过手写故事，很容易忘记slots。强烈建议构建故事的时候使用[Interactive Learning with Forms](https://rasa.com/docs/rasa/core/interactive-learning/#section-interactive-learning-forms) 取代手写。

## Slot类型

### Text slot

类型：text

用于：用户首选项，其中您只关心是否已指定它们。

示例：

```
slots:
  cuisine:
    type: text
```

描述：如果有值被设定，那么特征为1，否者特征为0.

### Boolean Slot

类型：bool

用于：True或False

示例：

```
slots:
   is_authenticated:
      type: bool
```

描述：确认slot是否被设置，并且是否是True

### Categorical Slot

类型：categorical

用于：slots可以是N个值中的其中一个

示例：

```
slots:
   risk_level:
      type: categorical
      values:
      - low
      - medium
      - high
```

描述：对values值匹配的进行one-hot编码。

### Float Slot

类型：float

用于：连续值

示例：

```
slots:
   temperature:
      type: float
      min_value: -100.0
      max_value:  100.0
```

默认值：`max_value=1.0,min_value=0.0`

描述：所有小于`min_value`的值会被认为是`min_value`，针对`max_value`也是类似的处理方式。因此，如果`max_value`设置成1，slot中2和3.5的值在特征化后就没有区别。（如，两个值对对话的影响是一致的，并且模型无法学到这两者的区别）

### List Slot

类型：list

用于：列表值

示例：

```
slots:
   shopping_items:
      type: list
```

描述：如果一个list的值设定了非空值，那么这个slot对应的特征就为1。相反的这个特征就为0. list的长度对对话没有影响。

### Unfeaturized Slot

类型：unfeaturized

用于：你想要存储不影响对话流程的数据。

示例：

```
slots:
   internal_user_id:
      type: unfeaturized
```

描述：这个slot并不会被特征化，因此这个值不会影响对话流程，当预测助手下一次性为的时候会被忽略。

## 自定义Slot类型

也许你的餐馆预定系统最多只能够处理6个客户的预定。这样的话，你需要有个slot值能够影响下一次选择行为。你可以自定义实现slot类。

在下面的代码中，我们定义了叫`NumberOfPeopleSlot`的slot类。featurization 定义了如何将slot获取的值转换成我们机器学习模型能够处理的向量。我们的slot含有三个值，可以用长度为2的向量表示：

(0,0)没有被设定；(1,0)1到6之间的值；(0,1)大于6的值

```python
from rasa.core.slots import Slot

class NumberOfPeopleSlot(Slot):
    def feature_dimensionality(self):
        return 2
    
    def as_feature(self):
        r = [0.0]*self.feature_dimensionality()
        if self.value:
            if self.value <= 6:
                r[0] = 1.0
            else:
                r[1] = 1.0
        return r
```

现在，我们可以训练一些故事，让Rasa学习到怎么处理不同的情况：

```
# story1
...
* inform{"people": "3"}
  - action_book_table
...
# story2
* inform{"people": "9"}
  - action_explain_table_limit
```

