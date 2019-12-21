+++
date="2019-12-21"
title="对话系统rasa - Messaging and Voice Channels （翻译）"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统rasa - Messaging and Voice Channels （翻译）

如果你在本机进行测试（而不是服务器上），你回需要使用[ngrok](https://rasa.com/docs/rasa-x/get-feedback-from-test-users/#use-ngrok-for-local-testing)。这个可以帮助Facebook，Slack等知道将消息发送到你的本地机器。

为了使得你的助手能够适用于消息平台，你需要提供`credentials.yml`配置文件。当你运行`rasa init`的时候会生成对应的示例文件，因此，最简单的方法是直接对其进行编辑。下面是facebook的示例：

```
facebook:
  verify: "rasa-bot"
  secret: "3e34709d01ea89032asdebfe5a74518"
  page-access-token: "EAAbHPa7H9rEBAAuFk4Q3gPKbDedQnx4djJJ1JmQ7CAqO4iJKrQcNT0wtD"
```

学习如何将你的助手适用到一下平台：

- 你的个人网站
- [Facebook Messenger](https://rasa.com/docs/rasa/user-guide/connectors/facebook-messenger/)
- [Slack](https://rasa.com/docs/rasa/user-guide/connectors/slack/)
- [Telegram](https://rasa.com/docs/rasa/user-guide/connectors/telegram/)
- [Twilio](https://rasa.com/docs/rasa/user-guide/connectors/twilio/)
- [Microsoft Bot Framework](https://rasa.com/docs/rasa/user-guide/connectors/microsoft-bot-framework/)
- [Cisco Webex Teams](https://rasa.com/docs/rasa/user-guide/connectors/cisco-webex-teams/)
- [RocketChat](https://rasa.com/docs/rasa/user-guide/connectors/rocketchat/)
- [Mattermost](https://rasa.com/docs/rasa/user-guide/connectors/mattermost/)
- Custom Connectors

下面仅对“你的个人网站”和“Custom Connectors”进行介绍。

## 你的个人网站

你需要一个简单的方式来测试你的bot，最好的方式使用Rasa X，通过Rasa X，你可以邀请用户对你的bot进行测试。

如果你已经有了一个网站，想要将Rasa助手添加到上面，你有下面几个备选项：

- [Rasa Webchat](https://github.com/mrbot-ai/rasa-webchat),使用websockets
- [Chatroom](https://github.com/scalableminds/chatroom), 使用常规的http请求
- [rasa-bot](https://github.com/assister-ai/assister/tree/master/packages/rasa), 使用常规http请求的web组件

**注意**：这些项目都是外部的开发者开发，有任何问题请联系对应的开发者。

### Websocket Channel

SocketIO频道使用了websockets，是实时的。你需要包含下面内容的`credentials.yml`文件：

```
socketio:
  user_message_evt: user_uttered
  bot_message_evt: bot_uttered
  session_persistence: true/false
```

配置中的前两项定义了Rasa Core通过socket.io接收和发送消息是对应的事件名称。

默认情况下，socketio通道使用socket id作为`sender_id`，这会使得重新加载每个页面的时候重启session。可以将`session_persistence`设置为`true`，来笔名重启。这种情况，前端有必要产生一个session id，在`connect`事件之后，通过触发事件`session_request`并将`{session_id: [session_id]}`发送给Rasa Core server。

[Webchat](https://github.com/mrbot-ai/rasa-webchat)实现了session创建机制。

### REST Channels

`RestInput`和`CallbackInput`频道可以用于自定义集成。他们提供了URL，你可以通过post消息并且直接接收回复，或者通过webhook实现异步。

#### RestInput

`rest`频道将提供你REST终端用来post消息机器，或接收来自机器的响应。下面是一个关于如何连接到`rest`输入channel的例子：

```
rasa run
```

你需要确保你的credentials.yml文件中有以下内容：

```
rest:
  # you don't need to provide anything here - this channel doesn't
  # require any credentials
```

在连接到`rest`输入channel后，你可以以下面的格式post消息到`POST /webhooks/rest/webhook`：

```json
{
    "sender": "Rasa",
    "message": "Hi there"
}
```

回复这个请求的将包括如下内容，如：

```json
[
  {"text": "Hey Rasa!"}, {"image": "http://example.com/image.jpg"}
]
```

#### CallbackInput

`callback`频道和`rest`输入非常类似，但是不会直接通过发送消息的http请求返回机器消息，它将调用你可以设置的URL来发送消息。

下面是一个关于如何连接到`callback`输入channel的例子，使用run脚本：

```
rasa run
```

你需要确保你的credentials.yml文件中有以下内容:

```
callback:
  # URL to which Core will send the bot responses
  url: "http://localhost:5034/bot"
```

在连接到callback输入频道之后，你能够将消息以下面的格式`POST /webhooks/callback/webhook`：

```json
{
  "sender": "Rasa",
  "message": "Hi there!"
}
```

响应是简单的success。一旦Core想要发送消息给用户的时候，它会调用你设定的URL，将JSON结果以POST的形式发送过来。

```json
[
  {"text": "Hey Rasa!"}, {"image": "http://example.com/image.jpg"}
]
```

## Custom Connectors

你可以实现你自己的channel。你可以使用`rasa.core.channels.channel.RestInput`类作为模板。你需要实现的函数是`blueprint`和`name`。这个函数需要创建sanic blueprint，用来挂载到sanic服务。

这允许你对服务器添加REST终端，外部的消息服务能够调用它进行消息传递。

你的blueprint应该至少要有两个routes：针对`/`提供health，`/webhook`用来http接收。

name函数定义了URL中的前缀，如，你的组件的名字是myio，webhook提供的外部的服务是：`http://localhost:5005/webhooks/myio/webhook`。（将hostname和端口替换成你自己的）。

为了发送消息，你需要运行类似如下的命令：

```bash
curl -XPOST http://localhost:5000/webhooks/myio/webhook \
  -d '{"sender": "user1", "message": "hello"}' \
  -H "Content-type: application/json"
```

这里myio就是你的组件的名字。

如果你想要在你的自定义action中使用前端提供的额外的信息，你可以将这些信息添加到metadata字典中。这些信息会在rasa server组成user message，然后传递给action server，接着你可以在tracker中找到它。消息的元数组并不会直接影响到NLU分类或者action预测。如果你想要改变该频道的metadata的提取方式，你可以重写get_metadata函数。该函数的返回值会传给UserMessage。

这里是UserMessage的所有属性：

```python
class rasa.core.channels.UserMessage(text=None, output_channel=None, sender_id=None, parse_data=None, input_channel=None, message_id=None, metadata=None)
```

用来表述输入的消息。

包括频道和响应需要被设置。

```python
__init__(text=None, output_channel=None, sender_id=None, parse_data=None, input_channel=None, message_id=None, metadata=None)
```

用来创建一个UserMessage对象。

参数：

- text： 消息文本内容
- output_channel：回复消息发送给用户的通道
- sender_id：消息所有者的id
- parse_data：关于消息的rasa data
- input_channel：用来接收消息的channel
- message_id：消息id
- metadata：额外的消息元数据

在你实现`receive`终端的时候，你需要确保调用了`on_new_message(UserMessage(text, output, sender_id))`。这将会告诉Rasa Core来处理用户消息。`output`是output channel，基于OutputChannel实现。你可以针对你的特殊chat channel实现这些方法（如，方法用来发送文本和图像）或者使用CollectingOutputChannel，来收集Core在处理消息的时创建的回复，并将其作为终端回复的一部分返回。RestInput频道就是这么实现的。举例，如何创建和使用你自己的output channel，你可以参见其他output channel的实现，如`rasa.core.channels.slack`中的`SlakBot`实现。

为了使用自定义的channel，你需要在命令行`--credentials`上提供`credentials.yml` 。这个文件必须包含你的自定义channel的模块路径，和任何需要的配置参数。如：

```
mypackage.MyIO:
  username: "user_name"
  another_parameter: "some value"
```

下面是一个例子，给出了接收消息，通过RasaCore处理消息，收集机器语句，将机器语句以json格式作为webhook调用的响应，的输入频道的实现：

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

    def blueprint(
        self, on_new_message: Callable[[UserMessage], Awaitable[None]]
    ) -> Blueprint:
        custom_webhook = Blueprint(
            "custom_webhook_{}".format(type(self).__name__),
            inspect.getmodule(self).__name__,
        )

        # noinspection PyUnusedLocal
        @custom_webhook.route("/", methods=["GET"])
        async def health(request: Request) -> HTTPResponse:
            return response.json({"status": "ok"})

        @custom_webhook.route("/webhook", methods=["POST"])
        async def receive(request: Request) -> HTTPResponse:
            sender_id = await self._extract_sender(request)
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

