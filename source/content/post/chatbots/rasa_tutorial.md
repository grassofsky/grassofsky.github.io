+++
date="2019-12-21"
title="对话系统Rasa-入门教程 [翻译]"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统Rasa-入门教程 [翻译]

## 基本流程

1. 安装rasa
2. 创建新项目
3. 查看NLU（自然语言理解）训练数据
4. 定义模型的配置
5. 写第一个stories（一个story，指的是用户和机器的一次对话）
6. 定义领域
7. 训练模型
8. 与助手进行对话

## 1. 安装rasa

`pip install rasa-x --extra-index-url https://pypi.rasa.com/simple`

## 2. 创建新项目

`rasa init --on-prompt`

该命令对应的输出如下：

```
Welcome to Rasa! 🤖

To get started quickly, an initial project will be created.
If you need some help, check out the documentation at https://rasa.com/docs/rasa.

Created project directory at '/home/jovyan'.
Finished creating project structure.
Training an initial model...
Training Core model...
Core model training completed.
Training NLU model...
NLU model training completed.
Your Rasa model is trained and saved at '/home/jovyan/models/20190917-070346.tar.gz'.
If you want to speak to the assistant, run 'rasa shell' at any time inside the project directory.
```

该命令创建Rasa项目需要的文件，以及训练一个简单对话机器人需要的数据。如果命令不使用`--no-prompt`标识，那么在创建过程中需要回答几个关于项目创建的问题。

创建的文件如下，其中`*`标记的为最重要的部分：

- `__init__.py`，用来帮助Python找到action的空文件
- `actions.py`，用来自定义actions的文件
- `config.yml *`，NLU和核心模型的配置文件
- `credentials.yml`，用来连接到其他服务
- `data/nlu.md *`，用于NLU训练的数据
- `data/stories.md *`，故事集
- `domain.yml *`，助手领域（assistant's domain）
- `endpoints.yml`，连接到类似于fb的通信对象的详细配置
- `models/<timestamp>.tar.gz`，初始化模型

## 3. 查看NLU（自然语言理解）训练数据

Rasa助手的第一块内容是NLU模型。NLU是自然语言理解，用来将用户消息转换成结构数据。在Rasa中，可以通过提供训练示例，告诉Rasa如何理解用户消息，然后训练获得模型。可以看一下`nlu.md`的文件内容：

```
## intent:greet
- hey
- hello
- hi
- good morning
- good evening
- hey there

## intent:goodbye
- bye
- goodbye
- see you around
- see you later

## intent:affirm
- yes
- indeed
- of course
- that sounds good
- correct

## intent:deny
- no
- never
- I don't think so
- don't like that
- no way
- not really

## intent:mood_great
- perfect
- very good
- great
- amazing
- wonderful
- I am feeling very good
- I am great
- I'm good

## intent:mood_unhappy
- sad
- very sad
- unhappy
- bad
- very bad
- awful
- terrible
- not very good
- extremely sad
- so sad
```

`##`开始的行定义了你的意图，是具有相同含义消息的集合。Rasa的任务是，预测用户输入的消息的正确意图。具体数据格式可以参见： [Training Data Format](https://rasa.com/docs/rasa/nlu/training-data-format/#training-data-format)。

## 4. 定义模型的配置

该配置文件，你的模型将会使用到的NLU和核心部分。这个例子中，NLU模型会使用`supervised_embeddings`流水线。不同NUL模型的流水线可以在[这里](https://rasa.com/docs/rasa/nlu/choosing-a-pipeline/#choosing-a-pipeline)查看。`config.ml`文件的内容为：

```
# Configuration for Rasa NLU.
# https://rasa.com/docs/rasa/nlu/components/
language: en
pipeline: supervised_embeddings

# Configuration for Rasa Core.
# https://rasa.com/docs/rasa/core/policies/
policies:
  - name: MemoizationPolicy
  - name: KerasPolicy
  - name: MappingPolicy
```

其中language和pipeline两个关键词，用来定义NLU模型应该如何构建。`policies`关键词定义核心模型中使用到的[policies](https://rasa.com/docs/rasa/core/policies/#policies)。

## 5. 写第一个stories

在这个阶段，你将教助手如何响应你的消息。这个叫做对话管理，受你的核心模型处理。

核心模块以训练stories的形式从真实的对话中进行学习。一个story是用户和助手之间真实的一次对话。意图和实体的行反应了用户的输入和助手将响应的行为名。

下面是一个简单的对话。用户说hello，助手回复hello。如下：

```
## story1
* greet
	- utter_greet
```

关于story的详细介绍可以参见：[Stories](https://rasa.com/docs/rasa/core/stories/#stories).

`-`开头的行表示助手采取的措施。在这个教程中，所有消息的响应都是发给用户的，如`utter_greet`，但是通常情况下，一个响应行为可以处理任何事情，包括调用API，以及和外面的世界的交互。下面给出了`data/stories.md`中的内容。

```
## happy path
* greet
  - utter_greet
* mood_great
  - utter_happy

## sad path 1
* greet
  - utter_greet
* mood_unhappy
  - utter_cheer_up
  - utter_did_that_help
* affirm
  - utter_happy

## sad path 2
* greet
  - utter_greet
* mood_unhappy
  - utter_cheer_up
  - utter_did_that_help
* deny
  - utter_goodbye

## say goodbye
* goodbye
  - utter_goodbye
```

## 6. 定义领域

下一个事情需要处理的是定义一个领域（ [Domain](https://rasa.com/docs/rasa/core/domains/#domains).）这个领域定义了你的助手所处的世界：期望从用户那边获取什么输入，他能够处理什么预测行为，怎么做相应，存储什么消息。领域相关的内容存储在`domain.yml`文件中。

```
intents:
  - greet
  - goodbye
  - affirm
  - deny
  - mood_great
  - mood_unhappy

actions:
- utter_greet
- utter_cheer_up
- utter_did_that_help
- utter_happy
- utter_goodbye

templates:
  utter_greet:
  - text: "Hey! How are you?"

  utter_cheer_up:
  - text: "Here is something to cheer you up:"
    image: "https://i.imgur.com/nGF1K8f.jpg"

  utter_did_that_help:
  - text: "Did that help you?"

  utter_happy:
  - text: "Great carry on!"

  utter_goodbye:
  - text: "Bye"
```

intents，定义了期望用户的输入；actions：定义了助手能够说什么做什么；templates：定义了助手相应的消息模板。

这些是怎么一起协调工作的呢？Rasa核心的任务是针对每个对话选择合正确的响应。在这个例子中，我们的响应是简单的发送消息给用户，这里面的响应都用`utter_`开头。助手根据模板进行响应。可以参见 [Custom Actions](https://rasa.com/docs/rasa/core/actions/#custom-actions) 创建更复杂的响应。

## 7. 训练模型

一旦我们添加了新的NLU或核心数据，或更新领域或配置文件的时候，我们需要重新训练模型。执行下面的命令`rasa train`。这个命令会调用Rasa核心和NLU训练函数，并将训练得到的模型存储到`models/`目录下面。这个命令会自动重新训练更新的部分。

## 8. 与助手进行对话

恭喜，到目前为止你建立了一个基于机器学习的对话助手。

下一步通过执行下面的命令和助手进行对话：`rasa shell`。

## 原文链接

https://rasa.com/docs/rasa/user-guide/rasa-tutorial/





