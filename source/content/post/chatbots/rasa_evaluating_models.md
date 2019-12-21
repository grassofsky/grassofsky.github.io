+++
date="2019-12-21"
title="对话系统rasa - 评估模型"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统rasa - 评估模型

如果你想要NLU模型的超参数调整，请参考：[rasa blog - 深入理解rasa NLU: Part3 - 超参数调整 (翻译)](https://zhuanlan.zhihu.com/p/84261043)

## 评估NLU模型

机器学习中通用的方法是从数据集中分离出测试集。你可以将训练数据拆分成训练和测试集：

```
rasa data split nlu
```

如果执行了该函数，你可以观察你的NLU模型对于测试集数据有怎样的表现：

```
rasa test nlu -u test_set.md --model models/nlu-20180323-145833.tar.gz
```

如果你不想要单独的测试集，你可以使用交叉验证来评估你的模型，如下：

```
rasa test nlu -u data/nlu.md --config config.yml --cross-validation
```

该命令的完整的描述如下：

```
usage: rasa test nlu [-h] [-v] [-vv] [--quiet] [-m MODEL] [-u NLU] [--out OUT]
                     [--successes] [--no-errors] [--histogram HISTOGRAM]
                     [--confmat CONFMAT] [-c CONFIG [CONFIG ...]]
                     [--cross-validation] [-f FOLDS] [-r RUNS]
                     [-p PERCENTAGES [PERCENTAGES ...]]

optional arguments:
  -h, --help            show this help message and exit （显示帮助信息并推出）
  -m MODEL, --model MODEL
                        Path to a trained Rasa model. If a directory is
                        specified, it will use the latest model in this
                        directory. (default: models)
                        （Rasa训练得到的模型的路径）
  -u NLU, --nlu NLU     File or folder containing your NLU data. (default:
                        data)
                        （NLU数据的文件或路径）
  --out OUT             Output path for any files created during the
                        evaluation. (default: results)
                        （用来保存在评估中输出的任意文件）
  --successes           If set successful predictions (intent and entities)
                        will be written to a file. (default: False)
                        （将预测正确的结果保存到文件中）
  --no-errors           If set incorrect predictions (intent and entities)
                        will NOT be written to a file. (default: False)
                        （不正确的预测不会写入到文件中）
  --histogram HISTOGRAM
                        Output path for the confidence histogram. (default:
                        hist.png)
                        （可信度直方图）
  --confmat CONFMAT     Output path for the confusion matrix plot. (default:
                        confmat.png)
  -c CONFIG [CONFIG ...], --config CONFIG [CONFIG ...]
                        Model configuration file. If a single file is passed
                        and cross validation mode is chosen, cross-validation
                        is performed, if multiple configs or a folder of
                        configs are passed, models will be trained and
                        compared directly. (default: None)
                        （模型配置文件，如果单个文件，进行交叉验证，否则，训练和比较是直接进行的）

Python Logging Options:
  -v, --verbose         Be verbose. Sets logging level to INFO. (default:
                        None)
  -vv, --debug          Print lots of debugging statements. Sets logging level
                        to DEBUG. (default: None)
  --quiet               Be quiet! Sets logging level to WARNING. (default:
                        None)

Cross Validation:
  --cross-validation    Switch on cross validation mode. Any provided model
                        will be ignored. (default: False)
  -f FOLDS, --folds FOLDS
                        Number of cross validation folds (cross validation
                        only). (default: 10)

Comparison Mode:
  -r RUNS, --runs RUNS  Number of comparison runs to make. (default: 3)
  -p PERCENTAGES [PERCENTAGES ...], --percentages PERCENTAGES [PERCENTAGES ...]
                        Percentages of training data to exclude during
                        comparison. (default: [0, 25, 50, 75])
```

### 比较NLU pipelines

可以通过传入多个pipeline配置文件（或包含他们的目录）到CLI，rasa将运行不同pipelines的比较试验。

```
$ rasa test nlu --config pretrained_embeddings_spacy.yml supervised_embeddings.yml --nlu data/nlu.md --runs 3 --percentages 0 25 50 70 90
```

上述示例中的命令将从你的数据中创建训练和测试集，然后在训练集中排除0,25,50,70,90%的目标数据的情况下多次训练每个pipeline。然后在测试集上对模型进行评估，并记录每个排除百分比的F1分数。这个过程运行三次（如，总共有三个测试集），然后使用F1分数的平均值和标准差绘制一个图标。

F1分数图，连通所有的训练、测试集，训练模型，分类和误差的报告，将保存到名为nlu_comparision_results的文件夹中。

### 意图分类

评估脚本会根据你的模型产生报告，混淆矩阵，置信直方图。

报告记录了每个意图和实体的精确度，召回率，f1值，同时提供了所有的均值。你可以使用`--report`参数将结果保存到JSON文件中。

混淆矩阵会告诉你哪些意图识别出错了；任何错误预测的示例都会被记录到`errors.json`中，方便后续的调试。

脚本产生的直方图可以让你对所有预测的置信度的分布有个视觉上的认识，正确和错误预测的数量分别用蓝色和红色条显示。提高训练数据的质量将使蓝色直方图条向右移动，红色直方图条向左移动。

**注意**：只有当你在测试集上对模型进行评估的时候，才会产生混淆矩阵。在交叉验证模式下，混淆矩阵不会产生。

**警告**：如果你的实体有错误标记的，你的评价会失败。一个常见的问题是，一个实体的开始和结束不能出现在token内部。比如，你有一个name实体，如`[Brain](name)'s house`，这个只有当`Brain's`被划分为两个token的时候才是有效的。而，空格分词方式并不能对上述的例子进行划分。

### 回复选择

评估脚本将为管道中的所有响应选择器模型生成一个组合报告。

报告会记录每一个回复的精度，召回率，和f1分数。并会提供所有回复的均值。你可以将这些结果使用参数`--report`记录到JSON文件中。

### 实体提取

CRFEntityExtractor是唯一一个利用你的数据进行训练的实体提取器，因此这个也是唯一一个会被评估的。如果你使用spaCy或者duckling预训练的实体提取器，Rasa NLU将不会把这些包括到评估中。

针对实体提取的每一个类型，Rasa NLU将会报告召回率，精度，f1值。

### 实体评分

为了评估实体提取，我们需要一个简单的打标签的过程。我们不考虑BILOU标记，只考虑每个标记的实体类型标记。对于`near Alexanderplatz`这样的地点实体，我们期望`LOC LOC`的标签替换基于BILOU的`B-LOC L-LOC`。我们的途径针对评估的时候会更加的宽容，因为它奖励部分提取，而不惩罚实体的分裂。例如，考虑到上述实体“near Alexanderplatz”和提取“Alexanderplatz”的系统，我们的方法奖励提取“Alexanderplatz”，并惩罚漏掉的单词“near”。然而，BILOU的方法，这样的识别是完全错误的，它期望Alexanderplatz标记成L-LOC(实体的最后一个token)，而不是单个token实体（U-LOC）。注意，我们的方法对于near和Alexanderplatz分开提取的方式，得到的分数也是满分，而BILOU会得到0分。

以下是“near Alexanderplatz tonight”这句话的两种评分机制的比较：

```
extracted                                          Simple tags(score)        BILOU tags (score)
[near Alexanderplatz](loc) [tonight](time)         loc loc time (3)          B-loc L-loc U-time (3)
[near](loc) [Alexanderplatz](loc) [tonight](time)  loc loc time (3)          U-loc U-loc U-time (1)
near [Alexanderplatz](loc) [tonight](time)         O loc time (2)            O U-loc U-time (1)
[near](loc) Alexanderplatz [tonight](time)         loc O time (2)            U-loc O U-time (1)
[near Alexanderplatz tonight](loc)                 loc loc loc (2)           B-loc I-loc L-loc (1)
```

## 评估Core模型

你可以通过测试故事评估训练得到的模型，如下：

```
rasa test core --stories test_stories.md --out results
```

失败的故事会写入到`results/failed_stories.md`。只要故事中有一个action预测出错了，那么就会被写入到`failed_stories.md`文件中。

此外，对应的混淆矩阵会存入到`results/story_confmat.pdf`。对于domain中的每个action，这个混淆矩阵显示了action被正确预测的可能性，有多少可能预测失败。

该命令完整的选项如下：

```
usage: rasa test core [-h] [-v] [-vv] [--quiet] [-m MODEL [MODEL ...]]
                      [-s STORIES] [--max-stories MAX_STORIES] [--out OUT]
                      [--e2e] [--endpoints ENDPOINTS]
                      [--fail-on-prediction-errors] [--url URL]
                      [--evaluate-model-directory]

optional arguments:
  -h, --help            show this help message and exit
  -m MODEL [MODEL ...], --model MODEL [MODEL ...]
                        Path to a pre-trained model. If it is a 'tar.gz' file
                        that model file will be used. If it is a directory,
                        the latest model in that directory will be used
                        (exception: '--evaluate-model-directory' flag is set).
                        If multiple 'tar.gz' files are provided, all those
                        models will be compared. (default: [None])
  -s STORIES, --stories STORIES
                        File or folder containing your test stories. (default:
                        data)
  --max-stories MAX_STORIES
                        Maximum number of stories to test on. (default: None)
  --out OUT             Output path for any files created during the
                        evaluation. (default: results)
  --e2e, --end-to-end   Run an end-to-end evaluation for combined action and
                        intent prediction. Requires a story file in end-to-end
                        format. (default: False)
  --endpoints ENDPOINTS
                        Configuration file for the connectors as a yml file.
                        (default: None)
  --fail-on-prediction-errors
                        If a prediction error is encountered, an exception is
                        thrown. This can be used to validate stories during
                        tests, e.g. on travis. (default: False)
  --url URL             If supplied, downloads a story file from a URL and
                        trains on it. Fetches the data by sending a GET
                        request to the supplied URL. (default: None)
  --evaluate-model-directory
                        Should be set to evaluate models trained via 'rasa
                        train core --config <config-1> <config-2>'. All models
                        in the provided directory are evaluated and compared
                        against each other. (default: False)

Python Logging Options:
  -v, --verbose         Be verbose. Sets logging level to INFO. (default:
                        None)
  -vv, --debug          Print lots of debugging statements. Sets logging level
                        to DEBUG. (default: None)
  --quiet               Be quiet! Sets logging level to WARNING. (default:
                        None)
```

## 比较Core配置

要为了你的核心模型选择配置，或者对特定策略选择超参数，你想要知道rasa core将如何概括为以前从未见过的对话。特别是在一个项目的开始，你并没有很多真实的对话训练你的机器，因此，你没必要从训练数据中挑选一部分作为测试集。

rasa核心有脚本帮助你选择和微调策略的配置。一旦你感到满意，你可以在你的全部数据集上针对你的最终配置进行训练。为了这样做，你首先要针对不同的配置文件进行训练。创建两个（或多个）包含你想要比较的策略的配置文件，然后使用`compare`来训练你的模型：

```
$ rasa train core -c config_l.yml config_2.yml \
  -d domain.yml -s stories_folder --out comparison_models --runs 3 \
  --percentages 0 5 25 50 70 95
```

对于提供的每个策略配置，Rasa Core将针对0,5,25,50,50,95%的训练数据被排除后分别进行训练。这个会被重复执行多次，确保结果的一致性。

一旦脚本完成了，你可以使用对训练得到的模型进行评估，如下：

```
$ rasa test core -m comparison_models --stories stories_folder
  --out comparison_results --evaluate-model-directory
```

这将评估所提供故事中的每个模型（可以是训练或测试集），并绘制一些图表，以显示哪个策略执行得最好。通过对全套故事进行评估，您可以衡量rasa core对被搁置的故事的预测好坏。

要比较单个策略，请创建每个策略仅包含一个策略的配置文件。如果你不确定要比较哪些策略，我们建议尝试EmbeddingPolicy和KerasPolicy，看看哪个更适合。

**注意**：这个训练过程会花费很长时间，我们建议让他再后台训练。

## 端到端的评估

Rasa允许你以端到端的方式对对话进行评估，运行测试对话，确保NLU和Core都能够得到正确的预测。

为了实现这个，你需要一些端到端格式的故事，这些故事包含NLU的输出，和原始文本，如：

```
## end-to-end story 1
* greet: hello
   - utter_ask_howcanhelp
* inform: show me [chinese](cuisine) restaurants
   - utter_ask_location
* inform: in [Paris](location)
   - utter_ask_price
```

如果您已将端到端故事保存为名为e2e_stories.md的文件，则可以通过运行以下命令来评估您的模型：

```
$ rasa test --stories e2e_stories.md --e2e
```

**注意**：确保模型中的模型文件是Core和NLU模型的组合。如果它不包含nlu模型，core将使用默认的RegexInterpreter。

## 原文链接

https://rasa.com/docs/rasa/user-guide/evaluating-models/