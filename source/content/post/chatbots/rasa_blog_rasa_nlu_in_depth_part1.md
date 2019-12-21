+++
date="2019-12-21"
title="rasa blog - 深入理解rasa NLU: Part1 - 意图分类 (翻译)"
categories=["chatbot","rasa","翻译"]
tags=["rasa-blog"]
+++

# rasa blog - 深入理解rasa NLU: Part1 - 意图分类 (翻译)

AI助手需要完成两项任务：理解用户和给出正确的回复。Rasa通过rasa自然语言理解（Rasa NLU）和对话管理组件（Rasa Core）还处理这两个任务。基于我们与rasa社区和世界各地的客户的合作，我们现在针对如何为你个人的AI助手定制Rasa NLU，分享我们的最佳实践和建议。为了你能够对各个组件有个深入的理解，我们分成了三个连续的blog文章：

- Part1：Intent Recognition - 如何更好的理解你的用户
- Part2：Entity Extraction - 针对每个实体选择合适的提取器
- Part3：Hyperparameters - 如何选择和优化它们

是什么使得用于上下文AI助手的NLP如此特殊，以致于我们决定在这个系列的三篇博文中介绍它？原因是上下文AI助手可以是高度领域化的，也就是说它们必须为你的用例进行定制，就像网站为每个公司进行定制一样。Rasa NLU提供了可以完全定制化的通过pipeline处理用户的消息。一个pipeline定义了顺序处理用户消息的不同组件。如果你想要学习pipeline是怎么工作的，以及如何实现你自己的NLU组件，可以查看：[latest blog post on custom components](https://blog.rasa.com/enhancing-rasa-nlu-with-custom-components/)。

这篇博文是系列文章的开头，将让你深入理解哪些NLU组件可以帮助更好的理解用户，包括：

- 在你的项目中，你应该使用哪个意图分类组件
- 如何处理常见的问题：缺少训练数据，出现超出单词表的单词，对于相近意图给出更健壮的分类，数据集扭曲（skewed dataset）

## Intents： 用户说了什么

Rasa使用intents的概念来描述用户消息该怎么被分类。Rasa NLU能够将用户消息分成一个或多个intents。这里有两个组件可供选择：

- 预训练embedding （[Intent_classifier_sklearn](http://rasa.com/docs/rasa/nlu/components/#sklearnintentclassifier))
- 监督式的embedding（[Intent_classifier_tensorflow_embedding](http://rasa.com/docs/rasa/nlu/components/#embeddingintentclassifier)）

### 预训练Embeddings：Intent Classifier Sklearn

该分类器使用了[spaCy library](https://spacy.io/) 来加载预训练语言模型，然后作为词嵌入被用来表示用户消息中的每个单词。词嵌入式单词的向量化表示，也就是说每个单词转换成稠密的数值向量。词嵌入能够包括单词的语义和语法。也就是说，相近的单词应该被表示成相近的向量。如果你想要学习更多关于词嵌入的内容，可以查看[word2vec paper](https://arxiv.org/abs/1301.3781)。

词嵌入针对被训练的语言。因此，你可以根据你使用的语言选择不同的模型。参见[overview of available spaCy language models](https://spacy.io/models/).如果你想要使用不同的词嵌入，如，[Facebook’s fastText embeddings](https://github.com/facebookresearch/fastText/blob/master/docs/crawl-vectors.md#models)。你可以参照 [spaCy guide here](https://spacy.io/usage/vectors-similarity#converting)将该词嵌入转换成兼容spaCy模型的形式，然后将转换得到的模型连接到你选择的语言（如，en），具体命令如`python -m spacy link <converted model> <language code>`.

Rasa NLU求得消息中所有词嵌入的均值，然后通过网格搜索，找到支持向量机分类器的最佳参数，将平均embeddings值转换成不同的意图。网格搜索就是用不同的参数配置训练多个支持向量机，然后根据测试结果选择最佳的配置参数。

**什么时候你需要使用这个组件：**

当你能够使用的预训练词嵌入能够让你从最先进的研究获取的更加powerful更加meaningful的词嵌入中受益。由于词嵌入已经是训练好了的，SVM只需要进行一小部分的训练就能够对意图进行预测。这使得该分类器非常适合你的上下文AI助手的起步阶段。即使你只有很少的训练数据，通常情况是这样的，你仍将获得健壮的分类结果。由于训练不是从什么都没有开始的，训练将以很快的速度执行。

不幸的是，对于所有语言的好的词嵌入并不是总存在的，因为大多数情况下针对公开数据集的词嵌入都是英语。并且他们也不是领域相关的单词，例如产品名称或首字母缩略词。在这种情况下，最好的办法是使用supervised embeddings classifier对你的词嵌入进行训练。

### Supervised Embeddings: Intent Classifier TensorFlow Embedding

意图分类器 [intent_classifier_tensorflow_embedding](http://rasa.com/docs/rasa/nlu/components/#embeddingintentclassifier)是由Rasa开发的，并且是受[Facebook’s starspace paper](https://arxiv.org/abs/1709.03856)的启发。与使用预训练embeddings和在其之上训练分类器的方式不同，它是从零开始训练词嵌入。它通常和[intent_featurizer_count_vectors](http://rasa.com/docs/rasa/nlu/components/#countvectorsfeaturizer)组件结合到一起使用，该组件用来统计训练数据中不同单词出现在消息中的频率，然后将这个值提供给意图分类器作为输入。下面的示例中你可以看到计数向量将如何区分句子`My bot is the btest bot`和`My bot is great`，如`bot`在第一句话中出现了两次。除了使用使用单词token计数，你可以通过改变[intent_featurizer_count_vectors](http://rasa.com/docs/rasa/nlu/components/#countvectorsfeaturizer)组件中的`analyzer`属性为`char`，就可以使用ngram计数。者能够使得意图分类器对于类型更加的强健，但是会增加训练时间。

![](https://blog.rasa.com/content/images/2019/02/image.png)

此外，针对意图标签的另一个计数向量也要被创建。与预训练词嵌入比较，tensorflow embedding classifier还支持带过个意图的消息（如，你说`hi, how is the weather?`这个消息的意图是`greet`和`ask_weather`）。这意味着计数向量并不一定是one-hot编码。该分类器能够针对特征和意图向量学会各自的嵌入。两个嵌入有相同的维度，这个使得使用余弦相似测量嵌入的向量和嵌入的意图之间的距离成为可能。在训练的时候，用户消息和相关的意图标签之间的余弦相似会被最大化。

**什么时候使用该组件：**

由于这个分类器是从零开始训练词嵌入，它相对于使用预训练embeddings的分类器而言需要更多的训练数据。但是，由于训练是完全基于你的训练数据的，它能够适应于你的领域独有的消息，如，没有消失单词的embeddings。同时，它是独立于语言的，并不会依赖于针对某种语言的好的词嵌入。该分类器另一个很棒的特征是它只是带多意图的消息。通常情况下，这使得该分类器称为能够支持高级使用场景的非常灵活的分类器。

注意，针对一些语言（如，中文），使用Rasa NLU的默认途径使用空格将句子分割成单词是不可能的。这种情况下，你必须使用不同的分词器组件（如，Rasa针对中文提供的jieba分词器）。

如果你仍然不确定哪个组件更适合你的AI助手，使用下面的流程图快速获得经验法则决策。

![](https://blog.rasa.com/content/images/2019/02/image-1.png)

## 常见问题

### 缺少训练数据

当你的机器实际被用户使用的时候，你将有大量的对话数据可供作为训练示例的选择。但是，在初始阶段，缺少甚至没有训练数据，以及你的意图分类器的准确率很低，都是常见的问题。克服这个问题常见的方法是使用数据生成工具[chatito](https://rodrigopivi.github.io/Chatito/)。从预定义的单词区块中生成句子能够很快给你很大的数据集。要注意避免过多的使用数据生成工具，那样会使得你的模型对生成的句子出现过拟合的情况。我们强烈建议你使用来自真实用户的数据。另一种获取更多训练数据的途径是众包，如，使用[Amazon Mechanical Turk](https://www.mturk.com/) (mturk)。通过我们的经验得出，[interactive learning feature of Rasa Core](http://rasa.com/docs/rasa/core/interactive-learning/)对于获取新的Core和NLU训练数据也是很有帮助的：当实际和机器进行对话的时候， 在孤立的环境中思考潜在的例子的时候，你会自动的用不同的方式框定你的消息。

### 出现超出单词表的单词

不可避免的，用户会使用到没有对应词嵌入的单词，如，使用到你没有考虑到的单词。在使用预训练词嵌入的时候，不能做的只有尝试从更大的语料库训练得到的语言模型。如果你是使用`intent_classifier_tensorflow_embedding`分类器从零开始训练得到embeddings的，那么你有两个选择：引入更多的训练数据，或者在示例中添加*OOV_token* (超出单词表的标记)。你可以通过配置`intent_classifier_tensorflow_embedding`组件的`OOV_token`参数，如，将其设置为`<OOV>`，然后添加包含该值的示例，如，`My <OOV> is Sara`。通过这种方式，分类器将会学会如何处理消息中包含的超出单词表的单词。

### 相近的意图

当意图非常相近的时候，很难将他们区分。这个听上去是显而易见的，但在创建意图的时候通常会被忽略。假想一种场景，用户提供了他们的名字或是给你一个日期。直觉上你回创建一个意图`provide_name`对应消息`It is Sara`，以及一个意图`provide_date`针对消息`It is on Monday`。但是，从NLU方面进行该考虑，这些消息是非常相近的，除了他们的实体不同。因此，一种更好的方式是创建`inform`意图来统一`provide_name`和`provide_date`。在你的Rasa Core故事中，你可以依赖提取出来的实体选择不同的故事路径。

### 扭曲的数据集（Skewed data）

你应该时刻注意维持每个意图示例数量的平衡。但是，有些情况意图（如，上面提到的`inform`意图）的示例相对于其他意图会增长的特别款。虽然通常情况下，更多的数据集有助于获取更高的精度，但是很强的不平衡性会导致分类器有比较大的偏差，从而会降低精度。超参数优化将在本系列的第三部分中介绍，它可以帮助您缓冲负面影响，但迄今为止最好的解决方案是重新建立一个平衡的数据集。

## 小结

这篇文章基于我们使用Rasa的日常工作给出了针对你的需求定制NLU意图识别器最好的实践方式和建议。不管你是想要减少训练时间快速的开始构建你的AI助手的项目，还是需要从零开始训练词嵌入：Rasa NLU都提供了完整的自定义途径。当读完Rasa NLU in Depth第一部分之后，你应该能够充分的认识到如何去选择更合适的组件用于意图分类，以及如何去配置它。

什么是实体识别呢？本系列接下来的第2部分将为您提供一些第一手建议：选择哪个实体提取器组件，以及如何解决地址提取或模糊实体识别等问题。

此时你可以做什么呢：

- [Share your NLU tweaking experiences with the community in the Rasa forum](https://forum.rasa.com/t/rasa-nlu-in-depth-part-1-intent-classification/5412)
- [Design your own NLU pipeline component](https://blog.rasa.com/enhancing-rasa-nlu-with-custom-components/)
- [Learn to fail gracefully](https://blog.rasa.com/failing-gracefully-with-rasa-core/)

## 原文链接

https://blog.rasa.com/rasa-nlu-in-depth-part-1-intent-classification/