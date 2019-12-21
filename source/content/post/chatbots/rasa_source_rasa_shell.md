+++
date="2019-12-21"
title="rasa source - rasa shell 代码走读"
categories=["chatbot","rasa"]
tags=["rasa-source"]
+++

# rasa source - rasa shell 代码走读

在`rasa/run.py: run`函数中添加断点`pudb.set_trace()`。运行示例，`examples/knowledgebasebot`。

相关的函数调用栈：

```
- rasa/run.py: run
  - rasa/core/run.py: serve_application
    - create_http_input_channels
      - `_create_single_channel` - 创建CmdlineInput channel，CmdlineInput的主要实现均在于基类中
    - configure_app
```

首先来看一下configure_app函数，如下：

```python
def configure_app(
    input_channels: Optional[List["InputChannel"]] = None,
    cors: Optional[Union[Text, List[Text], None]] = None,
    auth_token: Optional[Text] = None,
    enable_api: bool = True,
    jwt_secret: Optional[Text] = None,
    jwt_method: Optional[Text] = None,
    route: Optional[Text] = "/webhooks/",
    port: int = constants.DEFAULT_SERVER_PORT,
    endpoints: Optional[AvailableEndpoints] = None,
    log_file: Optional[Text] = None,
):
    """Run the agent."""
    from rasa import server

    rasa.core.utils.configure_file_logging(logger, log_file)

    if enable_api:  # False
        app = server.create_app(
            cors_origins=cors,
            auth_token=auth_token,
            jwt_secret=jwt_secret,
            jwt_method=jwt_method,
            endpoints=endpoints,
        )
    else:
        # 创建Sanic对象
        app = _create_app_without_api(cors)

    if input_channels:
        # 将input_channels注册到app
        channels.channel.register(input_channels, app, route=route)
    else:
        input_channels = []

    if logger.isEnabledFor(logging.DEBUG):
        rasa.core.utils.list_routes(app)

    # configure async loop logging
    async def configure_async_logging():
        if logger.isEnabledFor(logging.DEBUG):
            rasa.utils.io.enable_async_loop_debugging(asyncio.get_event_loop())

    # 添加task，在app起来之后会被调用
    app.add_task(configure_async_logging)

    if "cmdline" in {c.name() for c in input_channels}:

        async def run_cmdline_io(running_app: Sanic):
            """Small wrapper to shut down the server once cmd io is done."""
            await asyncio.sleep(1)  # allow server to start
            await console.record_messages(
                server_url=constants.DEFAULT_SERVER_FORMAT.format("http", port)
            )

            logger.info("Killing Sanic server now.")
            running_app.stop()  # kill the sanic serverx
	    # 添加shell运行的关键函数，run_cmdline_io
        app.add_task(run_cmdline_io)

    return app
```

上面的函数添加了两个task，在app运行起来之后会被调用。

接着来看一下serve_application，如下：

```python
def serve_application(
    model_path: Optional[Text] = None,
    channel: Optional[Text] = None,
    port: int = constants.DEFAULT_SERVER_PORT,
    credentials: Optional[Text] = None,
    cors: Optional[Union[Text, List[Text]]] = None,
    auth_token: Optional[Text] = None,
    enable_api: bool = True,
    jwt_secret: Optional[Text] = None,
    jwt_method: Optional[Text] = None,
    endpoints: Optional[AvailableEndpoints] = None,
    remote_storage: Optional[Text] = None,
    log_file: Optional[Text] = None,
    ssl_certificate: Optional[Text] = None,
    ssl_keyfile: Optional[Text] = None,
    ssl_ca_file: Optional[Text] = None,
    ssl_password: Optional[Text] = None,
):
    from rasa import server

    if not channel and not credentials:
        channel = "cmdline"

    # 创建的是CmdlineInput，该类的主要实现位于RestInput类中
    input_channels = create_http_input_channels(channel, credentials)

    # 创建sanic app
    app = configure_app(
        input_channels,
        cors,
        auth_token,
        enable_api,
        jwt_secret,
        jwt_method,
        port=port,
        endpoints=endpoints,
        log_file=log_file,
    )

    ssl_context = server.create_ssl_context(
        ssl_certificate, ssl_keyfile, ssl_ca_file, ssl_password
    )
    protocol = "https" if ssl_context else "http"

    logger.info(
        "Starting Rasa server on "
        "{}".format(constants.DEFAULT_SERVER_FORMAT.format(protocol, port))
    )

    # 注册事件，当服务启动之前会被调用
    app.register_listener(
        partial(load_agent_on_start, model_path, endpoints, remote_storage),
        "before_server_start",
    )

    # noinspection PyUnresolvedReferences
    async def clear_model_files(_app: Sanic, _loop: Text) -> None:
        if app.agent.model_directory:
            shutil.rmtree(_app.agent.model_directory)

    app.register_listener(clear_model_files, "after_server_stop")

    rasa.utils.common.update_sanic_log_level(log_file)

    # 启动服务
    app.run(
        host="0.0.0.0",
        port=port,
        ssl=ssl_context,
        backlog=int(os.environ.get(ENV_SANIC_BACKLOG, "100")),
        workers=rasa.core.utils.number_of_sanic_workers(
            endpoints.lock_store if endpoints else None
        ),
    )
```

当服务启动之前会调用，load_agent_on_start，让我们来看一下这个函数到底干了些什么：

```python
# noinspection PyUnusedLocal
async def load_agent_on_start(
    model_path: Text,
    endpoints: AvailableEndpoints,
    remote_storage: Optional[Text],
    app: Sanic,
    loop: Text,
):
    """Load an agent.

    Used to be scheduled on server start
    (hence the `app` and `loop` arguments)."""
    import rasa.core.brokers.utils as broker_utils

    # noinspection PyBroadException
    try:
        # 从model目录中加载模型
        with model.get_model(model_path) as unpacked_model:
            # 获取nlu模型
            _, nlu_model = model.get_model_subdirectories(unpacked_model)
            # 根据nlu模型创建NaturalLanguageInterpreter
            # 此处创建的为RasaNLUInterpreter
            _interpreter = NaturalLanguageInterpreter.create(nlu_model, endpoints.nlu)
    except Exception:
        logger.debug("Could not load interpreter from '{}'.".format(model_path))
        _interpreter = None

    # 事件代理相关，此处无
    _broker = broker_utils.from_endpoint_config(endpoints.event_broker)
    # 创建TrackerStore，此处为默认的InMemoryTrackerStore
    _tracker_store = TrackerStore.find_tracker_store(
        None, endpoints.tracker_store, _broker
    )
    # 对于的lock为InMemoryLockStore
    _lock_store = LockStore.find_lock_store(endpoints.lock_store)

    model_server = endpoints.model if endpoints and endpoints.model else None

    # 从模型中创建agent对象
    app.agent = await agent.load_agent(
        model_path,
        model_server=model_server,
        remote_storage=remote_storage,
        interpreter=_interpreter,
        generator=endpoints.nlg,
        tracker_store=_tracker_store,
        lock_store=_lock_store,
        action_endpoint=endpoints.action,
    )

    if not app.agent:
        logger.warning(
            "Agent could not be loaded with the provided configuration. "
            "Load default agent without any model."
        )
        app.agent = Agent(
            interpreter=_interpreter,
            generator=endpoints.nlg,
            tracker_store=_tracker_store,
            action_endpoint=endpoints.action,
            model_server=model_server,
            remote_storage=remote_storage,
        )

    return app.agent
```

当app运行起来之后，就会运行之前添加的task，下面再回头来看下run_cmdline_io：

```python
		async def run_cmdline_io(running_app: Sanic):
            """Small wrapper to shut down the server once cmd io is done."""
            await asyncio.sleep(1)  # allow server to start
            await console.record_messages(
                server_url=constants.DEFAULT_SERVER_FORMAT.format("http", port)
            )

            logger.info("Killing Sanic server now.")
            running_app.stop()  # kill the sanic serverx
```

该函数调用的是`console.record_messages`，如下：

```python
# rasa/core/channels/console.py
async def record_messages(
    server_url=DEFAULT_SERVER_URL,
    auth_token=None,
    sender_id=UserMessage.DEFAULT_SENDER_ID,
    max_message_limit=None,
    use_response_stream=True,
):
    """Read messages from the command line and print bot responses."""

    auth_token = auth_token if auth_token else ""

    exit_text = INTENT_MESSAGE_PREFIX + "stop"

    cli_utils.print_success(
        "Bot loaded. Type a message and press enter "
        "(use '{}' to exit): ".format(exit_text)
    )

    num_messages = 0
    button_question = None
    while not utils.is_limit_reached(num_messages, max_message_limit):
        # 获取用户输入文本
        text = get_user_input(button_question)

        if text == exit_text or text is None:
            break

        if use_response_stream: # True
            # 发送信息到服务器获取回复
            bot_responses = send_message_receive_stream(
                server_url, auth_token, sender_id, text
            )
            async for response in bot_responses:
                button_question = print_bot_output(response)
        else:
            bot_responses = await send_message_receive_block(
                server_url, auth_token, sender_id, text
            )
            for response in bot_responses:
                button_question = print_bot_output(response)

        num_messages += 1
    return num_messages
```

send_message_receive_stream的实现如下：

```python
async def send_message_receive_stream(server_url, auth_token, sender_id, message):
    payload = {"sender": sender_id, "message": message}

    url = "{}/webhooks/rest/webhook?stream=true&token={}".format(server_url, auth_token)

    # Define timeout to not keep reading in case the server crashed in between
    timeout = ClientTimeout(DEFAULT_STREAM_READING_TIMEOUT_IN_SECONDS)
    # TODO: check if this properly receives UTF-8 data
    # 通过调用webhooks http api接口，获取响应信息
    async with aiohttp.ClientSession(timeout=timeout) as session:
        async with session.post(url, json=payload, raise_for_status=True) as resp:

            async for line in resp.content:
                if line:
                    yield json.loads(line.decode(DEFAULT_ENCODING))
```

那么webhooks对应的路由是在什么地方添加的呢？在上文介绍的configure_app函数内部的`rasa.core.channels.channel.register`函数实现中，具体如下：

```python
# rasa/core/channels/channel.py
def register(
    input_channels: List["InputChannel"], app: Sanic, route: Optional[Text]
) -> None:
    async def handler(*args, **kwargs):
        # 用来处理消息
        await app.agent.handle_message(*args, **kwargs)

    for channel in input_channels:
        if route:
            # p： '/webhooks/rest'
            p = urljoin(route, channel.url_prefix())
        else:
            p = None
        # 添加到app的路由上
        app.blueprint(channel.blueprint(handler), url_prefix=p)

    app.input_channels = input_channels
```

那么消息处理的路由函数为CmdlineInput子类RestInput中，如下：

```python
class RestInput(InputChannel):
    """A custom http input channel.

    This implementation is the basis for a custom implementation of a chat
    frontend. You can customize this to send messages to Rasa Core and
    retrieve responses from the agent."""

    @classmethod
    def name(cls) -> Text:
        return "rest"

    @staticmethod
    async def on_message_wrapper(
        on_new_message: Callable[[UserMessage], Awaitable[Any]],
        text: Text,
        queue: Queue,
        sender_id: Text,
        input_channel: Text,
        metadata: Optional[Dict[Text, Any]],
    ) -> None:
        collector = QueueOutputChannel(queue)

        message = UserMessage(
            text, collector, sender_id, input_channel=input_channel, metadata=metadata
        )
        await on_new_message(message)

        await queue.put("DONE")  # pytype: disable=bad-return-type

    async def _extract_sender(self, req: Request) -> Optional[Text]:
        return req.json.get("sender", None)

    # noinspection PyMethodMayBeStatic
    def _extract_message(self, req: Request) -> Optional[Text]:
        return req.json.get("message", None)

    def _extract_input_channel(self, req: Request) -> Text:
        return req.json.get("input_channel") or self.name()

    def stream_response(
        self,
        on_new_message: Callable[[UserMessage], Awaitable[None]],
        text: Text,
        sender_id: Text,
        input_channel: Text,
        metadata: Optional[Dict[Text, Any]],
    ) -> Callable[[Any], Awaitable[None]]:
        async def stream(resp: Any) -> None:
            # 这里的Queue作用于生产者消费者模式
            q = Queue()
            task = asyncio.ensure_future(
                self.on_message_wrapper(
                    on_new_message, text, q, sender_id, input_channel, metadata
                )
            )
            result = None  # declare variable up front to avoid pytype error
            while True:
                result = await q.get()
                if result == "DONE":
                    break
                else:
                    await resp.write(json.dumps(result) + "\n")
            await task

        return stream  # pytype: disable=bad-return-type

    # 这里的on_new_message就是上一个片段的代码中出现的：
    # async def handler(*args, **kwargs):
    #     # 用来处理消息
    #     await app.agent.handle_message(*args, **kwargs)
    def blueprint(
        self, on_new_message: Callable[[UserMessage], Awaitable[None]]
    ) -> Blueprint:
        custom_webhook = Blueprint(
            "custom_webhook_{}".format(type(self).__name__),
            inspect.getmodule(self).__name__,
        )

        # noinspection PyUnusedLocal
        # 最后组合后的url为 ip:port/webhooks/rest/
        @custom_webhook.route("/", methods=["GET"])
        async def health(request: Request) -> HTTPResponse:
            return response.json({"status": "ok"})

        # 最后组合后的url为 ip:port/webhooks/rest/webhook
        @custom_webhook.route("/webhook", methods=["POST"])
        async def receive(request: Request) -> HTTPResponse:
            sender_id = await self._extract_sender(request)
            # 用户输入的消息
            text = self._extract_message(request)
            should_use_stream = rasa.utils.endpoints.bool_arg(
                request, "stream", default=False
            )
            input_channel = self._extract_input_channel(request)
            metadata = self.get_metadata(request)

            if should_use_stream:
                return response.stream(
                    self.stream_response(
                        on_new_message, text, sender_id, input_channel, metadata
                    ),
                    content_type="text/event-stream",
                )
            else:
                collector = CollectingOutputChannel()
                # noinspection PyBroadException
                try:
                    await on_new_message(
                        UserMessage(
                            text,
                            collector,
                            sender_id,
                            input_channel=input_channel,
                            metadata=metadata,
                        )
                    )
                except CancelledError:
                    logger.error(
                        "Message handling timed out for "
                        "user message '{}'.".format(text)
                    )
                except Exception:
                    logger.exception(
                        "An exception occured while handling "
                        "user message '{}'.".format(text)
                    )
                return response.json(collector.messages)

        return custom_webhook
```

最后关于消息的处理会进入到agent中（在之前准备好了的agent在这里有起到作用了）。关于agent的介绍可以参见：https://zhuanlan.zhihu.com/p/88194193。agent的handle_message函数如下：

```python
    async def handle_message(
        self,
        message: UserMessage,
        message_preprocessor: Optional[Callable[[Text], Text]] = None,
        **kwargs,
    ) -> Optional[List[Dict[Text, Any]]]:
        """Handle a single message."""

        if not isinstance(message, UserMessage):
            logger.warning(
                "Passing a text to `agent.handle_message(...)` is "
                "deprecated. Rather use `agent.handle_text(...)`."
            )
            # noinspection PyTypeChecker
            return await self.handle_text(
                message, message_preprocessor=message_preprocessor, **kwargs
            )

        def noop(_):
            logger.info("Ignoring message as there is no agent to handle it.")
            return None

        if not self.is_ready():
            return noop(message)
	    # 创建MessageProcessor
        processor = self.create_processor(message_preprocessor)

        async with self.lock_store.lock(message.sender_id):
            # 利用MessageProcessor处理消息
            return await processor.handle_message(message)
```

MessageProcessor类为处理消息的集大成者，该类中handle_message实现如下：

```python
# rasa/core/processor.py
    async def handle_message(
        self, message: UserMessage
    ) -> Optional[List[Dict[Text, Any]]]:
        """Handle a single message with this processor."""

        # preprocess message if necessary
        # 将消息记录到tracker中
        tracker = await self.log_message(message, should_save_tracker=False)
        if not tracker:
            return None

        if not self.policy_ensemble or not self.domain:
            # save tracker state to continue conversation from this state
            self._save_tracker(tracker)
            logger.warning(
                "No policy ensemble or domain set. Skipping action prediction "
                "and execution."
            )
            return None

        # 对下一个action进行预测以及执行
        await self._predict_and_execute_next_action(message, tracker)
        # save tracker state to continue conversation from this state
        self._save_tracker(tracker)

        # 如果有返回消息，则返回
        if isinstance(message.output_channel, CollectingOutputChannel):
            return message.output_channel.messages
        else:
            return None

```

## 小结

关键的类为Agent，和MessageProcessor。其他为与webapi实现相关的逻辑。

## 补充知识点

- [Sanic Blueprint](https://www.osgeo.cn/sanic/sanic/blueprints.html)
- [Sanic 中文文档](https://www.osgeo.cn/sanic/)