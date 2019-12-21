+++
date="2019-12-21"
title="对话系统Rasa - 对话设计 [翻译]"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统Rasa - 对话设计 [翻译]

## 对话元素

对话元素是常见的会话元素。我们使用三个不同的抽象层次来讨论人工智能助手。这对产品团队很有帮助，这样就有了一种通用语言，设计师、开发人员和产品所有者可以使用它来讨论问题和新特性。

- 最高等级：用户目标
- 中间等级：对话元素
- 最低等级：意图，实体，动作，插槽，和模板（intents, entities, actions, slots, and templates）

> 注意：
>
> 一些聊天机器人工具会使用intent描述用户目标。这是令人困惑的，因为只有一些消息才会告诉你用户的目标是什么。如果一个用户说“I want to open an account”（intent: open_account），那么很显然目标也是这个。但是，很多用户消息，如("yes", "what does that mean?", "I don't know") 并没有明确的目标。在Rasa，每一个消息都有一个intent，而用户的目标描述的是一个人想要获得什么。

![](https://rasa.com/docs/rasa/_images/intents-user-goals-dialogue-elements.png)

## 闲聊

闲聊指的是一来一回没有明确目标的谈话。包括问候，感谢，回应和话题外的闲聊。

### 问候

问候和再见是最简单的交流。每个对话系统都会遇到它们：

```
user> hello
bot> hi, how are you?

user> how are you?
bot> I am well, and you?

user> goodbye
bot> bye bye!
```

为了能够正确的针对问候和再见提供回复，需要针对它们分别定义回复语。如果你总是想要一样的回复，你可以使用`MappingPolicy`来触发这些回复。

在你的domain文件中，添加trigger元数据，到相应的intents上：

```
intents:
	- greet: {triggers: utter_greet}
	- goodbye: {triggers: utter_goodbye}
```

并且确认mapping policy出现在`config.yml`文件中。

```
policies:
	- name: "MappingPolicy"
	...
```

如果你想要实现更少限制的行为，可以使用常规的stories，而不用mapping policy。如下：

```
* greet
	- utter_greet
* goodbye
	- utter_ask_why_leaving
```

### 确认

用户通常会对助手的话做出反应，并期望得到确认。确认的表述可以简单到只竖起一个大拇指。助手需要向用户保证他们收到了消息。可以针对致谢实现特定的反馈。

```
user> woah that's expensive!
bot> we offer good value.
bot> would you like to continue getting a quote.

user> that's awesome!
bot> glad you think so :)
```

首先，需要NLU数据定义对应的响应和确认信息：

```
## intent:acknowledge
- ok
- got it
- understood
- k

## intent:opinion+positive
- nice!
- excellent
- that's awesome

## intent:opinion+negative
- ugh
- that sucks
- woah! that's [expensive](price)
```

那么在创建故事的时候，如下：

```
## price reaction
* opinion+negative{"price": "expensive"}
  - utter_good_value
  - utter_ask_continue
  
## simple acknowledgement
* opinion+positive
  - utter_positive_feedback_reaction
```

### 题外话的闲聊

你的助手通常会接收到意料之外的输入。我们称为题外话的闲聊。(chitchat)。对用户的每个输入进行合理的顺畅的响应是不可能办到的，但是可以通过简单的确认，让用户知道我们已经接收到了消息。一个策略是从用户中收集训练数据，并定义意图和响应。参见 [Explaining Possibilities](https://rasa.com/docs/rasa/dialogue-elements/guiding-users/#explaining-possibilities)，解释如何处理意料之外的输入。

```
user> will you marry me?
bot> no

user> aahhh
bot> I feel you

user> I can juggle 7 oranges
bot> wow!
```

### 辱骂

不幸的是，用户通常会辱骂你的助手。你需要承认他们的评价，并以一种反映助手个人形象的方式作出回应。如果用笑话进行回应，那么会鼓励用户继续辱骂的言语，因此要仔细考虑你的回应。你在这篇论文中（ [this paper](https://www.aclweb.org/anthology/W18-0802)）可以查看更多相关内容。

```
user> stupid bot
bot> that's not very nice
```

针对辱骂处理的最简单的方式是建立insult意图，并利用mapping policy进行响应。如下domain文件中：

```
intents:
  - insult: {triggers: utter_respond_insult}
```

在配置文件中：

```
policies:
  - name: "MappingPolicy"
  ...
```

## 完成任务

### 简单问题

简单问题，或FAQs，不管之前的对话内容是什么，应该接收到相同的答案。用户会问一些基本的问题，而你的助手应该可靠的回答这些问题。

```
user> what's your email address?
bot> it's contact@example.com

user> do you have a loyalty program?
bot> unfortunately we don't
```

与问候类似，可以使用mapping policy实现。

### 业务逻辑

注意：这里有详细的tutorial，介绍如何使用Rasa Forms，实现slot filling和业务逻辑。[here](https://blog.rasa.com/building-contextual-assistants-with-rasa-formaction/?_ga=2.113508802.416131191.1568702367-1750168845.1568702367) 

你的AI助手通常需要按照一些预定义的业务逻辑执行。为了指出如何帮助用户，通常你的助手会询问一些问题。获取得到的答案会影响到之后的会话。举个例子，一些产品只适用于特定区域或特定年龄段的用户。将相关逻辑实现到form内部，与学习的行为相分离是一种比较好的实现方式。一个独立的form可以覆盖所有的happy路径。（例如，用户需要提供所有需要的信息）。更多的内容可以参见：[this tutorial](https://blog.rasa.com/building-contextual-assistants-with-rasa-formaction/?_ga=2.40452711.416131191.1568702367-1750168845.1568702367).

```
user> I'd like to apply for a loan
bot> I'd love to help. Which state are you in?
user> Alaska
bot> Unfortunately, we only operate in the continental U.S.

user> I'd like to apply for a loan
bot> I'd love to help. Which state are you in?
user> California
bot> Thanks. Do you know what your credit score is?
```

关于如何利用forms实现业务逻辑，可以参见：[Handling conditional slot logic](https://rasa.com/docs/rasa/core/forms/#conditional-logic) 

### 语境问题

不像针对FAQs的回答，针对语境问题的正确回复需要依赖于对话历史。这些问题指的是谈话中早些时候的一些问题，而这些问题本身就是模棱两可的。真正的用户经常会问“哪个更好？”“为什么？“。如果你的助手不理解这一点，并且只能回答诸如“哪一个储蓄账户的利率更高”这样的完整问题，这会让用户感到沮丧。“理解语境问题是二级和三级助理之间的一个关键区别（ [level 2 and level 3 assistants](https://www.oreilly.com/ideas/the-next-generation-of-ai-assistants-in-enterprise)）。

```
bot> what's your email address?
user> why do you need to know that?
bot> I need your email so I can send you a confirmation

bot> are you currently a premium customer?
user> what do you mean?
bot> We have different memberships. Check your statement to see if you are a premium member.
```

### 不愉快的路径

当你的助手向用户询问想要的信息的时候，你通常会获取不是你想要的回复。举个例子，你的用户可能拒绝提供信息，他们也许纠正之前说的东西，或者用闲聊进行打断。你的助手能够处理这些边界问题是很重要的。除了你提问的你想要的信息，用户可以输入任何的话语，甚至简单的完全不管你的助手进行对话中断。这个从真实数据学习构建助手的关键原因。

针对不愉快的路径收集训练数据最好的方式是使用[Interactive Learning](https://rasa.com/docs/rasa/core/interactive-learning/#interactive-learning).

```
bot> what's your email address?
user> no.
bot> I will need your email address in order to create an account.
user> what's your email address?

bot> what's your email address?
user> work@example.com
bot> thanks, and your phone number?
user> no wait, please use personal@example.com
bot> ok, I'll use that email.
bot> thanks, and your phone number?
```

## 引导用户

### 隐式确认

隐式确认包括将理解的信息再次反馈给用户，让用户进行确认。这个也是让用户有个机会判断助手在理解上是不是有问题。

```
user> Can I get a large hawaiian and bbq pizza
bot> Sure, that's one large hawaiian and one regular bbq pizza
bot> Anything else?
user> No they should both be large
```

### 显式确认

显示确认意思是让用户明白你应该如何帮助他们。关于智能助手，要记住的一件重要事情是用户永远不会出错。当一个用户告诉你“I just moved”，即使你的助手不能够明确如何帮助他们，他们的表述也是很清楚的。如果你的助手没法明确的识别出用户的目标，可以直接要求澄清。

```
user> I just moved
bot> I'm not sure I understood you correctly. Do you mean ...
user> I want to cancel my contract
user> I want to update my personal details
```

你可以配置`TwoStageFallbackPolicy`，要求用户进行澄清，并将最可能的意图快速提供给用户。为了实现这点，需要配置policy，如下：

```
policies:
- name: TwoStageFallbackPolicy
  nlu_threshold: 0.3
  core_threshold: 0.3
  fallback_core_action_name: "action_default_fallback"
  fallback_nlu_action_name: "action_default_fallback"
  deny_suggestion_intent_name: "out_of_scope"
...
```

### 解释可能性

智能助手总是局限于帮助用户完成一组特定的任务，并且应该能够向用户解释他们能做什么。这包括对于超出范围的问题给出一致的响应。

```
user> What can you do?
bot> I can help you update your personal details, change your plan, and answer any questions you have about our products.

user> Can you get me a pizza?
bot> I'm afraid I can't help with that.
bot> I can help you update your personal details, change your plan, and answer any questions you have about our products.
```

何时对助手的适用性给出解释，需要在程序中进行设定，如：

```
## user asks whats possible
* ask_whatspossible
  - utter_explain_whatspossible

## user asks for something out of scope
* out_of_scope
  - utter_cannot_help
  - utter_explain_whatspossible
```

### 收集用户反馈

想要知道你的回复是不是解决了用户的问题，询问反馈是一种很好的方式。将这些反馈信息进行存储，有助于提高助手的可用性。

```
bot> Was that helpful?
user> no.
bot> Thanks. Why wasn't I able to help?
user> you didn't understand me correctly
user> you understood me, but your answers weren't very helpful.
```

可以利用表格收集用户反馈。为了实现这个，需要自定义表格行为，详细见：[Forms](https://rasa.com/docs/rasa/core/forms/#forms) 。

```python
from rasa_sdk.action import FormAction

class FeedbackForm(FormAction):
    def name(self):
        return "feedback_form"
    
    @staticmethod
    def required_slots(tracker):
        return ["feedback", "negative_feedback_reason"]
```

在domain中的定义如下：

```
forms:
  - feedback_form
slots:
  feedback:
    type: bool
  feedback_reason:
    type: text
  requested_slot:
    type: text
```

还要确保将`FormPolicy`添加到configuration文件：

```
policies:
  - FormPolicy
  ...
```

### 交给人

当助手不能够帮助到用户，并且不能将对话交接给人工代理，用户会变得非常沮丧。总有一种方法可以让对话中断，并且有多重原因触发人工切换：

- 用户要求与人对话
- 助手陷入到理解用户的泥潭中
- 助手理解用户想要什么，但是需要人工的参与

```
user> let me speak to a human
bot> let me put you in touch with someone

user> I want to cancel
bot> I'm afraid I can't help you with that.
bot> let me put you in touch with someone.
```

针对直接要求和人进行对话的实现可以使用mapping policy：

```
intents:
  - request_human: {"triggers": "action_human_handoff"}
```

## 原文链接

- https://rasa.com/docs/rasa/dialogue-elements/dialogue-elements/
- https://rasa.com/docs/rasa/dialogue-elements/small-talk/
- https://rasa.com/docs/rasa/dialogue-elements/completing-tasks/
- https://rasa.com/docs/rasa/dialogue-elements/guiding-users/#