+++
date="2019-12-21"
title="rasa source - nlu classifiers源码走读"
categories=["chatbot","rasa"]
tags=["rasa-source"]
+++

# rasa source - nlu classifiers源码走读

classifiers包括如下实现：

- embedding_intent_classifier.py：构建神经网络
- keyword_intent_classifier.py：根据关键词进行分类
- mitie_intent_classifier.py：利用mitie
- sklearn_intent_classifier.py：使用sklearn中的SVC

重点介绍一下`embedding_intent_classifier.py`的实现。从源代码的注释中可以发现，该实现基于：https://arxiv.org/abs/1709.03856。关于该文章的其他人的读书笔记：https://www.jianshu.com/p/35c15221c1c4。

示例准备参见：[rasa source - nlu 实体提取代码走读](https://zhuanlan.zhihu.com/p/86715765)

类EmbeddingIntentClassifier构造函数在第一次调用的时候除了第一个参数以外，其他的参数均为None。下面首先来看一下train函数调用的时候，不同函数的调用情况（PS 在后续不同函数走读的时候需要不是的返回到这里，查看走读的函数所处的位置）：

```
- `train`
  - `preprocess_train_data`：数据可用性测试，准备训练数据
    - `_create_label_id_dict`：创建label和id对应的字典
    - `_create_encoded_label_ids`：将label_ids编码为词袋
      - `_find_example_for_label`：针对每个label收集一个example
      - `_check_labels_features_exist`：判断所有的label是不是向量化
      - `_extract_labels_precomputed_features`：返回编码后的labels示例
      - `_compute_default_label_features`：返回编码后的labels示例
    - `_create_session_data`：为了训练准备数据，创建SessionData对象
    - `check_input_dimension_consistency`：对数据的维度进行校验
  - `_check_enough_labels`：labels的数量要大于等于2
  - `train_utils.train_val_split`：训练数据分割
  - `train_utils.create_iterator_init_datasets`：创建数据集的iterator
  - `_build_tf_train_graph`：构建训练网络
    - `_create_tf_embed_fnn`：构建全连接网络
    - `train_utils.calculate_loss_acc`
      - `sample_negatives`：示例的负采样
        - `_tf_get_negs`
        - `_tf_make_flat`
      - `tf_sim`：相似度计算
      - `tf_calc_accuracy`：准确度计算
      - `choose_loss`：选择损失函数
  - `train_utils.train_tf.dataset`
  - `_build_tf_pred_graph`：构建预测用的网络
```

首先来看下`_create_label_id_dict`的实现，如下：

```python
    # training data helpers:
    @staticmethod
    def _create_label_id_dict(
        training_data: "TrainingData", attribute: Text
    ) -> Dict[Text, int]:
        """Create label_id dictionary"""

        # distinct_label_ids: {'deny', 'greet', 'inform', 'affirm', 'thankyou', 'request_info'}
        distinct_label_ids = set(
            [example.get(attribute) for example in training_data.intent_examples]
        ) - {None}
        # 返回值为：{'affirm': 0, 'deny': 1, 'greet': 2, 'inform': 3, 'request_info': 4, 'thankyou': 5}
        return {
            label_id: idx for idx, label_id in enumerate(sorted(distinct_label_ids))
        }
```

`_create_encoded_label_ids`实现如下：

```python
    def _create_encoded_label_ids(
        self,
        training_data: "TrainingData",
        label_id_dict: Dict[Text, int],
        attribute: Text,
        attribute_feature_name: Text,
    ) -> np.ndarray:
        """Create matrix with label_ids encoded in rows as bag of words. If the features are already computed, fetch
        them from the message object else compute a one hot encoding for the label as the feature vector
        Find a training example for each label and get the encoded features from the corresponding Message object"""

        labels_example = []

        # Collect one example for each label
        # 针对每个label收集一个example
        for label_name, idx in label_id_dict.items():
            label_example = self._find_example_for_label(
                label_name, training_data.intent_examples, attribute
            )
            labels_example.append((idx, label_example))

        # Collect features, precomputed if they exist, else compute on the fly
        # 判断所有的label是不是向量化
        if self._check_labels_features_exist(labels_example, attribute_feature_name):
            # 返回编码后的labels示例
            encoded_id_labels = self._extract_labels_precomputed_features(
                labels_example, attribute_feature_name
            )
        else:
            encoded_id_labels = self._compute_default_label_features(labels_example)
        # shape为：[6,138]
        return encoded_id_labels
```

该函数返回之后，接着会调用`_create_session_data`，如下：

```python
    # noinspection PyPep8Naming
    def _create_session_data(
        self,
        training_data: "TrainingData",
        label_id_dict: Dict[Text, int],
        attribute: Text,
    ) -> "train_utils.SessionData":
        """Prepare data for training and create a SessionData object"""

        X = []
        label_ids = []
        Y = []

        for e in training_data.intent_examples:
            if e.get(attribute):
                X.append(e.get(MESSAGE_VECTOR_FEATURE_NAMES[MESSAGE_TEXT_ATTRIBUTE]))
                label_ids.append(label_id_dict[e.get(attribute)])

        X = np.array(X) # shape: 20, 418
        label_ids = np.array(label_ids) # shape: 20,

        for label_id_idx in label_ids:
            Y.append(self._encoded_all_label_ids[label_id_idx])

        Y = np.array(Y)  # shape: 20, 138
        # 创建SessionData
        return train_utils.SessionData(X=X, Y=Y, label_ids=label_ids)
```

接下来看一下构建训练网络的代码，如下：

```python
    def _build_tf_train_graph(self) -> Tuple["tf.Tensor", "tf.Tensor"]:
        # 设置入参，a_in.shape: ?, 418; b_in.shape: ?, 138
        self.a_in, self.b_in = self._iterator.get_next()

        # 设置编码的label的常量，shape：6,138
        all_label_ids = tf.constant(
            self._encoded_all_label_ids, dtype=tf.float32, name="all_label_ids"
        )

        # 构建消息输入层
        self.message_embed = self._create_tf_embed_fnn(
            self.a_in,
            self.hidden_layer_sizes["a"],
            fnn_name="a_b" if self.share_hidden_layers else "a",
            embed_name="a",
        )

        # 构建意图label输入层
        self.label_embed = self._create_tf_embed_fnn(
            self.b_in,
            self.hidden_layer_sizes["b"],
            fnn_name="a_b" if self.share_hidden_layers else "b",
            embed_name="b",
        )
        
        # 所有label的网络层
        self.all_labels_embed = self._create_tf_embed_fnn(
            all_label_ids,
            self.hidden_layer_sizes["b"],
            fnn_name="a_b" if self.share_hidden_layers else "b",
            embed_name="b",
        )

        return train_utils.calculate_loss_acc(
            self.message_embed,
            self.label_embed,
            self.b_in,
            self.all_labels_embed,
            all_label_ids,
            self.num_neg,
            None,
            self.loss_type,
            self.mu_pos,
            self.mu_neg,
            self.use_max_sim_neg,
            self.C_emb,
            self.scale_loss,
        )
```

以构建消息输入层为例，看下函数`_create_tf_embed_fnn`的实现，如下：

```python
    # tf helpers:
    def _create_tf_embed_fnn(
        self,
        x_in: "tf.Tensor",
        layer_sizes: List[int], # 256,128
        fnn_name: Text,         # 'a'
        embed_name: Text,       # 'a'
    ) -> "tf.Tensor":
        """Create nn with hidden layers and name"""

        x = train_utils.create_tf_fnn(
            x_in,
            layer_sizes,
            self.droprate,
            self.C2,
            self._is_training,
            layer_name_suffix=fnn_name,
        )
        # 在隐藏层后面再添加一层，作为输出层
        return train_utils.create_tf_embed(
            x,
            self.embed_dim,
            self.C2,
            self.similarity_type,
            layer_name_suffix=embed_name,
        )

```

`train_utils.create_tf_fnn`实现如下：

```python
# noinspection PyPep8Naming
# 这部分的代码还是比较清楚的，用来构建dense层
def create_tf_fnn(
    x_in: "tf.Tensor",
    layer_sizes: List[int],
    droprate: float,
    C2: float,
    is_training: "tf.Tensor",
    layer_name_suffix: Text,
    activation: Optional[Callable] = tf.nn.relu,
    use_bias: bool = True,
    kernel_initializer: Optional["tf.keras.initializers.Initializer"] = None,
) -> "tf.Tensor":
    """Create nn with hidden layers and name suffix."""

    reg = tf.contrib.layers.l2_regularizer(C2)
    x = tf.nn.relu(x_in)
    for i, layer_size in enumerate(layer_sizes):
        x = tf.layers.dense(
            inputs=x,
            units=layer_size,
            activation=activation,
            use_bias=use_bias,
            kernel_initializer=kernel_initializer,
            kernel_regularizer=reg,
            name="hidden_layer_{}_{}".format(layer_name_suffix, i),
            reuse=tf.AUTO_REUSE,
        )
        x = tf.layers.dropout(x, rate=droprate, training=is_training)
    return x
```

关于训练的后一部分内容由于牵涉到对于文章https://arxiv.org/abs/1709.03856的理解不足，只能给出个大概。

`sample_negative`的函数如下：

```python
def sample_negatives(
    a_embed: "tf.Tensor",  # shape(?, 20)
    b_embed: "tf.Tensor",  # shape(?, 20)
    b_raw: "tf.Tensor",    # shape(?, 138) 意图的输入
    all_b_embed: "tf.Tensor", # shape(6,20) 所有意图的embed
    all_b_raw: "tf.Tensor",# shape(6,138) 所有意图的表示
    num_neg: int, # 5
) -> Tuple[
    "tf.Tensor", "tf.Tensor", "tf.Tensor", "tf.Tensor", "tf.Tensor", "tf.Tensor"
]:
    """Sample negative examples."""

    # 其中_tf_make_flat为生成二维张量，针对当前的示例，没有实际效用
    neg_dial_embed, dial_bad_negs = _tf_get_negs(
        _tf_make_flat(a_embed), _tf_make_flat(b_raw), b_raw, num_neg
    )

    neg_bot_embed, bot_bad_negs = _tf_get_negs(all_b_embed, all_b_raw, b_raw, num_neg)
    return (
        tf.expand_dims(a_embed, -2),
        tf.expand_dims(b_embed, -2),
        neg_dial_embed,
        neg_bot_embed,
        dial_bad_negs,
        bot_bad_negs,
    )
```

`_tf_get_negs`的实现如下，注释中的代码理解可能有误，请指出：

```python
def _tf_get_negs(
    all_embed: "tf.Tensor", # shape(?, 20)
    all_raw: "tf.Tensor",  # shape(?, 138)
    raw_pos: "tf.Tensor", # shape(?, 138) IteratorGetNext
    num_neg: int # 5
) -> Tuple["tf.Tensor", "tf.Tensor"]:
    """Get negative examples from given tensor."""

    if len(raw_pos.shape) == 3:
        batch_size = tf.shape(raw_pos)[0]
        seq_length = tf.shape(raw_pos)[1]
    else:  # len(raw_pos.shape) == 2
        batch_size = tf.shape(raw_pos)[0]
        seq_length = 1

    raw_flat = _tf_make_flat(raw_pos)

    total_candidates = tf.shape(all_embed)[0]

    # 构造indices
    all_indices = tf.tile(
        tf.expand_dims(tf.range(0, total_candidates, 1), 0),
        (batch_size * seq_length, 1),
    )
    # 对indices进行重排
    shuffled_indices = tf.transpose(
        tf.random.shuffle(tf.transpose(all_indices, (1, 0))), (1, 0)
    )
    # 挑选出一部分作为neg
    neg_ids = shuffled_indices[:, :num_neg]

    # 通过交集并集操作后，求得bad_negs
    bad_negs = _tf_calc_iou_mask(raw_flat, all_raw, neg_ids)
    if len(raw_pos.shape) == 3:
        bad_negs = tf.reshape(bad_negs, (batch_size, seq_length, -1))
    # 根据neg_ids进行采样
    neg_embed = _tf_sample_neg(batch_size * seq_length, all_embed, neg_ids)
    if len(raw_pos.shape) == 3:
        neg_embed = tf.reshape(
            neg_embed, (batch_size, seq_length, -1, all_embed.shape[-1])
        )

    return neg_embed, bad_negs

```

`calculate_loss_acc`函数实现如下：

```python
# noinspection PyPep8Naming
def calculate_loss_acc(
    a_embed: "tf.Tensor",
    b_embed: "tf.Tensor",
    b_raw: "tf.Tensor",
    all_b_embed: "tf.Tensor",
    all_b_raw: "tf.Tensor",
    num_neg: int,
    mask: Optional["tf.Tensor"],
    loss_type: Text,
    mu_pos: float,
    mu_neg: float,
    use_max_sim_neg: bool,
    C_emb: float,
    scale_loss: bool,
) -> Tuple["tf.Tensor", "tf.Tensor"]:
    """Calculate loss and accuracy."""

    # 负采样？
    (
        pos_dial_embed,
        pos_bot_embed,
        neg_dial_embed,
        neg_bot_embed,
        dial_bad_negs,
        bot_bad_negs,
    ) = sample_negatives(a_embed, b_embed, b_raw, all_b_embed, all_b_raw, num_neg)

    # calculate similarities
    # 相似度计算？
    (sim_pos, sim_neg, sim_neg_bot_bot, sim_neg_dial_dial, sim_neg_bot_dial) = tf_sim(
        pos_dial_embed,
        pos_bot_embed,
        neg_dial_embed,
        neg_bot_embed,
        dial_bad_negs,
        bot_bad_negs,
        mask,
    )

    # 确定准确度计算？
    acc = tf_calc_accuracy(sim_pos, sim_neg)

    # 选择损失函数？
    loss = choose_loss(
        sim_pos,
        sim_neg,
        sim_neg_bot_bot,
        sim_neg_dial_dial,
        sim_neg_bot_dial,
        mask,
        loss_type,
        mu_pos,
        mu_neg,
        use_max_sim_neg,
        C_emb,
        scale_loss,
    )

    return loss, acc
```

## 知识点

- [negative sampling负采样和nce loss](https://blog.csdn.net/weixin_40901056/article/details/88568344)
- [StarSpace: Embed All The Things!](https://arxiv.org/abs/1709.03856)