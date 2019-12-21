+++
date="2019-12-21"
title="rasa blog - 设计Rasa训练故事"
categories=["chatbot","rasa","翻译"]
tags=["rasa-blog"]
+++

# rasa blog - 设计Rasa训练故事

训练数据都是每个机器学习模型的关键部分。对于对话AI来说也是一回事。我们设计了Rasa为了避免手写规则，你的助手应该可以观察到真实的对话的样子，然后从这些对话中进行学习，使用学习到的知识来管理对话。

由于人类语言的复杂性，对话建模并不是一件简单的任务。这就需要精心设计会话数据，以优化助理学习的模式。在Rasa，这些对话数据被叫做Rasa Stories。

但是，你该怎么设计他们呢？在这篇文章中，我们将介绍实际的设计rasa训练故事的方案，来帮助你构建最佳的对话助手。这篇文章的例子，包括故事，都是从`Sara`中获取，它是构建与Rasa Stack之上的Rasa docs的开源助手。具体见：[here](https://github.com/RasaHQ/rasa-demo).

## Rasa故事介绍

Rasa stories是训练Rasa Core对话管理模型需要的训练数据的一种表现形式。一个故事可以表示为用户和AI助手之间的实际对话，这种对话被转化成了一种特殊的格式，用户的输入表示成了特定的意图（可以带上对应的实体），助手的回复被表示成了相关的action名字。

下面给出了Rasa故事的组成：

![](./image/stories_gif_new3.gif)

- 一个故事以`##`前缀开始。这个不是强制的，但是建议讲故事的名字写在上面，有助于阅读以及问题排查。当对你的故事进行命名的时候，最好给一个有意义的唯一的名称，能够这个故事想要表述的对话的类型。
- 用户的输入被表示成以`'*'`开头的意图。
- 针对对话特殊的状态，会影响对话走向的关键实体的名称以及对应的值也需要给出。
- 助手的回复以`'-'`打头，后面跟着action 的名称。
- 所有的故事的后面都需要跟着空行，用来标记训练故事的结束。

注意，在你的故事中，一个用户的意图后面可以跟多个回复。比如，下面的故事中当用户询问什么是Rasa NLU的时候，AI助手优先会对消息进行确认，如果能够处理的话，然后再执行后面的action：

```
* greet
  - utter_greet
* ask_product{"product":"Rasa NLU"}
  - utter_on_it
  - utter_explain_rasa_nlu
* thanks
```

通过观察训练故事，对话管理模型学习当前用户输入和会话的前一状态如何影响下一个响应的预测。但首先你如何获得这些会话训练数据呢？我们将在这篇文章的下一节讨论这个问题。

## 如何开始创建训练故事

获取培训数据是我们看到新的rasa开发人员面临的一个常见挑战。这很有挑战性，因为公开的对话数据源并不多。此外，大多数情况下，人工智能助理的培训数据必须为你的特定域和用例定制。所以，很可能你必须从头开始生成训练数据。

**产生训练数据**

为了使得这个过程够简单以及更加有效，你需要了解用户和你的助手之间最常见的对话形式，然后首先设计出happy path的流程。为了实现这一步，你可以使用类似[Botsociety](https://botsociety.io/) 的工具，来帮助你设计对话，检测在你偏爱的对话平台上会变得怎么样，以及更重要的是， [export them in a Rasa format and move to the development stage right away](https://www.youtube.com/watch?v=AF5O5zotXoQ).在设计完happy path之后，你应该生成更多的示例，来覆盖happy path的一些变种。RASA有三个核心特性，你可以利用它们更容易地生成更多的训练故事：交互式学习、检查点和/或语句。

**交互式学习**

译者注：交互式学习更多内容可以参见：[Rasa - Interactive Learning （翻译）](https://zhuanlan.zhihu.com/p/85583802)。

交互式学习是一种很好的方式在对话中来训练你的AI助手，以及生成训练故事。在交互式学习过程中，机器人要求你为它所做的每一个预测提供反馈-意图分类以及响应预测。在交互式学习课程中，你与人工智能助理的所有对话稍后都可以导出为NLU和对话训练示例，并附加到原始训练数据示例中。使用交互式学习，你还可以可视化培训故事，并在与助手交谈时查看对话流程的变化。

为了通过交互式学习训练助手，你必须提前准备一些训练故事。这个可以从之前提到的Rasa and Botsociety集成中获取 - 通过Botsociety设计happy path，然后导出成Rasa格式。

**checkpoints**

[Checkpoints](https://rasa.com/docs/core/stories/#checkpoints)可以帮助你模块化故事。检查点的主要思想是，你可以创建一个故事的蓝图，稍后可以使用检查点名称将其附加到任何故事中，而不是手动编写整个故事。为了创建检查点，你可以在想要复用的对话的结尾添加`> checkpoint_name`。比如：

```
## first story
* hello
  - action_ask_user_question
> check_asked_question
```

这意味着，你现在可以通过引用checkpoint的名字，在你的训练数据文件中的任何其他的故事添加这段对话。下面是两个例子。例子中每个故事都是以checkpoint开始，然后根据用户输入affirm或deny进行选择。

```
## user affirms question
> check_asked_question
* affirm
  - action_handle_affirmation

## user denies question
> check_asked_question
* deny
  - utter_goodbye
```

虽然checkpoints能够节约你创建训练故事的时间，但是他会很快的导致严重的内存问题，并且使得训练数据很难进行阅读。这是为什么你需要非常留意何时何地使用checkpoint，尽量在必要的情况才使用它。

**OR statements**

另一种加速写stroies的方法是使用[OR statements](https://rasa.com/docs/core/stories/#or-statements) 。这样，你可以在一个故事来描述不同的对话。如，一个故事通过提供两种可能意图覆盖了不同的对话：

```
## story
---
  - utter_ask_confirm
* affirm OR thankyou
  - action_handle_affirmation
```

OR语句对于能够包含的意图的数量是没有限制的。但是，和checkpoints类似，过度使用OR语句可能会导致内存问题，也可能意味着训练数据中的一些意图需要考虑合并。

一旦你实现了一个简单的对话助手，你需要让你的真实用户使用它，来帮助提高助手的易用性。这样做，将助手给到真实用户那，收集对话数据（[collect the conversational data](https://rasa.com/docs/core/tracker_stores/) ），观察用户和助手是怎么对话的。这有助于你更好的理解你的助手应该怎么处理问题，应该学习什么新的技能，更重要的是，能够获取更好的数据，来提高助手的能力。

现在你知道怎样生成训练数据，接下来来看下设计训练故事的最常见的实践方案。

## 使用slots设计故事

除了意图分类，实体提取，和对话状态管理，对话管理模型的预测还受到slots的影响。slots的结构是key-value的结构，用来存储对话过程中关键信息片段。使用slots，你可以知道什么时候助手需要询问必要的信息，什么时候有些问题可以避免询问。

举个例子，Sara的一个技能是订阅一个新的rasa用户到rasa newsletter。为了实现这个功能，rasa需要知道用户的email，因为没有email地址，rasa没法执行后续的action。因此，目标是需要教会助手，如果用户没有提供email地址，助手需要进行询问，如果用户提供了，那么助手需要跳过询问email的过程，到对话的下一个状态。这样的行为可以用slot实现 - 如果email slot没有赋值，助手进行询问，如果这个slot已经赋值了，助手就会继续执行后续的action。Rasa故事中的`slot{}`方法用来指明对话中slot 的值。下面是两个例子，用来明证不同阶段的`slot{}`事件如何影响对话。一开始用户没有提供email的故事如下：

```
## story_email_not provided
* greet
  - utter_greet
* subscribe_newsletter
  - utter_ask_email
* inform{email:'example@example.com'}
  - slot{email:'example@example.com'}
  - action_subscribe_newsletter
```

初始请求的时候用户提供了email，如下：

```
## story_email provided
* greet
  - utter_greet
* subscribe_newsletter{email:'example@example.com'}
  - slot{email:'example@example.com'}
  - action_subscribe_newsletter
```

如果你有个email的slot，你的NLU模型提取出了email名字的实体，那么slot会自动的用提取得到的实体的值进行填充。因此，这种情况下，你没有必要包含`slot{}`方法。如果你使用交互式学习来生成stroy，`slot{}`方法会自动添加到你的故事中，你没有必要担心这个。

当对依赖提取出来的详细信息的不同对话轮回进行建模的时候，slots是很重要的。你可以使用不同的slot类型，来决定他们如何影响下一个action的预测。比如，当slot类型是text，slot的有值和没有值才会影响AI助手下一步的行为。但是在其他情况下，你也许要用categorical或boolean类型的slots，就需要考虑slots的具体值，当然你也可能只是想用slots存储一些信息，而不想影响预测流程，可以使用unfeaturized类型。你可以在这里看到关于slots类型的更多的内容：[here](https://rasa.com/docs/core/slots/).

在一些情况下，自定义行为的详细信息的返回也会影响到对话。自定义action返回的slots可以提供外部世界的信息，和驱动对话到特定的方向。比如，在上面的例子中，助手的行为与用户当前是否已经subscribe newsletter有关。这种情况下，一个自定义行为可以检查洪湖是否已经订阅了newsletter，如果是，将对应的boolean类型的slot设置为True，反之设置为False。

```python
class ActionSubscribeNewsletter(Action):
    """ This action calls our newsletter API and subscribes the user with
    their email address"""

    def name(self):
        return "action_subscribe_newsletter"

    def run(self, dispatcher, tracker, domain):
        email = tracker.get_slot('email')
        if email:
            # if the email is already subscribed, this returns False
            subscribed = check_if_subscribed(email)

            return [SlotSet('subscribed', subscribed)]
        return []
```

这个行为也同样反映在训练故事中。这需要在自定义action之后，在你的故事中包含`slot{}`事件（如果使用interactive learning，这一步将自动为你做好）。下面的例子给出了slot的值如何影响对话流程。如果用户没有不是一个订阅者，助手会进行订阅，并将确认邮箱发送给用户，但是如果用户已经是一个订阅者，助手会发送消息说用户已经订阅了：

```
## newsletter + not subscribed
* greet
  - action_greet_user
* signup_newsletter
  - utter_ask_email
* inform{"email": "example@example.com"}
  - slot{"email": "example@example.com"}
  - action_subscribe_newsletter
  - slot{"subscribed":true}
  - utter_awesome
  - utter_confirmationemail
  - utter_ask_feedback
* deny
  - utter_thumbsup
  
## newsletter + already subscribed
* greet
  - action_greet_user
* signup_newsletter
  - utter_ask_email
* inform{"email": "example@example.com"}
  - slot{"email": "example@example.com"}
  - action_subscribe_newsletter
  - slot{"subscribed": false}
  - utter_already_subscribed
  - utter_ask_feedback
* deny
  - utter_thumbsup
```

如果AI助手仅需要收集一个或两个消息片段来执行特定的action，你可以使用slots，但是，通常情况下，你的助手需要收集很多信息。这种情况下，助手需要对很多slots进行填值，那么仅使用slots将会使得训练故事变得很复杂。针对这种情况，你将需要很多的训练故事来覆盖一个happy path。为了解决这个问题，以及避免故事变得过长，你应该使用Rasa Forms，进行填槽。

## Stories with Rasa Forms

[Rasa Forms](https://rasa.com/docs/core/slotfilling/)（[对话系统rasa - forms (翻译)](https://zhuanlan.zhihu.com/p/84441651)）是Rasa Feature中一个很重要的部分，它使得happy path的设计变得非常简单，同时使得填槽更加的可靠。比如，使用Sara的时候，RASA用户可以预订销售电话，以了解有关RASA企业功能的更多信息。在调度电话之前，Sara需要知道用户的更多的详细信息：用户的职位，使用场景，名字，公司等。想要仅使用slots 实现这样的对话模型，你需要考虑用户提供消息的不同方式（如在初始请求的时候提供所有的信息，仅提供不同的信息，还是在进行询问的时候才提供信息）。然而，使用Rasa Forms，这些场景可以用单个故事就能够表述清楚，并能够确保bot在获取到足够的信息的时候才会执行后续的过程。

Rasa Forms背后的主要思想是一旦form被激活，FormPolicy将会掌管对话管理，并使用form action来检查哪些slots还没有被填值。一旦所有的slots都设置了值，form会停止，formpolicy会将对话管理交给其他的policies（定义在配置文件中）。你可以在这里查看更多的关于配置forms action的内容：[here](https://rasa.com/docs/core/slotfilling/).

很显然，用户并不是以happy path的方式进行对话，通常会拒绝提供相关的信息，或插入其他不相关的对话。为了构建这些场景，你的故事需要包括中间的action。比如，下面的故事中的场景是用户决定要终止提供信息，但是后来又继续提供需要的信息：

```
## stop but continue path
* contact_sales
  - sales_form
  - form{"name": "sales_form"}
* stop
  - utter_ask_continue
* affirm 
  - sales_form
  - form{"name":null}
  - utter_slots_values
* thankyou
  - utter_noworries
```

由于带Rasa Forms的故事会变得很复杂，尤其是当对happy path的变种进行建模的时候，我们建议使用interactive learning来产生训练数据。

当开发你的上下文对话助手的时候，很有可能你不得不面对更严重的偏离快乐道路的情况。比如，很有可能你的用户完全背离了他之前的目标，开始进行所谓的chitchat，也就是说一些完全和目标或助手处理的领域完全不相关的内容。在下一个步骤，我们会介绍如何处理这种情况。

## 处理 ChitChat

chitchat是只用户讨论一些和bot能够处理的事情完全不相关的话题。一个例子就是问餐厅查找bot，哪个电影建议看，或者谁是美国的总统。然而，想要你的AI助手对于这些chitchat能够有一个很人性化的响应是很困难的，但是能够识别chitchat和尽可能友好的处理它是很重要的。那么，该怎么识别chitchat呢？最佳的方式是创建一个独立的意图，如，`chitchat`。然后用不同的输入进行训练。一旦你的NLU模型能够从其他的意图中分别出chitchat，你可以在你的故事中使用他们。

助手会如何响应chitchat，取决于你。一个简单的方法，是以通用的语句进行回复，如“Sorry, I can't help you with that”。一个例子如下：

```
## form + stop + come back
* greet
  - utter_greet
* contact_sales
  - sales_form
  - form{"name":"sales_form"}
* stop
  - utter_ask_continue
* affirm
  - sales_form
  - form{"name": null}
  - utter_slots_values
* thankyou
  - utter_noworries
```

由于chitchat可以出现在对话中的任何位置，使用常规的训练策略，你需要很多的训练示例来正确的处理它。为了解决这个问题，我们创建了一个新的policy，叫做MappingPolicy，使得处理chitchat和FAQ-like的对话变得更加简单。MappingPolicy的主要思想是，它允许你将一些意图映射到特定的actions，这些actions能够确保一旦特别的意图被识别出来（如，chitchat），AI助手总是能够用映射的action（如，utter_chitchat）进行响应，并且会忽略之前发生的对话。为了指定意图映射到那个actions，在你的domain文件中，需要添加`triggers`。具体如下：

```
intents:
- chitchat: {triggers: utter_chitchat}
```

对于chitchat的识别和响应仅仅是基于目标的助手的一部分的工作。让助手能够接管对话，把用户带到之前的路径上也是相当重要的。为了实现这一点，你的故事不单单需要包括对chitchat的响应，还需要提醒用户继续之前的对话。下面的故事是一个很好的例子 - 一旦用户开始chitchat，助手会回复他，并且将之前的问题发给用户：

```
## form + chitchat
* contact_sales
  - utter_ask_jobfunction
* chitchat
  - utter_chitchat
  - utter_ask_jobfunction
* enter_data{"jobfunction":"product manager"}
  - utter_ask_usecase
* chitchat
  - utter_chitchat
  - utter_ask-usecase
```

为了对这样的对话进行建模，你可以尝试Rasa的[EmbeddingPolicy](https://rasa.com/docs/core/policies/#embedding-policy) ，它专门被设计用来处理[uncooperative user behaviour](https://arxiv.org/abs/1811.11707)，因此对于看不到的对话更容易概括。

## 需要多少的训练故事

这是一个很常见的也是一个很重要的问题。然而并没有一个明确的规则，几十个训练故事足够启动开发。使用更多的训练故事，你的AI助手可以学习的更好，能够更合理的处理复杂的背离happy path的情况。需要记住的是，为了能够实现上面的结果，训练数据中的故事必须覆盖不同的对话场景。为了构建产品级别的助手，你至少需要几百个训练故事（这个依赖于你想要机器处理的领域和对话的复杂度）。随着训练数据的增长，为了更好的debugging，你会考虑将训练数据分成不同的文件。如果你这么做，那么你需要将带训练数据的文件夹传递给训练函数，那么所有文件夹内的文件都会被组成为一个大文件。下面是一个文件夹结构的示例：

![](./image/rasa_design_train_data.png)

## 小结

训练数据对于建立一个成功的人工智能助手至关重要。你需要多少和什么样的故事取决于你的用例，所以记住你开发助手的用户是什么是很重要的。我们一直在寻找使会话数据的创建更容易和更有效的方法。我们很想听听你在为RASA核心模型生成训练故事方面的经验。与我们分享它通过加入Rasa Community Forum讨论！

## 资源

- [Rasa documentation](https://rasa.com/docs/core/)
- [Sara demo bot](https://github.com/RasaHQ/rasa-demo)
- [Rasa and Botsociety integration](https://www.youtube.com/watch?v=AF5O5zotXoQ)
- [Rasa Community Forum](https://forum.rasa.com/)

