+++
date="2019-12-21"
title="rasa source - nlu中的tokenizer实现走读"
categories=["chatbot","rasa"]
tags=["rasa-source"]
+++

# rasa source - nlu中的tokenizer实现走读

所有的tokenizer相关的代码见目录：https://github.com/RasaHQ/rasa/tree/master/rasa/nlu/tokenizers.

rasa实现了四种分词方法，分别为j

- jieba分词
- mitie分词
- spacy分词
- 空格分词

所有的分词器都继承自`Tokenizer`类和`Component`类。属于多继承，可以参见[here](https://www.liaoxuefeng.com/wiki/1016959663602400/1017502939956896)。

## jieba分词

jieba项目以及使用说明：https://github.com/fxsjy/jieba

源文件为：https://github.com/RasaHQ/rasa/tree/master/rasa/nlu/tokenizers/jieba_tokenizer.py

```python
class JiebaTokenizer(Tokenizer, Component):
    # MESSAGE_ATTRIBUTES = ["text", "intent", "response"]
	# provides = ["tokens", "intent_tokens", "response_tokens"]
    provides = [MESSAGE_TOKENS_NAMES[attribute] for attribute in MESSAGE_ATTRIBUTES]

    language_list = ["zh"]

    defaults = {
        "dictionary_path": None,
        # Flag to check whether to split intents
        "intent_tokenization_flag": False,
        # Symbol on which intent should be split
        "intent_split_symbol": "_",
    }  # default don't load custom dictionary

    def __init__(self, component_config: Dict[Text, Any] = None) -> None:
        """Construct a new intent classifier using the MITIE framework."""

        super(JiebaTokenizer, self).__init__(component_config)

        # path to dictionary file or None
        self.dictionary_path = self.component_config.get("dictionary_path")

        # flag to check whether to split intents
        self.intent_tokenization_flag = self.component_config.get(
            "intent_tokenization_flag"
        )

        # symbol to split intents on
        self.intent_split_symbol = self.component_config.get("intent_split_symbol")

        # load dictionary
        if self.dictionary_path is not None:
            self.load_custom_dictionary(self.dictionary_path)

    # 关于classmethod，staticmethod的详细介绍，参见补充知识
    @classmethod
    def required_packages(cls) -> List[Text]:
        return ["jieba"]

    # 加载自定义词典
    @staticmethod
    def load_custom_dictionary(path: Text) -> None:
        """Load all the custom dictionaries stored in the path.

        More information about the dictionaries file format can
        be found in the documentation of jieba.
        https://github.com/fxsjy/jieba#load-dictionary
        """
        import jieba

        jieba_userdicts = glob.glob("{}/*".format(path))
        for jieba_userdict in jieba_userdicts:
            logger.info("Loading Jieba User Dictionary at {}".format(jieba_userdict))
            jieba.load_userdict(jieba_userdict)

    def train(
        self, training_data: TrainingData, config: RasaNLUModelConfig, **kwargs: Any
    ) -> None:
        # example的类型是Dict
        # 针对nlu中的每句话进行分词，将分词得到的列表值设入字典中
        for example in training_data.training_examples:

            for attribute in MESSAGE_ATTRIBUTES:  # MESSAGE_ATTRIBUTES = ["text", "intent", "response"]

                if example.get(attribute) is not None:
                    example.set(
                        MESSAGE_TOKENS_NAMES[attribute],
                        self.tokenize(example.get(attribute), attribute),
                    )

    def process(self, message: Message, **kwargs: Any) -> None:

        message.set(
            MESSAGE_TOKENS_NAMES[MESSAGE_TEXT_ATTRIBUTE],
            self.tokenize(message.text, MESSAGE_TEXT_ATTRIBUTE),
        )

    def preprocess_text(self, text, attribute):
	    # 确定对意图是不是需要进行split
        if attribute == MESSAGE_INTENT_ATTRIBUTE and self.intent_tokenization_flag:
            return " ".join(text.split(self.intent_split_symbol))
        else:
            return text

    def tokenize(self, text: Text, attribute=MESSAGE_TEXT_ATTRIBUTE) -> List[Token]:
        import jieba

        text = self.preprocess_text(text, attribute)
        tokenized = jieba.tokenize(text)
        # 创建token列表
        # Token的定义很简单，用来存储单词，单词出现的偏置等基本信息
        tokens = [Token(word, start) for (word, start, end) in tokenized]
        return tokens

    @classmethod
    def load(
        cls,
        meta: Dict[Text, Any],
        model_dir: Optional[Text] = None,
        model_metadata: Optional["Metadata"] = None,
        cached_component: Optional[Component] = None,
        **kwargs: Any
    ) -> "JiebaTokenizer":

        relative_dictionary_path = meta.get("dictionary_path")

        # get real path of dictionary path, if any
        if relative_dictionary_path is not None:
            dictionary_path = os.path.join(model_dir, relative_dictionary_path)

            meta["dictionary_path"] = dictionary_path

        return cls(meta)

    @staticmethod
    def copy_files_dir_to_dir(input_dir, output_dir):
        # make sure target path exists
        if not os.path.exists(output_dir):
            os.makedirs(output_dir)

        target_file_list = glob.glob("{}/*".format(input_dir))
        for target_file in target_file_list:
            shutil.copy2(target_file, output_dir)

    def persist(self, file_name: Text, model_dir: Text) -> Optional[Dict[Text, Any]]:
        """Persist this model into the passed directory."""

        # copy custom dictionaries to model dir, if any
        if self.dictionary_path is not None:
            target_dictionary_path = os.path.join(model_dir, file_name)
            self.copy_files_dir_to_dir(self.dictionary_path, target_dictionary_path)

            return {"dictionary_path": file_name}
        else:
            return {"dictionary_path": None}
```

## mitie分词

mitie项目地址为：https://github.com/mit-nlp/MITIE

源文件为：https://github.com/RasaHQ/rasa/tree/master/rasa/nlu/tokenizers/mitie_tokenizer.py

使用了`mitie.tokenize_with_offsets`进行分词，下面主要是调用逻辑，主要函数与上面类似，train函数调用，内部调用了tokenize实现。

```python
class MitieTokenizer(Tokenizer, Component):

    provides = [MESSAGE_TOKENS_NAMES[attribute] for attribute in MESSAGE_ATTRIBUTES]

    @classmethod
    def required_packages(cls) -> List[Text]:
        return ["mitie"]

    def train(
        self, training_data: TrainingData, config: RasaNLUModelConfig, **kwargs: Any
    ) -> None:

        for example in training_data.training_examples:

            for attribute in MESSAGE_ATTRIBUTES:

                if example.get(attribute) is not None:
                    example.set(
                        MESSAGE_TOKENS_NAMES[attribute],
                        self.tokenize(example.get(attribute)),
                    )

    def process(self, message: Message, **kwargs: Any) -> None:

        message.set(
            MESSAGE_TOKENS_NAMES[MESSAGE_TEXT_ATTRIBUTE], self.tokenize(message.text)
        )

    def _token_from_offset(self, text, offset, encoded_sentence):
        return Token(
            text.decode("utf-8"), self._byte_to_char_offset(encoded_sentence, offset)
        )

    def tokenize(self, text: Text) -> List[Token]:
        import mitie

        encoded_sentence = text.encode("utf-8")
        tokenized = mitie.tokenize_with_offsets(encoded_sentence)
        tokens = [
            self._token_from_offset(token, offset, encoded_sentence)
            for token, offset in tokenized
        ]
        return tokens

    @staticmethod
    def _byte_to_char_offset(text: bytes, byte_offset: int) -> int:
        return len(text[:byte_offset].decode("utf-8"))
```

## spacy分词

源文件为：https://github.com/RasaHQ/rasa/tree/master/rasa/nlu/tokenizers/spacy_tokenizer.py

```python
class SpacyTokenizer(Tokenizer, Component):
    provides = [
        MESSAGE_TOKENS_NAMES[attribute] for attribute in SPACY_FEATURIZABLE_ATTRIBUTES
    ]

    # SPACY_FEATURIZABLE_ATTRIBUTES: ["text", "response"]
    # requires: ["spacy_doc", "response_spacy_doc"]
    requires = [
        MESSAGE_SPACY_FEATURES_NAMES[attribute]
        for attribute in SPACY_FEATURIZABLE_ATTRIBUTES
    ]

    def train(
        self, training_data: TrainingData, config: RasaNLUModelConfig, **kwargs: Any
    ) -> None:
        for example in training_data.training_examples:
            for attribute in SPACY_FEATURIZABLE_ATTRIBUTES:
                attribute_doc = self.get_doc(example, attribute)
                if attribute_doc is not None:
                    example.set(
                        MESSAGE_TOKENS_NAMES[attribute], self.tokenize(attribute_doc)
                    )

    def get_doc(self, message, attribute):
        return message.get(MESSAGE_SPACY_FEATURES_NAMES[attribute])

    def process(self, message: Message, **kwargs: Any) -> None:
        message.set(
            MESSAGE_TOKENS_NAMES[MESSAGE_TEXT_ATTRIBUTE],
            self.tokenize(self.get_doc(message, MESSAGE_TEXT_ATTRIBUTE)),
        )

    def tokenize(self, doc: "Doc") -> typing.List[Token]:
        return [Token(t.text, t.idx) for t in doc]
```

spacy分词依赖于spacynlu，其源文件为：https://github.com/RasaHQ/rasa/tree/master/rasa/nlu/utils/spacy_utils.py

此处仅通过调用`self.get_doc(example, attribute)`实现，具体的分词已经在SpacyNLU中实现完成。

spacy项目地址：https://github.com/explosion/spaCy，入门介绍：https://spacy.io/usage/spacy-101。

## 空格分词

源文件为：https://github.com/RasaHQ/rasa/tree/master/rasa/nlu/tokenizers/whitespace_tokenizer.py

详细见上一篇文章：[rasa source - nlu模型训练源码走读](https://zhuanlan.zhihu.com/p/86548776)

## 自定义分词

从上面的实现中可知，如果需要自定义分词实现`CustomTokenizer`，需要定义的函数有：`__init__`,`train`,`tokenize`，具体如下：

```python
# 自定义模板如下
class CustomTokenizer(Tokenizer, Component):
    provides = [MESSAGE_TOKENS_NAMES[attribute] for attribute in MESSAGE_ATTRIBUTES]
    
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
    
    # 主要是要实现对应的
    tokenize
    def tokenize(self, text: Text) -> List[Token]:
        pass
```

## 补充知识

- [【Python】Python3 多继承的super init()问题](https://blog.csdn.net/u013451157/article/details/78722573)
- [正确理解Python中的 @staticmethod@classmethod方法](https://zhuanlan.zhihu.com/p/28010894)
- [自然语言处理是如何工作的？一步步教你构建 NLP 流水线](https://zhuanlan.zhihu.com/p/41850756)
- [spacy主页](https://spacy.io)