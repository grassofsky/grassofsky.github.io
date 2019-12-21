+++
date="2019-12-21"
title="对话系统rasa - Policies （翻译）"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统rasa - Policies （翻译）

## Configuring Policies

`rasa.core.policies.Policy`用来确定对话过程中每个步骤将要采取什么Action。

这里有不同的policies用来选择，并且你可以将多个policies包含在一个`rasa.core.agent.Agent`中。

**注意**：针对每个用户消息，默认可预测的最大action数量为10。你可以通过设置环境变量`MAX_NUMBER_OF_PREDICTIONS`来改变这个值。

你的项目中的config.yml配置文件有一个policies关键字用来设定你的助手将会使用哪些policies。下面的例子中，最后两行是针对你的自定义的policy类以及需要的参数。

```
policies:
  - name: "KerasPolicy"
    featurizer:
    - name: MaxHistoryTrackerFeaturizer
      max_history: 5
      state_featurizer:
        - name: BinarySingleStateFeaturizer
  - name: "MemoizationPolicy"
    max_history: 5
  - name: "FallbackPolicy"
    nlu_threshold: 0.4
    core_threshold: 0.3
    fallback_action_name: "my_fallback_action"
  - name: "path.to.your.policy.class"
    arg1: "..."
```

### Max History

针对rasa core policies的一个重要的超参数是`max_history`。这个控制了当确定要采用哪个action的时候，需要考虑的对话历史的数量。

你可以在配置文件中针对Featurizer policy传入max_history参数。

**注意**：只有MaxHistoryTrackerFeaturizer使用了最大历史，而FullDialogueTrackerFeaturizer总是查看整个对话历史。具体参见：[Featurization](https://zhuanlan.zhihu.com/p/88111971)。

作为示例，假设有一个`out_of_scope`的意图用来表示用户消息脱离主题了。如果你的助手多次看到了这个意图，也许你想要告诉用户你有什么可以帮助他们。因此你的故事看上去是这样的：

```
* out_of_scope
   - utter_default
* out_of_scope
   - utter_default
* out_of_scope
   - utter_help_message
```

为了让rasa core学会这个模式，`max_history`需要至少被设置成3。

如果你增加`max_history`，你的模型将会变得更大，训练时间也会变得更常。如果你的一些消息会在未来的某个地方影响到对话，你应该将它存储成slot。slot针对每个featurizer都是可用的。

### Data Augmentation

在训练模型时，默认情况下，Rasa Core将随机地将故事文件中的故事粘在一起，从而创建更长的故事。这是因为如果你有这样的故事：

```
# thanks
* thankyou
   - utter_youarewelcome

# bye
* goodbye
   - utter_goodbye
```

实际上，你想要这样训练你的策略，在对话历史不相关的情况下忽略对话历史，不管之前发生了什么，只要用同样的行动来回应。

你可以通过使用`--augmentation`来改变这种行为。这个参数允许你设置`augmentation_factor`。`augmentation_factor`参数决定在训练的时候有多少augmented故事是子采样的。扩增的故事是在训练之前进行采样的，由于他们的数量增加的很快，因此我们需要限制它。采样的故事的数量是`augmentation_factor`的10倍。默认情况下`augmentation`是20，因此最大是200个augmentated故事。

`--augmentation 0`将禁止所有的augmentation行为。基于记忆的policies不会受这个参数的影响，会自动忽略所有扩增的故事。

## Action Selection

每一轮对话中，定义在配置文件中的每个policy在预测下一个action的时候都会有对应的置信度。想要了解关于每个policy如何做出预测的，可以阅读下面更加详细的介绍。助手会从这些预测中选择出置信度最高的结果。

如果两个策略得到相同的置信度（比如，memorization和mapping policy的置信度总是1或0），将会考虑到policies的优先级。rasa policies为各个policy设定了默认优先级。他们看上去是这样的，更大的数值意味着更高的优先级：

```
5. FormPolicy
4. FallbackPolicy and TwoStageFallbackPolicy
3. MemoizationPolicy and AugmentedMemoizationPolicy
2. MappingPolicy
1. EmbeddingPolicy, KerasPolicy, and SklearnPolicy
```

优先级设置的情况下，确保了如下行为，如果有个意图预测为mapped action，但是NLU值不高于`nlu_threshold`，bot仍然会fall back。通常情况下，不建议使用相同优先级的多个策略。

如果你创建自己的policy，使用上面的优先级顺序来指导设定你的优先级。比如，你创建了基于深度学习的策略，你的优先级应该是1，和rasa的深度学习策略的优先级保持一致。

**警告**：所有的policy的优先级都可以通过`priority:`参数进行设定，但是我们不建议除了自定义policy以外的策略上进行使用。否则可能会有意料之外的结果。

## Keras Policy

`KerasPolicy`使用了基于Keras实现的神经网络选择下一个action。默认的结构是基于LSTM试下你的，但是你可以重写KerasPolicy.model_architecture方法，来实现你自己的网络结构。

```python
def model_architecture(
    self, input_shape: Tuple[int, int], output_shape: Tuple[int, Optional[int]]
) -> tf.keras.models.Sequential:
    """Build a keras model and return a compiled model."""

    from tensorflow.keras.models import Sequential
    from tensorflow.keras.layers import (
        Masking,
        LSTM,
        Dense,
        TimeDistributed,
        Activation,
    )

    # Build Model
    model = Sequential()

    # the shape of the y vector of the labels,
    # determines which output from rnn will be used
    # to calculate the loss
    if len(output_shape) == 1:
        # y is (num examples, num features) so
        # only the last output from the rnn is used to
        # calculate the loss
        model.add(Masking(mask_value=-1, input_shape=input_shape))
        model.add(LSTM(self.rnn_size, dropout=0.2))
        model.add(Dense(input_dim=self.rnn_size, units=output_shape[-1]))
    elif len(output_shape) == 2:
        # y is (num examples, max_dialogue_len, num features) so
        # all the outputs from the rnn are used to
        # calculate the loss, therefore a sequence is returned and
        # time distributed layer is used

        # the first value in input_shape is max dialogue_len,
        # it is set to None, to allow dynamic_rnn creation
        # during prediction
        model.add(Masking(mask_value=-1, input_shape=(None, input_shape[1])))
        model.add(LSTM(self.rnn_size, return_sequences=True, dropout=0.2))
        model.add(TimeDistributed(Dense(units=output_shape[-1])))
    else:
        raise ValueError(
            "Cannot construct the model because"
            "length of output_shape = {} "
            "should be 1 or 2."
            "".format(len(output_shape))
        )

    model.add(Activation("softmax"))

    model.compile(
        loss="categorical_crossentropy", optimizer="rmsprop", metrics=["accuracy"]
    )

    if obtain_verbosity() > 0:
        model.summary()

    return model
```

训练的代码是这样的：

```python
def train(
    self,
    training_trackers: List[DialogueStateTracker],
    domain: Domain,
    **kwargs: Any,
) -> None:

    # set numpy random seed
    np.random.seed(self.random_seed)

    training_data = self.featurize_for_training(training_trackers, domain, **kwargs)
    # noinspection PyPep8Naming
    shuffled_X, shuffled_y = training_data.shuffled_X_y()

    self.graph = tf.Graph()
    with self.graph.as_default():
        # set random seed in tf
        tf.set_random_seed(self.random_seed)
        self.session = tf.compat.v1.Session(config=self._tf_config)

        with self.session.as_default():
            if self.model is None:
                self.model = self.model_architecture(
                    shuffled_X.shape[1:], shuffled_y.shape[1:]
                )

            logger.info(
                "Fitting model with {} total samples and a "
                "validation split of {}"
                "".format(training_data.num_examples(), self.validation_split)
            )

            # filter out kwargs that cannot be passed to fit
            self._train_params = self._get_valid_params(
                self.model.fit, **self._train_params
            )

            self.model.fit(
                shuffled_X,
                shuffled_y,
                epochs=self.epochs,
                batch_size=self.batch_size,
                shuffle=False,
                verbose=obtain_verbosity(),
                **self._train_params,
            )
            # the default parameter for epochs in keras fit is 1
            self.current_epoch = self.defaults.get("epochs", 1)
            logger.info("Done fitting keras policy model")
```

你可以通过重写这些方法，实现自己的模型，或者使用预定义的`keras model`初始化keraspolicy。

为了针对相同的输入得到重用的训练结果，你可以将KerasPolicy的random_seed设置成任何整数。

## Embedding Policy

Transformer Embedding Dialogue Policy。

Transformer version of the Recurrent Embedding Dialogue Policy 在我们的文章中被使用，如下：https://arxiv.org/abs/1811.11707

这个policy有预定义的架构，由以下步骤组成：

- 将用户输入（用户意图和实体），之前的系统响应，slots，针对每个时间步的active form，组成一个输入向量用于pre-transformer embedding layer
- 把它传递给transformer
- 在transformer的输出后面连接一个密集层，用来获取每个时间步的对话嵌入
- 添加一个密集来创建用于每个时间步系统响应的嵌入
- 计算对话嵌入和嵌入的系统响应之间的相似度。这个过程是基于[StarSpace](https://arxiv.org/abs/1709.03856) 的思想。

建议使用`state_featurizer=LabelTokenizerSingleStateFeaturizer(...)`（详细见：[Featurization](https://zhuanlan.zhihu.com/p/88111971)）。

**配置**：

配置文件中的配置参数可以传递给`EmbeddingPolicy`。

**警告**：传递合适的`epochs`给`EmbeddingPolicy`，否则policy将会被训练为1 epoch。

该算法包括如下超参数：

- 神经网络结构相关：
  - `hidden_layers_sizes_b`：设定system actions嵌入层之前的隐藏层大小（list），隐藏层的数量等于列表长度；
  - `transformer_size`：设定transformer中单元的数量；
  - `num_transformer_layers`：设定transformer层数；
  - `pos_encoding`：设定transformer位置编码的类型，可以是`timing`或`emb`；
  - `max_seq_length`：如果使用位置编码，则设置最大序列长度；
  - `num_heads`：设置multihead attention中heads的数量；
- training：
  - `batch_size`：设置一次前向反向计算的训练数据的数量，batch size的值越大，需要的内存空间就越大；
  - `batch_strategy`：设置batching的策略，`sequence`或`balanced`；
  - `epochs`：设置算法遇到所有训练数据的次数，一个epoch表示所有训练的数据的一次前向和后向计算；
  - `random_seed`：如果设置为任何int，将获得相同输入的可重复训练结果；
- embedding：
  - `embed_dim`：设置嵌入空间的维度；
  - `num_neg`：设置不正确的意图标签的数量，算法在训练过程中将其与用户输入的相似度最小化；
  - `similarity_type`：设置相似度计算的类型，它可以使`auto`，`cosine`或者`inner`，如果是`auto`，这个类型的设置将依赖于`loss_type`，对于`softmax`采用`inner`，针对`margin`使用`cosine`；
  - `loss_type`：设置损失函数的类型，`softmax`或`margin`；
  - `mu_pos`：控制算法应尝试为正确的意图标签生成嵌入向量的相似程度，只有到`loss_type`设置为`margin`的时候才使用；
  - `mu_neg`：控制不正确意图的最大负相似度，只有到`loss_type`设置为`margin`的时候才使用；
  - `use_max_sim_neg`：如果是真的，算法只在不正确的意图标签上最小化最大相似度，只有到`loss_type`设置为`margin`的时候才使用；
  - `scale_loss`：如果是真的，算法将缩小损失，例如以高置信度预测的正确的标签，只有到`loss_type`设置为`softmax`的时候才使用；
- regularization：
  - `C2`：L2正则化的比例
  - `C_emb`：设置最小化不同意图标签embeddings之间最大相似度的比例，只有到`loss_type`设置为`margin`的时候才使用；
  - `droprate_a`：为用户输入设置嵌入层之前层之间的dropout；
  - `droprate_b`：设置系统动作嵌入层前各层之间的dropout；
- 训练准确度计算：
  - `evaluate_every_num`：设置计算训练精度的频率，小值可能会影响性能；
  - `evaluate_on_num_examples`：从验证集中挑选出多少示例来计算验证的准确度，较大值可能会影响性能。

**警告**：这个policy默认的`max_history`值为None，意味着使用`FullDialogueTrackerFeaturizer`。我们建议设置这个值，进而使用`MaxHistoryTrackerFeaturizer`用于快速训练。我们建议增加`MaxHistoryTrackerFeaturizer` 的`batch_size`（如，`"batch_size":[32, 64]`）

**警告**：如果evaluate_on_num_examples为非零，则通过分层分割选取随机样本作为验证集，并将其排除在训练数据之外。如果数据集包含许多独特的对话转换示例，我们建议将其设置为零。

**注意**：droprate应该是0到1之间的值，例如，`droprate=0.1`，将drop out 10%的输入单元。

**注意**：针对cosine相似度，`mu_pos`和`mu_neg`的值应该在-1到1之间。

**注意**：有一个选项可以使用线性增加的批处理大小。这个想法来自于<https://arxiv.org/abs/1711.00489>. 为了使用这个功能，将`batch_size`设置成列表值，如，`"batch_size": [8, 32]`。如果`batch_size`是常量，传入int，如，`"batch_size": 8`。

这个参数可以在配置文件中设置，默认值定义在`EmbeddingPolicy.defaults`：

```python
defaults = {
    # nn architecture
    # a list of hidden layers sizes before user embed layer
    # number of hidden layers is equal to the length of this list
    "hidden_layers_sizes_pre_dial": [],
    # a list of hidden layers sizes before bot embed layer
    # number of hidden layers is equal to the length of this list
    "hidden_layers_sizes_bot": [],
    # number of units in transformer
    "transformer_size": 128,
    # number of transformer layers
    "num_transformer_layers": 1,
    # type of positional encoding in transformer
    "pos_encoding": "timing",  # string 'timing' or 'emb'
    # max sequence length if pos_encoding='emb'
    "max_seq_length": 256,
    # number of attention heads in transformer
    "num_heads": 4,
    # training parameters
    # initial and final batch sizes:
    # batch size will be linearly increased for each epoch
    "batch_size": [8, 32],
    # how to create batches
    "batch_strategy": "balanced",  # string 'sequence' or 'balanced'
    # number of epochs
    "epochs": 1,
    # set random seed to any int to get reproducible results
    "random_seed": None,
    # embedding parameters
    # dimension size of embedding vectors
    "embed_dim": 20,
    # the type of the similarity
    "num_neg": 20,
    # flag if minimize only maximum similarity over incorrect labels
    "similarity_type": "auto",  # string 'auto' or 'cosine' or 'inner'
    # the type of the loss function
    "loss_type": "softmax",  # string 'softmax' or 'margin'
    # how similar the algorithm should try
    # to make embedding vectors for correct labels
    "mu_pos": 0.8,  # should be 0.0 < ... < 1.0 for 'cosine'
    # maximum negative similarity for incorrect labels
    "mu_neg": -0.2,  # should be -1.0 < ... < 1.0 for 'cosine'
    # the number of incorrect labels, the algorithm will minimize
    # their similarity to the user input during training
    "use_max_sim_neg": True,  # flag which loss function to use
    # scale loss inverse proportionally to confidence of correct prediction
    "scale_loss": True,
    # regularization
    # the scale of L2 regularization
    "C2": 0.001,
    # the scale of how important is to minimize the maximum similarity
    # between embeddings of different labels
    "C_emb": 0.8,
    # dropout rate for dial nn
    "droprate_a": 0.1,
    # dropout rate for bot nn
    "droprate_b": 0.0,
    # visualization of accuracy
    # how often calculate validation accuracy
    "evaluate_every_num_epochs": 20,  # small values may hurt performance
    # how many examples to use for hold out validation set
    "evaluate_on_num_examples": 0,  # large values may hurt performance
}
```

**注意**：参数`mu_neg`这是成负值用来模拟原始算法中`mu_neg = mu_pos`和`use_max_sim_neg = False`的情况。详细见论文：[starspace paper](https://arxiv.org/abs/1709.03856) 。

## Mapping Policy

MappingPolicy可以直接将意图映射到actions。这个映射是通过给意图添加一个`triggers`参数实现的，如下：

```
intents:
- ask_is_not:
    triggers: action_is_bot
```

一个意图最多被映射成一个action。一旦接收到对应的意图的消息，助手会运行映射的action。然后，它将会监听下一条消息。结合下一个用户消息，将恢复正常预测行为。

如果你不想要你的意图-action的映射影响对话历史，映射的action必须返回`UserUtteranceReverted()` 事件。这将从对话历史记录中删除用户的最新消息及其之后发生的任何事件。这意味着您不应该在您的故事中包含意图-动作交互。

举个例子，如果用户在对话过程中，脱离主题，问“Are you a bot?”，你可能想在不影响下一个动作预测的情况下回答。触发的自定义action可以做任何事情，但这里有一个简单的例子，它发送一个bot语句，然后还原交互：

```python
class ActionIsBot(Action):
"""Revertible mapped action for utter_is_bot"""

def name(self):
    return "action_is_bot"

def run(self, dispatcher, tracker, domain):
    dispatcher.utter_template("utter_is_bot", tracker)
    return [UserUtteranceReverted()]
```

**注意**：如果你使用MappingPolicy来直接预测机器的语句（如，`triggers: utter_{}`），这些交互将直接出现在你的故事中，这种情况下没有`UserUtteranceReverted()`，意图和映射的utterance将会出现在对话历史中。

**注意**：MappingPolicy还负责执行默认操作action_back和action_restart以响应/back和/restart。如果您的策略示例中没有包含这些意图，则这些意图将不起作用。

## Memoization Policy

MemoizationPolicy仅仅记住了训练数据中的对话。如果确切对话出现在训练数据中，它预测下一个action的置信度为1.0，否则为None，置信度为0.0.

## Fallback Policy

如果下面几点中有一点发生，那么FallbackPolicy将用来触发[Fallback Actions](https://zhuanlan.zhihu.com/p/89057333)：

1. 意图识别的置信度低于`nlu_threshold`。
2. 排名最高的意图与排名第二的意图之间的置信度差异小于模糊阈值。
3. 没有一个对话策略的预测结果的置信度高于`core_threshold`。

**配置**：

阈值和fallback action可以在配置文件中调整：

```
policies:
  - name: "FallbackPolicy"
    nlu_threshold: 0.3
    ambiguity_threshold: 0.1
    core_threshold: 0.3
    fallback_action_name: 'action_default_fallback'
```

你也可以在你的python代码中进行配置，如：

```python
from rasa.core.policies.fallback import FallbackPolicy
from rasa.core.policies.keras_policy import KerasPolicy
from rasa.core.agent import Agent

fallback = FallbackPolicy(fallback_action_name="action_default_fallback",
                          core_threshold=0.3,
                          nlu_threshold=0.3,
                          ambiguity_threshold=0.1)

agent = Agent("domain.yml", policies=[KerasPolicy(), fallback])
```

## Two-Stage Fallback Policy

TwoStageFallbackPolicy通过尝试消除用户输入的歧义，在多个阶段处理低NLU置信度的情况。

- 如果一个NLU预测获取一个低置信度值，或者排名第一个和第二个的差别不大，要求用户意图的分类进行确认。
  - 如果他们确认，story正常执行
  - 如果他们拒绝，用户被要求改变说辞。
- 改变说法
  - 如果此时的意图能够正确识别，story继续正常执行
  - 如果改变说法后的意图仍然没有得到一个高的置信度值，用户再次被询问对识别的意图进行确认。
- 第二次确认
  - 如果用户确认意图，story继续正常执行
  - 如果用户拒绝，原始意图会被识别成`deny_suggestion_intent_name`，将触发最终回退操作（例如，切换到人）。

**配置**：

为了使用 `TwoStageFallbackPolicy`，需要在你的配置文件中添加以下内容：

```
policies:
  - name: TwoStageFallbackPolicy
    nlu_threshold: 0.3
    ambiguity_threshold: 0.1
    core_threshold: 0.3
    fallback_core_action_name: "action_default_fallback"
    fallback_nlu_action_name: "action_default_fallback"
    deny_suggestion_intent_name: "out_of_scope"
```

**注意**：你可以在你的配置文件中引入`FallbackPolicy`或者`TwoStageFallbackPolicy`，但是不能一下子引入两个。

## Form Policy

FormPolicy是对MemorizationPolicy的扩展，用来处理forms填充的场景。一旦FormAction被调用，FormPolicy会持续预测FormAction，直到所有需要的slots被填满。更多的信息，见[Forms](https://zhuanlan.zhihu.com/p/84441651)。

## 原文链接

https://rasa.com/docs/rasa/core/policies/