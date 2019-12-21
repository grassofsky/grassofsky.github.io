+++
date="2019-12-21"
title="rasa source - core训练源码走读"
categories=["chatbot","rasa"]
tags=["rasa-source"]
+++

# rasa source - core训练源码走读

通过在`rasa/cli/train.py: train_core`函数中添加断点，使用pudb进行python调试调试跟踪的方式进行源码走读，pudb详细见：https://documen.tician.de/pudb。

刚开始的几个调用函数如下：

- `rasa/cli/train.py: train_core`
- 调用`rasa/train.py:train_core`
- 调用`rasa/train.py:train_core_async`
- 调用`rasa/train.py:_train_core_with_validated_data`
- 调用`rasa/core/train.py：train`

`rasa/core/train.py：train`实现的代码如下：

```python
async def train(
    domain_file: Union[Domain, Text],
    training_resource: Union[Text, "TrainingDataImporter"],
    output_path: Text,
    interpreter: Optional["NaturalLanguageInterpreter"] = None,
    endpoints: "AvailableEndpoints" = None,
    dump_stories: bool = False,
    policy_config: Optional[Union[Text, Dict]] = None,
    exclusion_percentage: int = None,
    kwargs: Optional[Dict] = None,
):
    from rasa.core.agent import Agent
    from rasa.core import config, utils
    from rasa.core.utils import AvailableEndpoints

    if not endpoints:
        endpoints = AvailableEndpoints()

    if not kwargs:
        kwargs = {}
	
    # nlu训练的时候加载的是component，core训练的时候需要加载的是policies
    # 关于policies更多的介绍可以参见：https://rasa.com/docs/rasa/core/policies/
    policies = config.load(policy_config)

    # Agent类为rasa最重要的功能提供了便捷的接口，包括训练，消息处理，加载对话模型，
    # 预测下一个action，处理一个channel
    agent = Agent(
        domain_file,
        generator=endpoints.nlg,
        action_endpoint=endpoints.action,
        interpreter=interpreter,
        policies=policies,
    )

    # data_load_args: {'augmentation_factor': 50, 'debug_plots':False}
    # kwargs: {'dump_stories': False}
    data_load_args, kwargs = utils.extract_args(
        kwargs,
        {
            "use_story_concatenation",
            "unique_last_num_states",
            "augmentation_factor",
            "remove_duplicates",
            "debug_plots",
        },
    )

    training_data = await agent.load_data(
        training_resource, exclusion_percentage=exclusion_percentage, **data_load_args
    )
    agent.train(training_data, **kwargs)
    agent.persist(output_path, dump_stories)

    return agent
```

正如上面代码注释中所述，Agent类为rasa最重要的功能提供了便捷的接口，包括训练，消息处理，加载对话模型， 预测下一个action，处理一个channel等。下面主要介绍Agent的一些实现。其构造函数如下：

```python
    def __init__(
        self,
        domain: Union[Text, Domain, None] = None,
        policies: Union[PolicyEnsemble, List[Policy], None] = None,
        interpreter: Optional[NaturalLanguageInterpreter] = None,
        generator: Union[EndpointConfig, NaturalLanguageGenerator, None] = None,
        tracker_store: Optional[TrackerStore] = None,
        lock_store: Optional[LockStore] = None,
        action_endpoint: Optional[EndpointConfig] = None,
        fingerprint: Optional[Text] = None,
        model_directory: Optional[Text] = None,
        model_server: Optional[EndpointConfig] = None,
        remote_storage: Optional[Text] = None,
    ):
        # Initializing variables with the passed parameters.
        self.domain = self._create_domain(domain)
        # 创建policies的集合
        self.policy_ensemble = self._create_ensemble(policies)

        if self.domain is not None:
            self.domain.add_requested_slot()
            self.domain.add_knowledge_base_slots()

        PolicyEnsemble.check_domain_ensemble_compatibility(
            self.policy_ensemble, self.domain
        )

        # 创建自然语言解释对象，此处创建的RegexInterpreter，用正则化获取实体
        # 另一个类是RasaNLUInterpreter，使用的是rasa.nlu.model.Interpreter
        self.interpreter = NaturalLanguageInterpreter.create(interpreter)

        # 创建自然有语言生成对象，根据对话状态生成机器对话，此处生成的是TemplateNaturalLanguageGenerator,
        # 是基于模板匹配的实现方式
        self.nlg = NaturalLanguageGenerator.create(generator, self.domain)
        # 默认创建InMemoryTrackerStore，对话历史都存储在内存中，
        # 其他的还有RedisTrackerStore，DynamoTrackerStore，MongoTrackerStore，SQLTrackerStore
        self.tracker_store = self.create_tracker_store(tracker_store, self.domain)
        # 用于存储的时候加锁
        self.lock_store = self._create_lock_store(lock_store)
        # 连接action的终端
        self.action_endpoint = action_endpoint

        self._set_fingerprint(fingerprint)
        # 此处下面的变量均为None
        self.model_directory = model_directory
        self.model_server = model_server
        self.remote_storage = remote_storage
```

`agent.load_data`内部调用了`rasa/core/training/__init__.py: load_data`，具体如下：

```python
async def load_data(
    resource_name: Union[Text, "TrainingDataImporter"],  # CoreDataImporter
    domain: "Domain",                                    # Domain
    remove_duplicates: bool = True,                      # True
    unique_last_num_states: Optional[int] = None,        # 5
    augmentation_factor: int = 50,                       # 50
    tracker_limit: Optional[int] = None,
    use_story_concatenation: bool = True,
    debug_plots=False,
    exclusion_percentage: int = None,
) -> List["DialogueStateTracker"]:
    from rasa.core.training.generator import TrainingDataGenerator
    from rasa.importers.importer import TrainingDataImporter

    if resource_name:
        if isinstance(resource_name, TrainingDataImporter):
            graph = await resource_name.get_stories(
                exclusion_percentage=exclusion_percentage
            )
        else:
            graph = await extract_story_graph(
                resource_name, domain, exclusion_percentage=exclusion_percentage
            )

        # 构建训练数据生成器
        g = TrainingDataGenerator(
            graph,
            domain,
            remove_duplicates,
            unique_last_num_states,
            augmentation_factor,
            tracker_limit,
            use_story_concatenation,
            debug_plots,
        )
        return g.generate()
    else:
        return []
```

`resource_name.get_stories`调用了`rasa/importers/importer.py:CoreDataImporter.get_stories`，接着调用了`rasa/importers/importer.py:CombinedDataImporter.get_stories`，然后调用了`rasa/importers/rasa.py:RasaFileImporter.get_stories`，来获取StoryGraph对象。用图的形式来存储stories，这类似于使用命令`rasa visualize`看到的样子。

训练数据生成完成之后，调用的是`agent.train`，用来进行训练，该函数调用了`rasa/core/agent.py:train`函数，对于该示例，`rasa/core/agent.py:train`函数执行的是`self.ensemble.train`。该函数位于`rasa/core/policies/ensemble.py`实现如下：

```python
    def train(
        self,
        training_trackers: List[DialogueStateTracker],
        domain: Domain,
        **kwargs: Any
    ) -> None:
        if training_trackers:
            for policy in self.policies:
                # 对每一个policy进行训练
                policy.train(training_trackers, domain, **kwargs)
        else:
            logger.info("Skipped training, because there are no training samples.")
        self.training_trackers = training_trackers
        self.date_trained = datetime.now().strftime("%Y%m%d-%H%M%S")
```

在policy训练过程中的x，y的训练数据，没别对应对话状态和针对状态需要的action。

关于每个policy的训练函数的实现，在这里不展开了。