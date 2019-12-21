+++
date="2019-12-21"
title="rasa source - nlu模型训练源码走读"
categories=["chatbot","rasa"]
tags=["rasa-source"]
+++

# rasa source - nlu模型训练源码走读

执行命令为：`rasa train nlu`

入口函数为：`rasa/cli/trian.py: train_nlu`，该函数内部调用了`rasa/train.py: train_nlu`，其实现如下：

```python
def train_nlu(
    config: Text,    # config.yml
    nlu_data: Text,  # data
    output: Text,    # models
    train_path: Optional[Text] = None,
    fixed_model_name: Optional[Text] = None,
) -> Optional[Text]:
    """Trains an NLU model.

    Args:
        config: Path to the config file for NLU.
        nlu_data: Path to the NLU training data.
        output: Output path.
        train_path: If `None` the model will be trained in a temporary
            directory, otherwise in the provided directory.
        fixed_model_name: Name of the model to be stored.
        uncompress: If `True` the model will not be compressed.

    Returns:
        If `train_path` is given it returns the path to the model archive,
        otherwise the path to the directory with the trained model files.

    """

    # 利用协程进行处理
    loop = asyncio.get_event_loop()
    return loop.run_until_complete(
        _train_nlu_async(config, nlu_data, output, train_path, fixed_model_name)
    )
```

`_train_nlu_async`函数定义如下：

```python
# rasa/train.py:390
async def _train_nlu_async(
    config: Text,
    nlu_data: Text,
    output: Text,
    train_path: Optional[Text] = None,
    fixed_model_name: Optional[Text] = None,
    persist_nlu_training_data: bool = False,
):
    # training NLU only hence the training files still have to be selected
    # 创建数据加载的importer
    file_importer = TrainingDataImporter.load_nlu_importer_from_config(
        config, training_data_paths=[nlu_data]
    )

    # 加载nlu data
    # 返回TraingData对象
    training_datas = await file_importer.get_nlu_data()
    if training_datas.is_empty():
        print_error(
            "No NLU data given. Please provide NLU data in order to train "
            "a Rasa NLU model using the '--nlu' argument."
        )
        return

    return await _train_nlu_with_validated_data(
        file_importer,
        output=output,
        train_path=train_path,
        fixed_model_name=fixed_model_name,
        persist_nlu_training_data=persist_nlu_training_data,
    )
```

`_train_nlu_with_validated_data`函数定义如下：

```python
# rasa/train.py: 420
async def _train_nlu_with_validated_data(
    file_importer: TrainingDataImporter,
    output: Text,
    train_path: Optional[Text] = None,
    fixed_model_name: Optional[Text] = None,
    persist_nlu_training_data: bool = False,
) -> Optional[Text]:
    """Train NLU with validated training and config data."""

    import rasa.nlu.train

    # 简单的来见，ExitStack是一个上下文管理器，在退出的时候会进行清理。
    # 可以参见：https://juejin.im/post/5c7b462cf265da2d97110f50
    with ExitStack() as stack:
        if train_path:
            # If the train path was provided, do nothing on exit.
            _train_path = train_path
        else:
            # Otherwise, create a temp train path and clean it up on exit.
            _train_path = stack.enter_context(TempDirectoryPath(tempfile.mkdtemp()))
        # config value is:
        # {
        #  'language': 'en', 
        #  'pipeline': 'supervised_embeddings', 
        #  'policies': [
        #      {'name': 'MemoizationPolicy'}, 
        #      {'name': 'KerasPolicy'}, 
        #      {'name': 'MappingPolicy'}]
        # }   
        config = await file_importer.get_config()
        print_color("Training NLU model...", color=bcolors.OKBLUE)
        # 具体执行训练的函数
        _, nlu_model, _ = await rasa.nlu.train(
            config,      # 见上面的注释
            file_importer, # NluDataImporter object
            _train_path,  # 临时目录： /tmp/tmp79v61479
            fixed_model_name="nlu",
            persist_nlu_training_data=persist_nlu_training_data, # False
        )
        print_color("NLU model training completed.", color=bcolors.OKBLUE)

        if train_path is None:
            # Only NLU was trained
            new_fingerprint = await model.model_fingerprint(file_importer)

            # 保存模型
            return model.package_model(
                fingerprint=new_fingerprint,
                output_directory=output,
                train_path=_train_path,
                fixed_model_name=fixed_model_name,
                model_prefix="nlu-",
            )

        return _train_path
```

`rasa.nlu.train`实现如下：

```python
# rasa/nlu/train.py: 48
async def train(
    nlu_config: Union[Text, Dict, RasaNLUModelConfig],    # 见上面代码片段中的config
    data: Union[Text, "TrainingDataImporter"],            # NluDataImporter object
    path: Optional[Text] = None,                          # /tmp/tmp79v61479
    fixed_model_name: Optional[Text] = None,              # 'nlu'
    storage: Optional[Text] = None,
    component_builder: Optional[ComponentBuilder] = None,
    training_data_endpoint: Optional[EndpointConfig] = None,
    persist_nlu_training_data: bool = False,
    **kwargs: Any
) -> Tuple[Trainer, Interpreter, Optional[Text]]:
    """Loads the trainer and the data and runs the training of the model."""
    from rasa.importers.importer import TrainingDataImporter

    if not isinstance(nlu_config, RasaNLUModelConfig):
        # 更新后值变成如下形式，将pipeline模板进行展开：
        # [('language', 'en'), 
        #  ('pipeline', [
        #                 {'name': 'WhitespaceTokenizer'}, 
        #                 {'name': 'RegexFeaturizer'}, 
        #                 {'name': 'CRFEntityExtractor'}, 
        #                 {'name': 'EntitySynonymMapper'}, 
        #                 {'name': 'CountVectorsFeaturizer'}, 
        #                 {'name': 'CountVectorsFeaturizer', 'analyzer': 'char_wb', 'min_ngram': 1, 'max_ngram': 4}, 
        #                 {'name': 'EmbeddingIntentClassifier'}
        #               ]
        #  ), 
        #  ('data', None), 
        #  ('policies', [{'name': 'MemoizationPolicy'}, 
        #                {'name': 'KerasPolicy'}, {'name': 'MappingPolicy'}
        #               ]
        #  )]   
        nlu_config = config.load(nlu_config)

    # Ensure we are training a model that we can save in the end
    # WARN: there is still a race condition if a model with the same name is
    # trained in another subprocess
    # 构建训练对象，关于Trianer类，在后面给出更详细的解释
    trainer = Trainer(nlu_config, component_builder)
    persistor = create_persistor(storage)   # 没有使用云端持久存储，persistor为None
    if training_data_endpoint is not None:  # training_data_endpoint为None
        training_data = await load_data_from_endpoint(
            training_data_endpoint, nlu_config.language
        )
    elif isinstance(data, TrainingDataImporter):
        training_data = await data.get_nlu_data(nlu_config.data)  # 加载训练数据
    else:
        training_data = load_data(data, nlu_config.language)

    # print_stats输出的结果如下：
    # - Training data stats: 
    #       - intent examples: 43 (7 distinct intents)    
    #       - Found intents: 'greet', 'mood_unhappy', 'mood_great', 'bot_challenge', 'goodbye', 'affirm', 'deny' 
    #       - Number of response examples: 0 (0 distinct response)
    #       - entity examples: 0 (0 distinct entities) 
    #       - found entities:   
    training_data.print_stats()
    interpreter = trainer.train(training_data, **kwargs)

    if path:
        persisted_path = trainer.persist(
            path, persistor, fixed_model_name, persist_nlu_training_data
        )
    else:
        persisted_path = None

    return trainer, interpreter, persisted_path
```

关于Trainer类的初始化构建如下：

```python
# rasa/nlu/model.py:129
class Trainer(object):
    """Trainer will load the data and train all components.

    Requires a pipeline specification and configuration to use for
    the training."""

    # Officially supported languages (others might be used, but might fail)
    SUPPORTED_LANGUAGES = ["de", "en"]

    def __init__(
        self,
        cfg: RasaNLUModelConfig,
        component_builder: Optional[ComponentBuilder] = None,
        skip_validation: bool = False,
    ):

        self.config = cfg
        self.skip_validation = skip_validation
        self.training_data = None  # type: Optional[TrainingData]

        if component_builder is None:
            # If no builder is passed, every interpreter creation will result in
            # a new builder. hence, no components are reused.
            # 用于组件创建以及对组件进行缓存
            component_builder = components.ComponentBuilder()

        # Before instantiating the component classes, lets check if all
        # required packages are available
        if not self.skip_validation:
            # 用来校验pipeline中使用的各个组件的依赖是否完整
            components.validate_requirements(cfg.component_names)

        # build pipeline
        self.pipeline = self._build_pipeline(cfg, component_builder)

    @staticmethod    # staticmethod指的是静态成员函数，直接通过类名进行调用
    def _build_pipeline(
        cfg: RasaNLUModelConfig, component_builder: ComponentBuilder
    ) -> List[Component]:
        """Transform the passed names of the pipeline components into classes"""
        pipeline = []

        # Transform the passed names of the pipeline components into classes
        for i in range(len(cfg.pipeline)):
            component_cfg = cfg.for_component(i)
            # 如果组件已经被创建过，那么直接返回缓存，
            # 否则新创建
            component = component_builder.create_component(component_cfg, cfg)
            pipeline.append(component)

        return pipeline
    
    # ......
```

`trainer.train`的实现如下：

```python
# rasa/nlu/model.py:129
class Trainer(object):
    """Trainer will load the data and train all components.

    Requires a pipeline specification and configuration to use for
    the training."""
    
    # ....
    def train(self, data: TrainingData, **kwargs: Any) -> "Interpreter":
        """Trains the underlying pipeline using the provided training data."""

        self.training_data = data

        self.training_data.validate()

        # 用字典来存储上下文
        context = kwargs

        for component in self.pipeline:
            # 针对新的pipeline，初始化component，大多数组件并不需要实现这个接口，
            # 大多数用户MITIE和spacy的框架环境变量的初始化
            updates = component.provide_context()
            if updates:
                context.update(updates)

        # Before the training starts: check that all arguments are provided
        if not self.skip_validation:
            # 进行校验
            components.validate_arguments(self.pipeline, context)
            components.validate_required_components_from_data(
                self.pipeline, self.training_data
            )

        # data gets modified internally during the training - hence the copy
        working_data = copy.deepcopy(data)

        # 由之前的代码分析，知道，相关的component有：
        #  ('pipeline', [
        #                 {'name': 'WhitespaceTokenizer'}, 
        #                 {'name': 'RegexFeaturizer'}, 
        #                 {'name': 'CRFEntityExtractor'}, 
        #                 {'name': 'EntitySynonymMapper'}, 
        #                 {'name': 'CountVectorsFeaturizer'}, 
        #                 {'name': 'CountVectorsFeaturizer', 'analyzer': 'char_wb', 'min_ngram': 1, 'max_ngram': 4}, 
        #                 {'name': 'EmbeddingIntentClassifier'}
        #               ]
        #  )
        for i, component in enumerate(self.pipeline):
            logger.info("Starting to train component {}".format(component.name))
            # 每个组件都继承于Component类
            # 将该component之前执行过的component和context传入
            component.prepare_partial_processing(self.pipeline[:i], context)
            # 实际训练的代码
            updates = component.train(working_data, self.config, **context)
            logger.info("Finished training component.")
            if updates:
                context.update(updates)

        return Interpreter(self.pipeline, context)

```

关于组件的介绍可以参见：[对话系统Rasa - 组件](https://zhuanlan.zhihu.com/p/83566179)

`WhitespaceTokenizer`类的主要函数为`tokenize`：

```python
# rasa/nlu/tokenizers/Whitespace_tokenizer.py: 44
    # 对 training_data内部的training_example对象进行了更新
    def train(
        self, training_data: TrainingData, config: RasaNLUModelConfig, **kwargs: Any
    ) -> None:
        for example in training_data.training_examples:
            for attribute in MESSAGE_ATTRIBUTES:
                if example.get(attribute) is not None:
                    example.set(
                        MESSAGE_TOKENS_NAMES[attribute],
                        self.tokenize(example.get(attribute), attribute),
                    )
                    
# rasa/nlu/tokenizers/Whitespace_tokenizer.py: 61
    def tokenize(
        self, text: Text, attribute: Text = MESSAGE_TEXT_ATTRIBUTE
    ) -> List[Token]:

        if not self.case_sensitive:
            text = text.lower()
        # remove 'not a word character' if
        if attribute != MESSAGE_INTENT_ATTRIBUTE:
            words = re.sub(
                # there is a space or an end of a string after it
                # 不包含字母或数字，#，@，&的字符串，后面跟着空格或结尾
                # 如：....\s
                r"[^\w#@&]+(?=\s|$)|"
                # there is a space or beginning of a string before it
                # not followed by a number
                # 空格或一行的开头，后面跟着不包含字母或数字，#，@，&的字符串，后面跟着不是数字，不是空格
                # 如：\s...nihao
                r"(\s|^)[^\w#@&]+(?=[^0-9\s])|"
                # not in between numbers and not . or @ or & or - or #
                # e.g. 10'000.00 or blabla@gmail.com
                # and not url characters
                r"(?<=[^0-9\s])[^\w._~:/?#\[\]()@!$&*+,;=-]+(?=[^0-9\s])",
                " ",
                text,
            ).split()
        else:
            # 针对意图，由配置控制是否需要分词，需要的话直接根据空格分词
            words = (
                text.split(self.intent_split_symbol)
                if self.intent_tokenization_flag
                else [text]
            )

        # 记录token
        running_offset = 0
        tokens = []
        for word in words:
            # 记录running_offset用来提高index效率
            word_offset = text.index(word, running_offset)
            word_len = len(word)
            running_offset = word_offset + word_len
            tokens.append(Token(word, word_offset))
        return tokens
```

`RegexFeaturizer`的train实现如下：

```python
# rasa/nlu/featurizers/regex_featurizer.py：40
    # 由于没有提供regex_pattern，因此此处设置均为None
    def train(
        self, training_data: TrainingData, config: RasaNLUModelConfig, **kwargs: Any
    ) -> None:

        self.known_patterns = training_data.regex_features
        self._add_lookup_table_regexes(training_data.lookup_tables)

        for example in training_data.training_examples:
            updated = self._text_features_with_regex(example)
            example.set(MESSAGE_VECTOR_FEATURE_NAMES[MESSAGE_TEXT_ATTRIBUTE], updated)

```

`CRFEntityExtractor`，条件随机场可以参见https://zhuanlan.zhihu.com/p/25558273。默认示例中，并没有定义实体，该组件直接返回，后面再进行介绍。同样，`EntitySynonymMapper`也并有执行具体内容，直接返回，后面再进行介绍。

`CountVectorsFeaturizer`的实现如下：

```python
# rasa/nlu/featurizers/count_vectors_featurizer.py：485
    def train(
        self, training_data: TrainingData, cfg: RasaNLUModelConfig = None, **kwargs: Any
    ) -> None:
        """Train the featurizer.

        Take parameters from config and
        construct a new count vectorizer using the sklearn framework.
        """

        spacy_nlp = kwargs.get("spacy_nlp") # None
        if spacy_nlp is not None:
            # create spacy lemma_ for OOV_words
            self.OOV_words = [t.lemma_ for w in self.OOV_words for t in spacy_nlp(w)]

        # process sentences and collect data for all attributes
        # 具体内容：
        # {'text': ['hey', 'hello', 'hi', 'good morning', 'good evening', 'hey there', 'bye', 'goodbye', 'see you around', 'see you later', 'yes', 'indeed', 'of course', 'that sounds good', 'correct', 'no', 'never', 'i don t think so', 'don t like that', 'no way', 'not really', 'perfect', 'very good', 'great', 'amazing', 'wonderful', 'i am feeling very good', 'i am great', 'i m good', 'sad', 'very sad', 'unhappy', 'bad', 'very bad', 'awful', 'terrible', 'not very good', 'extremely sad', 'so sad', 'are you a bot', 'are you a human', 'am i talking to a bot', 'am i talking to a human'], 
        #  'intent': ['greet', 'greet', 'greet', 'greet', 'greet', 'greet', 'goodbye', 'goodbye', 'goodbye', 'goodbye', 'affirm', 'affirm', 'affirm', 'affirm', 'affirm', 'deny', 'deny', 'deny', 'deny', 'deny', 'deny', 'mood_great', 'mood_great', 'mood_great', 'mood_great', 'mood_great', 'mood_great', 'mood_great', 'mood_great', 'mood_unhappy', 'mood_unhappy', 'mood_unhappy', 'mood_unhappy', 'mood_unhappy', 'mood_unhappy', 'mood_unhappy', 'mood_unhappy', 'mood_unhappy', 'mood_unhappy', 'bot_challenge', 'bot_challenge', 'bot_challenge', 'bot_challenge'], 
        #  'response': ['', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '']} 
        processed_attribute_texts = self._get_all_attributes_processed_texts(
            training_data
        )

        # 针对processed_attribute_texts中的内容使用，
        # sklearn.feature_extraction.text.CountVectorizer进行编码
        # train for all attributes
        if self.use_shared_vocab: # false
            self._train_with_shared_vocab(processed_attribute_texts)
        else:
            self._train_with_independent_vocab(processed_attribute_texts)

        # transform for all attributes
        for attribute in MESSAGE_ATTRIBUTES:

            attribute_features = self._get_featurized_attribute(
                attribute, processed_attribute_texts[attribute]
            )

            # 将编码后的特征设置到训练数据中
            if attribute_features is not None:
                self._set_attribute_features(
                    attribute, attribute_features, training_data
                )
```

`EmbeddingIntentClassifier`使用神经网络进行意图分类。详细介绍见后续内容。

## 补充知识点

- 协程：https://www.liaoxuefeng.com/wiki/1016959663602400/1017968846697824
- yield关键字以及next，send函数：https://blog.csdn.net/mingC0758/article/details/53783001
- asyncio：https://www.liaoxuefeng.com/wiki/1016959663602400/1017970488768640
- async：https://www.liaoxuefeng.com/wiki/1016959663602400/1048430311230688
- contextlib：https://juejin.im/post/5c7b462cf265da2d97110f50
- 正则化表达式：https://www.runoob.com/python/python-reg-expressions.html
- 前向肯定界定符：https://blog.csdn.net/hanli1992/article/details/86591196
- 条件随机场：https://zhuanlan.zhihu.com/p/25558273
- countvector：https://blog.csdn.net/weixin_38278334/article/details/82320307