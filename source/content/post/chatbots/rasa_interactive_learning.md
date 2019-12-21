+++
date="2019-12-21"
title="Rasa - Interactive Learning （翻译）"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# Rasa - Interactive Learning （翻译）

这篇文档会介绍如何使用命令行使用interactive learning（互动学习）。

在互动学习模式，当你和机器进行对话的时候，你为机器提供了反馈。这对于探索你的机器能够做什么是很有帮助的，并且这也是发现它犯错的最简单的方式。基于机器学习的对话系统的一个优点是当机器不知道怎么做的时候，你可以教它。

**注意**：Rasa X提供了UI界面用于交互式学习，见：[Annotate Conversations](https://rasa.com/docs/rasa-x/annotate-conversations/) 。

## Running Interactive Learning

执行下面的命令开始交互式学习：

```
rasa run actions --actions actions&
rasa interactive \
  -m models/20190515-135859.tar.gz \
  --endpoints endpoints.yml
```

第一个命令启动action服务。

第二个命令开启交互式学习模式。

在交互式学习模式中，Rasa针对每一个NLU预测的结果向你进行确认，然后才能用于Core。下面是一个例子：

```
Bot loaded. Type a message and press enter (use '/stop' to exit).

? Next user input:  hello

? Is the NLU classification for 'hello' with intent 'hello' correct?  Yes

------
Chat History

 #    Bot                        You
────────────────────────────────────────────
 1    action_listen
────────────────────────────────────────────
 2                                    hello
                         intent: hello 1.00
------

? The bot wants to run 'utter_greet', correct?  (Y/n)
```

对话历史和slot的值会被打印到屏幕上，这些是你需要确定下一个行为是否正确需要的所有的信息。

在这个例子中，bot选择了正确的行为（`utter_greet`），因此你输入`y`。然后我们再次输入`y`，因为greeting之后的`action_listen`行为是正确的。我们继续这个循环，和机器进行对话，知道机器选择了一个错误的行为。

## Providing feedback on errors

这里我们将使用`concertbot`例子，可以从此处下载： [github repo](https://github.com/RasaHQ/rasa/tree/master/examples/concertbot).

如果你问`/search_concerts`，机器会建议`action_search_concerts`，然后`action_listen`。这个时候输入`/compare_reviews`。bot可能会选择错误的结果（依赖于训练结果，此处也可能是正确的）：

```
------
Chat History

 #    Bot                                           You
───────────────────────────────────────────────────────────────
 1    action_listen
───────────────────────────────────────────────────────────────
 2                                            /search_concerts
                                  intent: search_concerts 1.00
───────────────────────────────────────────────────────────────
 3    action_search_concerts 0.72
      action_listen 0.78
───────────────────────────────────────────────────────────────
 4                                            /compare_reviews
                                  intent: compare_reviews 1.00


Current slots:
  concerts: None, venues: None

------
? The bot wants to run 'action_show_concert_reviews', correct?  No
```

因为它选择了一个错误的行为，现在我们输入`n，`由于它选择了错误的行为，我们会得到新的提示，用于询问哪个是正确的action。新的提示中同时给出了模型预测的时候得到的不同action的可能性：

```
? What is the next action of the bot?  (Use arrow keys)
 ❯ 0.53 action_show_venue_reviews
   0.46 action_show_concert_reviews
   0.00 utter_goodbye
   0.00 action_search_concerts
   0.00 utter_greet
   0.00 action_search_venues
   0.00 action_listen
   0.00 utter_youarewelcome
   0.00 utter_default
   0.00 action_default_fallback
   0.00 action_restart
```

在这个例子中，bot想要执行的是`action_show_concert_reviews`，而不是venue。因此我们选择第二个action。

现在，如果我们需要创建一个长对话，我们可以继续和机器进行对话。在任意时刻，都可以通过按`Ctrl-C`，告诉bot退出。你可以将新创建的故事和NLU数据输出到文件。你也可以回退到出问题的一步。

在下一次训练的时候，一定要将导出的故事和NLU示例与原值的训练数据结合。

## Visualization of conversations

在交互训练的过程中，rasa可以绘制当前的会话，和从训练数据中得到的类似的会话，帮助你对你的行为进行追踪。

当交互是会话进行后，你可以在http://localhost:5005/visualization.html进行查看。

如果不需要可视化，可以运行`rasa interactive --skip-visualization`。

![](./image/interactive_learning_graph.gif)

## Interactive Learning with Forms

如果你使用FormAction，那么在使用交互式学习的时候，还需要注意一些其他的事情。

### The form: prefix

form逻辑是咋FormAction类中定义的，而不是故事中定义。机器学习策略不需要学习这个行为。并且，如果你在之后改变了你的form action，如添加或删除一些必要的slot，bot也不应该感到困惑。。当你使用交互学习产生包含form的故事ide时候，被form处理的对话步骤包含`form:`前缀。这个可以告诉Rasa Core在训练你的其他策略的时候忽略这些步骤。这里没有其他特别需要处理的。

下面是一个例子：

```
* request_restaurant
    - restaurant_form
    - form{"name": "restaurant_form"}
    - slot{"requested_slot": "cuisine"}
* form: inform{"cuisine": "mexican"}
    - slot{"cuisine": "mexican"}
    - form: restaurant_form
    - slot{"cuisine": "mexican"}
    - slot{"requested_slot": "num_people"}
* form: inform{"number": "2"}
    - form: restaurant_form
    - slot{"num_people": "2"}
    - form{"name": null}
    - slot{"requested_slot": null}
    - utter_slots_values
```

### Input validation

每一次用户用所请求的slot或任何一个需要的slot进行一个响应的时候，都会询问你是否希望form action尝试并在返回表单时从用户的消息中提取一个slot。用例子进行解释：

```
 7    restaurant_form 1.00
      slot{"num_people": "3"}
      slot{"requested_slot": "outdoor_seating"}
      do you want to sit outside?
      action_listen 1.00
─────────────────────────────────────────────────────────────────────────────────────
 8                                                                             /stop
                                                                   intent: stop 1.00
─────────────────────────────────────────────────────────────────────────────────────
 9    utter_ask_continue 1.00
      do you want to continue?
      action_listen 1.00
─────────────────────────────────────────────────────────────────────────────────────
 10                                                                          /affirm
                                                                 intent: affirm 1.00


Current slots:
    cuisine: greek, feedback: None, num_people: 3, outdoor_seating: None,
  preferences: None, requested_slot: outdoor_seating

------
2018-11-05 21:36:53 DEBUG    rasa.core.tracker_store  - Recreating tracker for id 'default'
? The bot wants to run 'restaurant_form', correct?  Yes
2018-11-05 21:37:08 DEBUG    rasa.core.tracker_store  - Recreating tracker for id 'default'
? Should 'restaurant_form' validate user input to fill the slot 'outdoor_seating'?  (Y/n)
```

这里用户要求停止form，bot回复用户是否不需要继续了。用户说他们想要继续。这里，`outdoor_seating`有一个`form_intent` slot mapping，将`/affirm`映射到`True`。因此，用户输入会被填充到那个slot。但是，这个例子中，用户仅仅对`do you want to continue?`这个问题进行回复，因此你选择`n`，用户输入将不会被校验。机器将继续询问`outdoor_seating` slot。