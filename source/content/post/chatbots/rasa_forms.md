+++
date="2019-12-21"
title="对话系统Rasa - forms [翻译]"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统rasa - forms (翻译)

注意：这里有个关于Rasa Forms用于slot filling更深入的介绍，见：[here](https://blog.rasa.com/building-contextual-assistants-with-rasa-formaction/?_ga=2.30454137.775468082.1569460046-1750168845.1568702367)

一种最常用的对话模式是从用户收集一些信息来处理一些事情，如订餐馆，调用API，查找数据库等。这个又叫做填槽（slot filling）。

如果你需要连续收集多条信息，我们建议你创建`FormAction`。这是一个单独的操作，它会循环的要求用户提供所需槽位上的信息。在`examples/formbot`目录有关于forms的完整示例，链接为：https://github.com/RasaHQ/rasa/tree/master/examples/formbot。

当你定义form时，你需要在你的domain文件中进行添加。如果你的form名字是`restaurant_form`，你的domain看上去是这样的：

```
forms:
  - restaurant_form
actions:
  ...
```

可以见：https://github.com/RasaHQ/rasa/tree/master/examples/formbot/domain.yml。

## 配置文件

为了使用forms，必须在policy配置文件中包含`FormPolicy`，如：

```
policies:
  - name: "FormPolicy"
```

可以参见：https://github.com/RasaHQ/rasa/tree/master/examples/formbot/config.yml。

## Form基础

使用`FormAction`，你可以用单个故事描述所有的happy path。这里的happy path的意思是，无论你想用户要求询问什么信息，他们都会回复对应的信息。

如果我们需要一个restaurant bot，单个故事的描述如下：

```
## happy path
* request_restaurant
	- restaurant_form
	- form{"name": "restaurant_form"}
	- form{"name": null}
```

这个例子中用户的意图是`request_restaurant`，下面紧接着form action，`restaurant_form`。使用`form{"name": "restaurant_form"}`，form被激活，使用`form{"name":null}`停用form。在[Handling unhappy paths](https://rasa.com/docs/rasa/core/forms/#section-unhappy) 一节中显示，当form仍然激活的情况下，bot可以在form之外执行任意类型的actions。当处于“happy path”的时候，即用户能够很好的和系统进行沟通，系统能够正确的理解用户的时候，form会没有中断的将所有请求的slot填上值。

`FormAction`只会针对还没有被填值的slot进行请求。如果一个用户以这么个对话开始“I'd like a vegetarian Chinese restaurant for 8 people”，那么他们将不会被问到关于`cuisine`和`num_people`的slots。

注意，为了使这个故事工作，你的slots的类型应该是 [unfeaturized](https://rasa.com/docs/rasa/core/slots/#unfeaturized-slot). 如果这些slots中的某个是featurized，那么你的故事里面需要包含`slot{}`事件，用来显示这些slot被设置了。这种情况，最简单的方式是使用[Interactive Learning](https://rasa.com/docs/rasa/core/interactive-learning/#interactive-learning)来创建合理的故事。

在上面的故事中，`restaurant_form`是form action 的名字。这里有个例子说明它看上去是什么样子的。你需要定义三个方法：

- `name`：action的名字
- `required_slots`：提交方法工作所需要填充的slots列表
- `submit`：当所有slots都赋值后，在form的结尾需要做什么

```python
def name(self) -> Text:
	"""Unique identifier of the form"""
	return "restaurant_form"
	
@staticmethod
def required_slots(tracker: Trakcer) -> List[Text]:
    """A list of required slots that the form has to fill"""
    return ["cuisine", "num_people", "outdoor_seating", "preferences", "feedback"]

def sumbit(
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

一旦这个form action被第一次调用后，form被激活，`FormPolicy`开始介入。`FormPolicy`相当简单，用来预测form action。查看[Handling unhappy paths](https://rasa.com/docs/rasa/core/forms/#section-unhappy)，了解如何处理意料之外的用户输入。

每一次form action被调用的时候，它将向用户询问`required_slots`中还没有被赋值的下一个slot。这个行为的执行是通过寻找叫做`utter_ask_{slot_name}`的模板实现的，因此你可以在domain文件中针对需要的slot给出模板定义。

一旦你的slot被填完后，`submit()`方法会被调用，在这个方法里，你可以使用收集到的信息帮助用户处理事情，如查找restaurant。如果你不想要你的form在结束之后做什么事情，那么只要返回`return []`即可。在submit方法被调用之后，form就停止使用的，并且你的核心模块中其他的policies将会被用来预测下一个action。

## 自定义slot mappings

如果你没有定义slot mappings，那么slots将会被从用户输入中挑选出来的具有相同名字的实体作为slot。一些slots，像`cuisine`，可以使用单个entity被挑选出来，但是`FormAction`同样支持是/否的问题，不需要更多文本的输入。`slot_mappings`方法定义了如何从用户响应中提取slot值。

这里有个用于restaurant机器人的示例：

```python
def slot_mappings(self) -> Dict[Text, Union[Dict, List[Dict]]]:
    """A dictionary to map required slots to
       - an extracted entity
       - intent: value pairs
       - a whole message
       or a list of them, where a first match will be picked"""
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
```

这些预定义的函数解释如下：

- `self.from_entity(entity=entity_name, intent=intent_name)`会查找叫做`entity_name`的实体，如果`intent_name`是`None`，那么不用考虑意图，否则仅考虑对应名称的意图。
- `self.from_intent(intent=intent_name, value=value)`，如果用户的意图是`intent_name`，会对`slot_name`的slot赋值为`value`。可以用来构建布尔型的slot，看一下定义中的`outdoor_seating`。注意：用来触发该form action的用户意图的消息不会用来填充slot。这种情况可以使用`self.from_trigger_intent`。
- `self.from_trigger_intent(intent=intent_name, value=value)`，如果form被用户输入的意图`intent_name`触发，那么会对`slot_name`的slot赋值为`value`。
- `self.from_text(intent=intent_name)`将使用下一次用户的输入填充到名字为`slot_name`的slot，如果`intent_name`不为空，那么只设定对应意图的输入。
- 如果你想要使用这些函数的组合，那么像上例中以列表的形式提供。

## 校验用户输入

在从用户输入中提取slot值之后，form将尝试校验slot的值。注意，在默认情况下，只在用户输入之后马上执行form action的话会进行校验。这个可以通过改变Rasa SDK `FormAction`类中的函数`_validate_if_required()`实现。任何在表格激活之前填入的slots都会在激活的时候进行校验。

默认情况下，校验仅仅检查请求的slot有没有成功的从slot mapping中提取出来。如果你想要添加自定义校验，如，通过数据库对值进行校验，你可以实现以`validate_{slot-name}`命名的帮助校验函数。

下面是一个例子，`validate_cuisine()`，用来提取出来的cuisine是不是在支持的`cuisines`列表中。

```python
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
```

这个帮助校验函数返回了slot名字和值的字典，除了设置需要校验的slot，还可以设置更多的slots。但是，你需要确保那些额外的slot值是没问题的。

你可以在校验的时候，通过返回`self.deactivate()`，直接停用这个form。（比如slot填入了你没法处理的内容）。

如果从用户的语言中没有提取出想要的slots，将会抛出`ActionExecutionRejection`错误，也就是action执行被拒绝了，因此Core会回退到一个不同的policy用来预测另一个action。

## 处理unhappy路径

当然，用户并不会总是针对你的问题给出合理的回答。如，用户会问问题，会说话题之外的话，会改变他们的想法，或否则从happy path偏离。针对这种情况，form会抛出`ActionExecutionRejection`异常。你需要对故事中可能引起该异常的事件进行处理。如，如果你希望用户和你的bot进行chitchat，你可以像下面一样添加故事：

```
## chitchat
* request_restaurant
	- restaurant_form
	- form{"name": "restaurant_form"}
* chitchat
	- utter_chitchat
	- restaurant_form
	- form{"name": null}
```

在某些情况，用户在form action执行期间改变他们的想法，决定不需要继续之前的请求。像这种情况，助手需要停止询问请求的slots。你可以使用默认的行为`action_deactivate_form`愉快的处理这种情况，它将停用form，并重置requested slot。这样的故事类似如下：

```
## chitchat
* request_restaurant
    - restaurant_form
    - form{"name": "restaurant_form"}
* stop
    - utter_ask_continue
* deny
    - action_deactivate_form
    - form{"name": null}
```

强烈建议使用interactive learning构建这些故事。如果你徒手写这些故事，很有可能丢失重要的事情。详细参阅：[Interactive Learning with Forms](https://rasa.com/docs/rasa/core/interactive-learning/#section-interactive-learning-forms) 。

## requested_slot 插槽

`request_slot`会被作为unfeaturized slot自动添加到domain中。如果你想要使得它是featurized，你需要在你的domain文件中作为分类slot添加。当你想要根据当前询问用户的slot来处理你的unhappy paths的时候，你也许想要个这么做。举个例子，你的用户用一个问题来回复bot的问题，如`why do you need to know that?`对于这个`explain`意图的响应，依赖于我们在故事中的位置。在restaurant case，你的故事看上去是这样的：

```
## explain cuisine slot
* request_restaurant
    - restaurant_form
    - form{"name": "restaurant_form"}
    - slot{"requested_slot": "cuisine"}
* explain
    - utter_explain_cuisine
    - restaurant_form
    - slot{"cuisine": "greek"}
    ( ... all other slots the form set ... )
    - form{"name": null}

## explain num_people slot
* request_restaurant
    - restaurant_form
    - form{"name": "restaurant_form"}
    - slot{"requested_slot": "num_people"}
* explain
    - utter_explain_num_people
    - restaurant_form
    - slot{"cuisine": "greek"}
    ( ... all other slots the form set ... )
    - form{"name": null}
```

再次，强烈建议使用interactive learning来构建这些故事。详细阅读：[Interactive Learning with Forms](https://rasa.com/docs/rasa/core/interactive-learning/#section-interactive-learning-forms) 。

## 处理条件slot逻辑

很多forms相比较请求属性值，需要更多的逻辑。举个例子，如果有人请求`greek`作为他们的风味，你也许想问，他们是否查找能够在外面吃的餐馆。

你可以在`required_slots()`方法中写一些逻辑实现它，如：

```python
@staticmethod
def required_slots(tracker) -> List[Text]:
   """A list of required slots that the form has to fill"""

   if tracker.get_slot('cuisine') == 'greek':
     return ["cuisine", "num_people", "outdoor_seating",
             "preferences", "feedback"]
   else:
     return ["cuisine", "num_people",
             "preferences", "feedback"]
```

这种方式是非常通用的，你可以在你的forms中构建不同的逻辑。

## 调试

第一件事情是试着用debug标签运行你的bot，详细参见：[Command Line Interface](https://rasa.com/docs/rasa/user-guide/command-line-interface/#command-line-interface) 。如果你刚刚开始，你可能值有一些手写的故事。这是一个很好的开端，但是你应该将你的bot尽快的给到用户手中进行测试。Rasa Core建议的一个原则是：

**从实际对话中学习比设计一些假想的更加重要。**

因此，不要在给到测试之前，尝试着用手写故事的方式覆盖所有的场景。真实的用户行为总会让你惊讶的。

## 原文链接

https://rasa.com/docs/rasa/core/forms/