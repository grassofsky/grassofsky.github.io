+++
date="2019-12-21"
title="对话系统rasa - Training Data Importers（翻译）"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统rasa - Training Data Importers（翻译）

默认情况下，可以使用命令行参数指定rasa在磁盘上查找训练数据的位置。然后，rasa加载任何可能的训练文件并使用它们来训练您的助手。

如果有必要，你可以自定义Rasa如何导入训练数据。潜在的使用场景有：

- 使用自定义parser加载其他格式的训练数据
- 使用不同的途径收集训练数据（如，从不同的资源加载数据）

你可以让Rasa加载和使用你自定义的importer，通过在Rasa配置文件中添加importers，如下：

```
importers:
- name: "module.CustomImporter"
  parameter1: "value"
  parameter2: "value2"
- name: "module.AnotherCustomImporter"
```

`name`用来说明那个importer需要被加载。任何其他额外的参数以构造函数参数的形式传入。

**注意**：你可以指定多个importers。Rasa会自动将结果合并。

## RasaFileImporter(default)

默认情况下Rasa使用RasaFileImporter。如果您想单独使用它，则不必在配置文件中指定任何内容。如果要与其他导入程序一起使用，请将其添加到配置文件中：

```
importers:
- name: "RasaFileImporter"
```

## MultiProjectImporter(experimental)

**警告**：此功能目前处于试验阶段，将来可能会更改或删除。请在论坛上分享您对该功能的反馈，以帮助我们将此功能准备就绪。

使用该importer，你可以通过组合多个可重用的rasa项目来构建上下文ai助手。例如，你可以处理一个项目的闲聊，并用另一个项目问候。这些项目可以单独开发，然后在训练你的时候合并以创建你的助手。

一个目录结构的示例如下：

```
.
├── config.yml
└── projects
    ├── GreetBot
    │   ├── data
    │   │   ├── nlu.md
    │   │   └── stories.md
    │   └── domain.yml
    └── ChitchatBot
        ├── config.yml
        ├── data
        │   ├── nlu.md
        │   └── stories.md
        └── domain.yml
```

在本例中，上下文ai助手导入chitchatbot项目，然后导入greetbot项目。项目导入在每个项目的配置文件中定义。要指示rasa使用multiprojectimporter模块，请将此部分放在根项目的配置文件中：

```
importers:
- name: MultiProjectImporter
```

然后指定要导入的项目。在我们的示例中，根项目中的config.yml如下所示：

```
imports:
- projects/ChitchatBot
```

ChitChatBot的配置文件中要引用GreeBot，如下：

```
imports:
- ../GreetBot
```

GreetBot项目没有指定更多的项目，因此config.yml可以忽略。

rasa使用引用配置文件中的相对路径导入项目。只要允许文件访问，这些文件可以位于文件系统中的任何位置。

在训练过程中， rasa将导入所有需要的训练文件，将它们合并，并训练一名统一的ai助理。在运行时合并训练数据，因此不会创建或显示包含训练数据的其他文件。

**注意**：在训练过程中，rasa使用根项目目录的policy和NLU pipeline配置文件。导入项目中policy和NLU配置文件会被忽略。

**注意**：相同的意图，实体，slots，模板，actions和forms将被合并，如，如果两个项目中都有意图为greet的训练数据，他们的训练数据会被合并。

## Writing a Custom Importer

如果你要创建自定义importer，需要实现[TrainingDataImporter](https://rasa.com/docs/rasa/api/training-data-importers/#training-data-importers-trainingfileimporter)类一些接口，如下：

```python
from typing import Optional, Text, Dict, List, Union

import rasa
from rasa.core.domain import Domain
from rasa.core.interpreter import RegexInterpreter, NaturalLanguageInterpreter
from rasa.core.training.structures import StoryGraph
from rasa.importers.importer import TrainingDataImporter
from rasa.nlu.training_data import TrainingData

class MyImporter(TrainingDataImporter):
    """Example implementation of a custom importer component."""

    def __init__(
        self,
        config_file: Optional[Text] = None,
        domain_path: Optional[Text] = None,
        training_data_paths: Optional[Union[List[Text], Text]] = None,
        **kwargs: Dict
    ):
        """Constructor of your custom file importer.

        Args:
            config_file: Path to configuration file from command line arguments.
            domain_path: Path to domain file from command line arguments.
            training_data_paths: Path to training files from command line arguments.
            **kwargs: Extra parameters passed through configuration in configuration file.
        """

        pass

    async def get_domain(self) -> Domain:
        path_to_domain_file = self._custom_get_domain_file()
        return Domain.load(path_to_domain_file)

    def _custom_get_domain_file(self) -> Text:
        pass

    async def get_stories(
        self,
        interpreter: "NaturalLanguageInterpreter" = RegexInterpreter(),
        template_variables: Optional[Dict] = None,
        use_e2e: bool = False,
        exclusion_percentage: Optional[int] = None,
    ) -> StoryGraph:
        from rasa.core.training.dsl import StoryFileReader

        path_to_stories = self._custom_get_story_file()
        return await StoryFileReader.read_from_file(path_to_stories, await self.get_domain())

    def _custom_get_story_file(self) -> Text:
        pass

    async def get_config(self) -> Dict:
        path_to_config = self._custom_get_config_file()
        return rasa.utils.io.read_config_file(path_to_config)

    def _custom_get_config_file(self) -> Text:
        pass

    async def get_nlu_data(self, language: Optional[Text] = "en") -> TrainingData:
        from rasa.nlu.training_data import loading

        path_to_nlu_file = self._custom_get_nlu_file()
        return loading.load_data(path_to_nlu_file)

    def _custom_get_nlu_file(self) -> Text:
        pass
```

### TrainingDataImporter

#### class rasa.importers.impoter.TrainingDataImporter

不同机制加载数据的通用接口。

#### get_domain()

获取bot的domain

**Returns:** 加载的Domain

**Return type**: Domain

#### get_config()

获取被用于训练的将配置

**Returns:** 字典形式存储的配置

**Return type**:`Dict`[~KT, ~VT]

#### get_nlu_data(language='en')

获取被用于训练的NLU训练数据

**参数**：language，可以用来只加载某种语言的训练数据

**Returns:** 加载的NLU `TrainingData`

**Return type**:`TrainingData`

#### `get_stories`(*interpreter=<rasa.core.interpreter.RegexInterpreter object>*, *template_variables=None*, *use_e2e=False*, *exclusion_percentage=None*)

获取被用于训练的故事

**参数**

- interpreter - 应该用来解析端到端学习注释的解释器。
- template_variables - 读取故事文件的时候需要被替换掉的模板值
- use_e2e - 指定是否分析端到端学习批注。
- exclusion_percentage - 训练数据中需要被排除的比例

**Returns:** 包含所有故事的`StoryGraph`

**Return type**:`StoryGraph`

## 原文链接

https://rasa.com/docs/rasa/api/training-data-importers/