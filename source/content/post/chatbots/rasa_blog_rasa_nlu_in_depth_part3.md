+++
date="2019-12-21"
title="rasa blog - 深入理解rasa NLU: Part3 - 超参数调整 (翻译)"
categories=["chatbot","rasa","翻译"]
tags=["rasa-blog"]
+++

# rasa blog - 深入理解rasa NLU: Part3 - 超参数调整 (翻译)

欢迎来到Rasa NLU深入理解系列的最后一篇文章。

在这个系列的文章中，我们会将从社区以及客户中获取的关于Rasa NLU的最佳实践和经验分享给大家。

快速的回顾一下，之前已经介绍的内容：

- [如何理解你的用户 - Rasa NLU意图分类器](https://zhuanlan.zhihu.com/p/84075497)
- [从用户消息中提取实体 - 实体识别](https://zhuanlan.zhihu.com/p/84220988)

该系列的前两个部分已经为你创建AI助手选择NLU管道提供了所有的工具和知识。最后一部分，将介绍如何对Rasa NLU管道中的组件进行配置调优，以用户最佳的表现。这篇文章包含以下内容：

- 如何用Rasa NLU进行大规模参数优化
- 哪个超参数在微调时对结果优化最明显

## 超参数优化

前两部分已经介绍了哪个NLU组件更加适合你的应用场景以及如何对潜在问题进行处理。选择合适的组件是你的AI助手应用成功的关键。但是，如果你想要进一步充分利用组件，你必须要对单个组件的配置参数（叫做超参数）进行调优。寻找最佳配置的方法是，用不同的配置参数进行训练，然后分别在验证集上面进行评估。通过超参数查找，可以找到评估分数最高的超参数配置。由于组件有很多的超参数，以及模型训练是时间密集型的，我们将展示如何使用Docker容器，帮助你将超参数查找更好的扩展到多个机器上。

### 定义查找空间

在开始之前，前克隆[rasaHQ/nlu-hyperopt repository](https://github.com/RasaHQ/nlu-hyperopt)。为你的NLU管道定义模板，可以对`data/template_config.yml`进行更改。并将参数替换成你想要优化的参数，使用大括号括起来，如：

```
language: en
pipeline:
- name: "intent_featurizer_count_vectors"
- name: "intent_classifier_tensorflow_embedding"
  epochs: {epochs} 
```

在上面的例子中，我们定义了NLU管道用于`intent_classifier_tensorflow_embedding`意图识别。在超参数查找的时候，我们将试图查找最佳的训练迭代次数。

下一个步骤是定义你的NLU模型中想要评估的参数的范围。根据你想要评估的超参数，调整`nlu_hyperopt/space.py`文件中的内容，如：

```python
from hyperopt import hp

search_space = {
    'epochs': hp.uniform(“epochs”, 1, 10)
}
```

针对查找空间，模型将被以不同的epochs数值进行训练，epochs的取值范围为从1到10。你可以从其他的分布中进行选择。参见[hyperopt docs](https://github.com/hyperopt/hyperopt/wiki/FMin#21-parameter-expressions)。

组件有很多的超参数，那么我们该从什么地方开始呢？由于预训练词嵌入[intent_classifier_sklearn](http://rasa.com/docs/rasa/nlu/components/#sklearnintentclassifier)已经在训练中执行了网格搜索，当你使用[intent_classifier_tensorflow_embedding](http://rasa.com/docs/rasa/nlu/components/#embeddingintentclassifier)训练你自己的词嵌入的时候，超参数优化会更你更多的额外的好处。该组件的重要的超参数是来自 [intent_featurizer_count_vectors](http://rasa.com/docs/rasa/nlu/components/#countvectorsfeaturizer)组件和分类器本身的。对于组件 [intent_featurizer_count_vectors](http://rasa.com/docs/rasa/nlu/components/#countvectorsfeaturizer)，我们建议考虑`min_df`，`max_df`，和`max_features`。更详细介绍参见：[Sklearn documentation](https://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.CountVectorizer.html)。

[tensorflow classifier](http://rasa.com/docs/rasa/nlu/components/#embeddingintentclassifier)含有很多的参数。我们建议从调整embeddings的维度（`embed_dim`）和隐藏层的大小（`hidden_layers_sizes_a,hidden_layers_sizes_b`）开始。针对上面三个参数，更大的值能够获得更高的精度，但是也可能会导致过拟合。

### 配置超参数搜索

最后，配置你的尝试。这是通过环境变量的设置实现的。如果你想要顺序的运行超参数查找，或不适用Docker，那么你可以忽略mongo数据库的设置。这个选项的详细介绍见[readme](https://github.com/RasaHQ/nlu-hyperopt/blob/master/README.md)，因此为了简单起见，我们将关注最重要的几个参数。

**MAX_EVALS**

这个参数描述了你想要运行多少次评估。如果参数的组合（查找空间）比较小，你也许需要选择一个较小的值。如果查找空间很大，为了更好的覆盖查找空间，必须要执行更多的评估。

**TARGET_METRIC**

这个值定义了用于比较不同训练模型的评价指标。可供的选择有：

- `f1_score`：在评估数据集中查找`f1`分数最高的模型
- accuracy：在评估数据集中查找准确度最高的模型
- precision：在评估数据集中查找精度最高的模型
- `threshold_loss`：其他的度量方式会选择出置信度最高的意图作为正确的结果，该损失函数仅会选择出预测结果是正确的，并且置信度值高于阈值的结果。这说明了使用fallback策略来消除低置信度预测的歧义。你可以使用参数`ABOVE_BELOW_WEIGHT`来指定使用阈值之上还是阈值之下的预测作为不正确的预测。

然后添加训练集用于模型训练，评价集用于模型评估。通过将训练数据放到`data/train.md`，将评估数据放到`data/validation.md`进行指定。

### 运行

终于到了运行参数搜索的时间了。你可以选择不使用Docker或使用Docker容器来运行超参数搜索。

如果你想要在本地运行，通过命令`pip install -r requirements.txt`安装依赖，然后执行`python -m nlu_hyperopt.app`。

如果想要通过Docker运行，可以执行命令：`docker-compose up -d --scale hyperopt-work=<number of concurrent workers>`。这个会构建包含你的数据集，搜索空间，和模板配置的docker镜像，接着使用mongo数据库并行的运行。当然，你可以在集群运行，将工作分发到不同的机器上。常见的集群编排工具，如Kubernetes，能够处理生成的docker-compose文件。

当评估结束之后，会在日志中输出最佳的pipeline配置，如：

```
INFO:__main__:The best configuration is:

language: en
pipeline:
- name: "intent_featurizer_count_vectors"
- name: "intent_classifier_tensorflow_embedding"
  epochs: 8.0
```

如果你通过Docker和mongo数据库进行超参数搜索，所有的评估结果会存储到mongo数据库中。通过执行`docker-compose exec mongodb mongo`进入mongo容器查看评估结果。下面的命令会输出对应的结果：

```
db.jobs.find({"exp_key" : "default-experiment", "result.loss":{$exists: 1}}).sort({"result.loss": 1})
```

## 小结

这篇博文是我们三篇Rasa NLU深入系列文章中的最后一篇，它反映了我们的最佳实践和建议，以便根据您的需求完全定制NLU管道。你现在应该是一个RASA NLU专家，并有信心为你的个人上下文人工智能助手选择和定制完美的RASA NLU管道。祝贺你！

## 原文地址

https://blog.rasa.com/rasa-nlu-in-depth-part-3-hyperparameters/