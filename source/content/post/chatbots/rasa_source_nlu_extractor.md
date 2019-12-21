+++
date="2019-12-21"
title="rasa source - nlu 实体提取代码走读"
categories=["chatbot","rasa"]
tags=["rasa-source"]
+++

# rasa source - nlu 实体提取代码走读

实体提取实现包括如下文件：

- crf_entity_extractor.py，基于条件随机场的实现
- mitie_entity_extractor.py，基于mitie实体提取
- spacy_entity_extractor.py，基于spacy实体提取
- entity_synonyms.py，实体同义词映射
- duckling_http_extractor.py，使用duckling服务实现，不需要进行训练

下面针对`crf_entity_extractor.py`进行介绍，其他类似操作即可。

## 条件随机场实现代码走读

相关代码路径为：https://github.com/RasaHQ/rasa/tree/master/rasa/nlu/extractor/crf_entity_extractor.py

这部分代码相比较分词而言复杂很多，为了更好的理解代码，使用调试模式，同时考虑到只有在nlu中定义了实体，才会走入到实体提取的代码，将https://github.com/RasaHQ/rasa/tree/master/examples/restaurantbot中`data/nlu.md`的内容进行了更改，如下：

```
## intent:affirm
- right
- yes
- i love that

## intent:deny
- no
- uh no

## intent:greet
- hi
- hey
- hello

## intent:inform
- what about [indian](cuisine) food
- um [english](cuisine)
- im looking for [world](cuisine) food
- how about [indian](cuisine) food
- ok how about [chinese](cuisine) food

## intent:request_info
- do you have their [address](info)
- do you have its [phone number](info)
- can i have their [phone number](info)
- what is the [phone number](info) of the restaurant
- what is their [address](info)

## intent:thankyou
- thank you
- thanks
```

并且对config.yml进行修改，不使用spacy，具体如下：

```python
language: en

pipeline: supervised_embeddings

policies:
  - name: KerasPolicy
  - name: MemoizationPolicy
  - name: MappingPolicy
```

然后对`site-package/rasa/nlu/extractors/crf_entity_extractor.py`的`__init__`和`train`函数，添加`pdb.set_trace()`，接着执行`rasa train nlu`跟踪调试状态。

在`__init__`代码中发现`self.pos_features`设置成了false。继续执行，断点进入train函数，该函数如下：

```python
class CRFEntityExtractor(EntityExtractor):

    provides = ["entities"] # 输出是识别后的实体

    requires = ["tokens"]   # 需要的输入是分词后的结果

    # ...
    ## 
    def train(
        self, training_data: TrainingData, config: RasaNLUModelConfig, **kwargs: Any
    ) -> None:

        # checks whether there is at least one
        # example with an entity annotation
        # examples为Message对象组成的列表
        if training_data.entity_examples:
            self._check_spacy_doc(training_data.training_examples[0]) # len(training_examples): 20

            # filter out pre-trained entity examples
            # 过滤出不需要训练的实体标记，将其entities字段赋值为[]
            filtered_entity_examples = self.filter_trainable_entities(
                training_data.training_examples
            )

            # convert the dataset into features
            # this will train on ALL examples, even the ones
            # without annotations
            dataset = self._create_dataset(filtered_entity_examples)

            self._train_model(dataset)

    
```

`_create_dataset()`函数实现如下：

```python
    def _create_dataset(
        self, examples: List[Message]
    ) -> List[
        List[
            Tuple[
                Optional[Text],
                Optional[Text],
                Text,
                Dict[Text, Any],
                Optional[Dict[Text, Any]],
            ]
        ]
    ]:
        dataset = []
        for example in examples:
            # 将示例转换成[(entity_start, entity_end, entity)]形式的列表
            entity_offsets = self._convert_example(example)
            # _from_json_to_crf主要工作是将json形式的数据转换成crfsuite要求的数据格式
            dataset.append(self._from_json_to_crf(example, entity_offsets))
        return dataset
```

`_train_model`的实现如下：

```python
    def _train_model(
        self,
        df_train: List[
            List[
                Tuple[
                    Optional[Text],
                    Optional[Text],
                    Text,
                    Dict[Text, Any],
                    Optional[Dict[Text, Any]],
                ]
            ]
        ],
    ) -> None:
        """Train the crf tagger based on the training data."""
        import sklearn_crfsuite

        X_train = [self._sentence_to_features(sent) for sent in df_train]
        y_train = [self._sentence_to_labels(sent) for sent in df_train]
        # 使用了sklearn_crfsuite
        self.ent_tagger = sklearn_crfsuite.CRF(
            algorithm="lbfgs",
            # coefficient for L1 penalty
            c1=self.component_config["L1_c"],
            # coefficient for L2 penalty
            c2=self.component_config["L2_c"],
            # stop earlier
            max_iterations=self.component_config["max_iterations"],
            # include transitions that are possible, but not observed
            all_possible_transitions=True,
        )
        self.ent_tagger.fit(X_train, y_train)
```

关于crfsuite的介绍在后续给出。

## 补充知识

- [条件随机场入门（一） 概率无向图模型](https://www.cnblogs.com/ooon/p/5817732.html)
- [条件随机场入门（二） 条件随机场的模型表示](https://www.cnblogs.com/ooon/p/5818227.html)
- [条件随机场入门（三） 条件随机场的概率计算问题](https://www.cnblogs.com/ooon/p/5823445.html)
- [条件随机场入门（四） 条件随机场的训练](https://www.cnblogs.com/ooon/p/5826757.html)
- [条件随机场入门（五） 条件随机场的预测算法](https://www.cnblogs.com/ooon/p/5827078.html)
- [An Introduction to Conditional Random Fields](http://homepages.inf.ed.ac.uk/csutton/publications/crftut-fnt.pdf)
- [crfsuit](http://www.chokkan.org/software/crfsuite/)

## 



