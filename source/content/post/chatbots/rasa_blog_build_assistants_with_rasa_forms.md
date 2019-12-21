+++
date="2019-12-21"
title="rasa blog - 利用Rasa Forms创建上下文对话助手 （翻译）"
categories=["chatbot","rasa","翻译"]
tags=["rasa-blog"]
+++

# rasa blog - 利用Rasa Forms创建上下文对话助手 （翻译）

一个超越简单的FAQ式交互的上下文对话助手[contextual assistant](https://blog.rasa.com/conversational-ai-your-guide-to-five-levels-of-ai-assistants-in-enterprise/) 需要的不仅仅是一个算法和一个祷告。一个上下文对话助手需要在正确的上下文中收集重要信息帮助更好的回答用户。否则，是no happy path。简单来讲就是槽填充（slot filling）。但在采取行动或提供回应之前，你如何收集和定义重要的细节？

FormPolicy的使用会使得slot filling变得简单。这是一个新的功能，以一种简单有效的方式实现插槽填充。怎么实现呢？FormPolicy允许你使用单个故事覆盖所有的happy paths。forms也允许你在不改变训练数据的前提下，改变happy path内部的逻辑。

因此，你改如何实现这个新的技术呢？很高兴你这么问了，这里给出了详细介绍。

## Slot Filling介绍

slot filling是为了满足用户需求收集重要信息片段的过程。这个取决于相关的有用的数据能够从对话中获取。

让我们以餐厅搜索助理为例来说明这一点。在助理实际执行餐厅搜索操作之前，它必须了解用户的偏好，如烹饪、价格范围、位置等，以便提出有用的建议。

为了存储这些信息，rasa使用插槽。在简单的情况下，只需使用slot就可以实现slot填充，但一旦会话的复杂性增加，事情就会很快变得复杂起来。更多的细节带来更多可能的对话回合，这样对于训练数据的需求越来越大。这就是form action要帮助我们解决的问题。它允许您对信息收集过程实施严格的逻辑，并大大减少构建良好对话模型所需的训练数据量。

## 使用Rasa Forms构建restaurant查找助手

在这篇文章的剩下部分，你将会学会如何在事件中使用FormPolicy。这篇文章基于formbot餐厅搜索助手。你可以在Rasa Github上面找到它：

```
git clone https://github.com/RasaHQ/rasa.git
cd rasa/examples/formbot
```

按照这篇文章和使用对应的代码，你将构建一个有意思的助手，能够根据你的偏好推荐餐厅，如烹饪，人的数量，位置等其他需求。用户和助手之间的对话看上去如下：

![](./image/typo_fix.gif)

### Step1：Extracting details from user inputs using Rasa NLU

在将重要信息作为slot进行存储之前，助手需要从用户输入中将其提取出来。

为了保证你的助手能够完成这些任务，有必要训练NLU模型，该模型可以用于用户输入意图的分类和实体提取。

formbot例子中已经带有了训练NLU模型需要的数据。使用提供的训练示例，你可以教会助手理解像问候，餐厅检索，提供所需信息的输入等输入，然后提取像烹饪、人数、附加需求等实体。formbot例子对应的训练示例见：[data/nlu.md](https://github.com/RasaHQ/rasa/blob/master/examples/formbot/data/nlu.md) 。

为了训练该模型，执行下面的命令。该命令将会调用Rasa NLU训练函数，将训练数据和模型配置文件作为参数传入，最后将训练出来的模型输出到你的模型目录中：

````
rasa train nlu
````

注意：如果你不了解Rasa NLU，并且向学习它，参见：[documentation](http://rasa.com/docs/rasa/user-guide/rasa-tutorial/)。

### Step2: Training the dialogue model: handling the happy path with forms

当助手能够理解用户的输入时，就是构建[dialogue management model](http://rasa.com/docs/rasa/core/about/)的时刻。当使用Forms进行槽位填充的时候，最好的开始的方式是，训练模型处理happy paths，即用户提供了所有需要的信息，使得助手能够得出合理的结论。

form最佳的部分是助手从单个故事中学习处理所有的happy path。**https://www.youtube.com/embed/WK7C8pLnfvY**。译者注：原文提供了youtobe地址，无法加载。。。

#### How does it work?

一旦form action `restaurant_form`获得执行的时候，助手会不停地询问，直到所有的slots都被设定好。此处，对于用户如何提供信息是没有限制的。如果用户在一开始的请求中指出了所有偏爱，如`book me tablefor two at the Chinese restaurant`，助手会跳过关于询问`cuisine`和`number of people`的问题。

如果用户在初始化请求的时候没有提供任何相关的信息，他么助手会按着问题，一个一个进行细问，知道获取所有需要的内容。这些场景展示了两种不同的对话模式（可以有比两种更多），但是通过使用FormsPolicy，使用单个故事就能够学习到。下面是训练故事的一个片段，用来构建formbot的所有happy paths：

```
## happy path
* greet
    - utter_greet
* request_restaurant
    - restaurant_form
    - form{"name": "restaurant_form"}
    - form{"name": null}
    - utter_slots_values
* thankyou
    - utter_noworries
```

具体内容查看：[data/stories.md](https://github.com/RasaHQ/rasa/blob/master/examples/formbot/data/stories.md) 

### Step3: Defining the domain

为了利用Rasa训练对话管理模型，你需要定义domain文件。这里你会指出哪些信息需要被存储为slots。

当为有slot filling的助手定义domain的时候，你需要考虑三个重要的事情：

1. 在Rasa中，不同的[slot types](http://rasa.com/docs/rasa/core/slots/)对于下一个action的预测有不同的影响。当使用FormAction进行槽填充的时候，你需要执行严格的规则，告诉你的助手下一步需要什么信息。这样，你就允许form action通过单个故事处理所有的happy paths，因为它会检查哪些slot已经填充，哪些仍然空缺。为此，domain文件中的需要请求的slots需要被定义成unfeaturized。
2. 被用来询问需要的slot信息的模板的名字需要遵循格式`utter_ask_{slotname}`。这对于让FormAction知道什么模板用于什么slot是很重要的。
3. domain配置中除了常见的部分（intents，entities，templates，actions和slots），你将还需要包含forms。这一节需要包含你的助手会基于训练数据文件中的故事调用到的所有form actions的名字。

下面是formbot例子中domain的一个片段：

```
entities:
  - cuisine
  - num_people
  - number
  - feedback
  - seating

slots:
  cuisine:
    type: unfeaturized
    auto_fill: false
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
    
forms:
    - restaurant_form
```

### Step4: Defining the FormAction

下一步是实际实现FormAction，一旦预测到该Action，就会用来处理slot filling。你可以在`actions.py`文件中实现所有的`FormActions`，和实现其他自定的action放到一块。让我们一步一步的来实现FormAction用于餐厅助手的slot filling（从[actions.py](https://github.com/RasaHQ/rasa/blob/master/examples/formbot/actions.py)查看完整实现）

- 首先定义form action类。注意，form action继承自FormAction类：

```python
  class RestaurantForm(FormAction):
      """Example of a custom form action"""
```

- 第一个需要定义的函数是`name`，该函数用来定义action的名字（这个名字出现在domain文件中）。在餐厅搜索示例中，名字是`restaurant_form`。

```python
  class RestaurantForm(FormAction):
      """Example of a custom form action"""
  
      def name(self):
          """Unique identifier of the form"""
          return "restaurant_form"
```

- 下一步，实现`required_slots`函数，该函数用来定义在给用户的请求提供回复之前，需要助手填充的slots列表：

```python
  class RestaurantForm(FormAction):
      """Example of a custom form action"""
  
      def name(self):
          """Unique identifier of the form"""
          return "restaurant_form"
      
  
  	@staticmethod
      def required_slots(tracker: Tracker) -> List[Text]:
          """A list of required slots that the form has to fill"""
  
          return ["cuisine", "num_people", "outdoor_seating",
                  "preferences", "feedback"]
```

  **技巧**：`reqyured_slots`函数是一个很适合引入一些自定义逻辑的位置。比如，仅仅针对特别的cuisine的餐厅引入outdoors seating选项是有意义的。你可以以简单的方式实现这个逻辑，如下：

```python
  def required_slots(tracker):
      # type: () -> List[Text]
      """A list of required slots that the form has to fill"""
  
      if tracker.get_slot('cuisine') == 'greek':
          return ["cuisine", "num_people", "outdoor_seating",
               	"preferences", "feedback"]
      else:
          return ["cuisine", "num_people",
               	"preferences", "feedback"]
```

- 创建一个简单的form action最后的一个步骤是定义`submit`函数，该函数定义了到所有的需要设置的slots都被赋值后需要处理的事情。针对餐厅查找的情况，当所有的slots都赋值后，助手会基于domian文件中的定义执行`utter_submit`。通过这个信息，我们可以确定助手已经完成了相关的任务。

```python
  class RestaurantForm(FormAction):
      """Example of a custom form action"""
  
      def name(self):
          """Unique identifier of the form"""
          return "restaurant_form"
  
  
      @staticmethod
      def required_slots(tracker: Tracker) -> List[Text]:
          """A list of required slots that the form has to fill"""
  
          return ["cuisine", "num_people", "outdoor_seating",
                  "preferences", "feedback"]
  
      def submit(self):
          """Define what the form has to do
              	after all required slots are filled"""
  
         	dispatcher.utter_template('utter_submit', tracker)
          return []
```

到现在，你已经实现了一个简单的`FormAction`。为了让你的助手处理更高级的场景，有很多内容可以加。让我们在下一篇文章中介绍。

### Step5: Handling the advanced cases with FormAction

一些必要的slots可以从非常不同的用户输入中获取。比如，用户可以回答问题`Would you like to sit outside?`，然后可能会有如下回复：

- Yes
- No
- I prefer sitting indoors (or similar direct answer)

上面的每个答案都与不同的意图相关，或者有不同的重要的实体，但是，由于他们针对问题提供了可行的答案，助手就必须接受它，并且设置slot值，继续执行。

这就是FormAction中`slot_mapping`函数的作用，它定义了如何从可能的用户响应中提取slot值并将它们映射到特定的slot。下面是将`slot_mappings`函数用于前面讨论的`outdoor_seating` slot的一个例子。基于定义的逻辑，`outdoor_seating` slot可以使用以下方式填充：

- 如果针对问题返回`affirm`的意图，那么值为True
- 如果针对问题返回`deny`的意图，那么值为False
- 提取到的`seating`值。

```python
def slot_mappings(self):
    # type: () -> Dict[Text: Union[Dict, List[Dict]]]
    """A dictionary to map required slots to
    - an extracted entity
    - intent: value pairs
    - a whole message or a list of them, where a first 
                                 match will be picked"""

    return { "outdoor_seating": [self.from_entity(entity="seating"),
                      self.from_intent(intent='affirm',
                                                 value=True),
                      self.from_intent(intent='deny',
                                                 value=False)]}
```

在示例的文件[actions.py](https://github.com/RasaHQ/rasa/blob/master/examples/formbot/actions.py)中，可以找到更多的关于`slot_mapping`的示例。

你可以使用FormAction可以做的另一件有用的事情是`slot validation`。举个例子，在允许你的助手带着问题继续前进之前，你也许想要依据你的数据库中存储的值对slot的设置的值进行校验，或者对slot值的格式进行校验。你可以通过`FormAction`类中的`validate`函数实现这个功能。默认情况下，它会校验请求的slot有没有被提取出来，但是你可以添加更多的逻辑。下面是一个校验函数的例子，该函数首先检查请求的slot是否设定了值，接着检查提供的数值是不是具有正确的格式：number是不是整型，如果校验通过，助手将使用提供的数值，否者会返回一条消息说明slot值是不合理的，将slot设定为None，然后继续询问。

```python
def validate(self,
                 dispatcher: CollectingDispatcher,
                 tracker: Tracker,
                 domain: Dict[Text, Any]) -> List[Dict]:
    """Validate extracted requested slot
            else reject the execution of the form action
    """
    # extract other slots that were not requested
    # but set by corresponding entity
    slot_values = self.extract_other_slots(dispatcher, tracker, domain)

    # extract requested slot
    slot_to_fill = tracker.get_slot(REQUESTED_SLOT)
    if slot_to_fill:
        slot_values.update(self.extract_requested_slot(dispatcher, tracker, domain))
        if not slot_values:
            # reject form action execution
            # if some slot was requested but nothing was extracted
            # it will allow other policies to predict another action
            raise ActionExecutionRejection(self.name(),
                                           "Failed to validate slot {0}"
                                           "with action {1}"
                                           "".format(slot_to_fill,
                                                         self.name()))

    # we'll check when validation failed in order
    # to add appropriate utterances
    for slot, value in slot_values.items():
        if slot == 'num_people':
            if not self.is_int(value) or int(value) <= 0:
                dispatcher.utter_template('utter_wrong_num_people',
                                              tracker)
                # validation failed, set slot to None
                slot_values[slot] = None

```

详细的实现可以参见：[actions.py](https://github.com/RasaHQ/rasa/blob/master/examples/formbot/actions.py)。

### Step6: Handling the deviations from the happy path

FormAction填槽背后的思想是按照严格的逻辑对重要的信息片段进行收集，并处理happy paths，而使用常规的机器学习来优雅的处理happy paths途径中的意外变化。这些意外变化可以是form action会话中的一些闲聊的消息，也可能是用户拒绝提供所有必要细节的情况。为了处理这样的情况，你必须写出代表这种对话转折的故事。

举个例子，下面的故事给出了用户在form action对话中途停止提供有用的信息，再后面在提供信息的场景：

```
## stop but continue path
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
```

为了让助手处理更加复杂的场景，你需要收集更多的故事来覆盖不同的对话场景。在一些场景，你也许想要依据询问的slot以不同的方式处理unhappy paths。为了实现这个，在domain文件中，你需要将`requested_slot`设置成categorical。这将允许下一个回复的预测受到当前请求的slot的值的影响。

具体查看[data/stories.md](https://github.com/RasaHQ/rasa/blob/master/examples/formbot/data/stories.md) 。

### Step7: Testing the restaurant search assistant

到目前为止，你已经学习了很多关于实现formaction的知识，并定义了对话管理模型的所有必要部分。现在是时候测试一下助理了！

首先，使用下面的命令训练对话管理模型，这将会调用Rasa train函数，并将domain和数据文件传入其中，然后将训练结果存储到你的工作目录的model目录下面。

```
rasa train
```

当训练得到模型之后，就要测试bot餐馆搜索的功能。

首先在一个新的终端，运行下面的命令运行duckling 服务。

```
docker run -p 8000:8000 rasa/duckling
```

通过执行以下命令启动助手，该命令将启动本地服务器以执行自定义操作，并加载助手以供聊天：

```
rasa run actions&
rasa shell -m models --endpoints endpoints.yml
```

为了更好地了解FormAction是如何工作的，需要花时间测试助手对于不同的happy和unhappy路径是怎么响应的。

## 小结

创建一个好的上下文助手并不是一件容易的事情。将FormAction和传统的机器学习相结合，使得你在不需要写很多的训练故事的情况下构建能够处理更深层次对话的助手。除了这一点，FormAction使得更改代码和训练故事中没有被用到的请求slot的对话变得更加容易了。

在你的数据集上测试新的forms，可以在[this thread](https://forum.rasa.com/t/building-contextual-assistants-with-rasa-forms/4228)和我们分享你的反馈。

## 原文链接

https://blog.rasa.com/building-contextual-assistants-with-rasa-formaction/
