+++
date="2020-01-16"
title=" rasa problem - 直接使用agent进行会话"
categories=["chatbot","rasa"]
tags=["rasa-problem"]

+++

# rasa problem - 直接使用agent进行会话


对于agent的使用，agent介绍的一节中，有示例如下：

```python
>>> from rasa.core.agent import Agent
>>> from rasa.core.interpreter import RasaNLUInterpreter
>>> agent = Agent.load("examples/restaurantbot/models/current")
>>> await agent.handle_text("hello")
[u'how can I help you?']
```

直接这样操作为有如下错误，

1. 格式错误，await不能直接在最外层调用。
2. 没法与custom action进行关联。

要实现上面两种要求需要进行如下操作：

```python
import asyncio
from rasa.core.agent import Agent
from rasa.utils.endpoints import EndpointConfig


async def test_agent():
    endpoint = EndpointConfig(url="http://localhost:5055/webhook")
    agent = Agent.load(
        "./models/20200113-230637.tar.gz",
        action_endpoint=endpoint
    )
    result = await agent.handle_text("hello")
    print(result)

loop = asyncio.get_event_loop()
loop.run_until_complete(test_agent())
```

