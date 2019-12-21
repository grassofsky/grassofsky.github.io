+++
date="2019-12-21"
title="对话系统rasa - 校验数据"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统rasa - 校验数据

## 测试域和数据文件是否有错误

要验证域文件、nlu数据或故事数据中是否有任何错误，请运行验证脚本。可以使用以下命令运行它：

```
rasa data validate
```

上面的脚本对您的文件运行所有验证。以下是脚本的选项列表：

```
usage: rasa data validate [-h] [-v] [-vv] [--quiet] [--fail-on-warnings]
                          [-d DOMAIN] [--data DATA]

optional arguments:
  -h, --help            show this help message and exit
  --fail-on-warnings    Fail validation on warnings and errors. If omitted
                        only errors will result in a non zero exit code.
                        (default: False)
  -d DOMAIN, --domain DOMAIN
                        Domain specification (yml file). (default: domain.yml)
  --data DATA           Path to the file or directory containing Rasa data.
                        (default: data)

Python Logging Options:
  -v, --verbose         Be verbose. Sets logging level to INFO. (default:
                        None)
  -vv, --debug          Print lots of debugging statements. Sets logging level
                        to DEBUG. (default: None)
  --quiet               Be quiet! Sets logging level to WARNING. (default:
                        None)
```

您还可以通过python api导入validator类来运行这些验证，该类具有以下方法：

**from_files()**：从特定的文件创建实例

**verify_intents()**：检查域文件中列出的意图是否与NLU数据一致。

**verify_example_repetition_in_intents()**：检查NLU数据的不同意图之间是否没有重复数据。

**verify_intents_in_stories()**：验证故事中的意图，以检查它们是否有效。

**verify_utterances(): **检查域文件中的语句模板与“操作”下列出的语句之间的一致性。

**verify_utterances_in_stories():** 检查故事中对话的有效性。

**verify_all()**：运行上面所有的检查。

为了使用这些函数，有必要创建Validator对象，初始化logger。如下：

```python
import logging
from rasa import utils
from rasa.core.validator import Validator

logger = logging.getLogger(__name__)

utils.configure_colored_logging('DEBUG')

validator = Validator.from_files(domain_file='domain.yml',
                                 nlu_data='data/nlu_data.md',
                                 stories='data/stories.md')

validator.verify_all()
```

