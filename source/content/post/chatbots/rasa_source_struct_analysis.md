+++
date="2019-12-21"
title="rasa source - 项目结构分析"
categories=["chatbot","rasa"]
tags=["rasa-source"]
+++

# rasa source - 项目结构分析

## rasa依赖项分析

相关依赖项定义在`setup.py`文件中，关于`setup.py`的介绍可以参见：https://www.cnblogs.com/maociping/p/6633948.html。

相关的依赖主要有如下三个变量定义，下面的注释中给出了对应依赖的简单说明，以及学习的相关链接：

```python
tests_requires = [
    # pytest： test framework
    # http://pytest.org/en/latest/contents.html
    "pytest~=4.5", 
    # pytest-cov: 测试覆盖率检测工具
    # https://pytest-cov.readthedocs.io/en/latest/
    "pytest-cov~=2.7",
    # pytest-localserver: 用于本地测试服务连接
    # https://pypi.org/project/pytest-localserver/
    "pytest-localserver~=0.5.0",
    # pytest-sanic: 针对sanic的pytest插件
    # https://github.com/yunstanford/pytest-sanic
    "pytest-sanic~=1.0.0",
    # responses： 用来模拟requests库
    # https://pypi.org/project/responses/
    "responses~=0.9.0",
    # freezegun: 通过伪造日期模块来生成不同的时间
    # https://github.com/spulec/freezegun
    "freezegun~=0.3.0",
    # nbsphinx: sphinx的扩展，用于*.ipynb文件的解析
    # https://pypi.org/project/nbsphinx/
    "nbsphinx>=0.3",
    # aioresponses：用来模拟python aiohttp包的页面请求
    # https://github.com/pnuckowski/aioresponses
    "aioresponses~=0.6.0",
    # moto：模拟基于AWS架构的测试
    # https://github.com/spulec/moto
    "moto~=1.3.8",
    # fakeredis: 这是针对没有redis或测试环境的机器的redis rb的伪实现
    # https://github.com/guilleiguaran/fakeredis
    "fakeredis~=1.0",
]

install_requires = [
    "requests>=2.20",       # http 请求库
    "boto3~=1.9",           # 适用于 Python 的 AWS 开发工具包
    "matplotlib~=3.0",      # 绘图
    "attrs>=18",            # 详细介绍见：https://www.jb51.net/article/162909.htm
    "jsonpickle~=1.1",      # 用来序列化复杂的 Python 对象到 JSON 文档，https://jsonpickle.github.io/
    "redis~=3.3.5",         # 与redis交互
    "pymongo[tls,srv]~=3.8",# mongod交互
    "numpy~=1.16",          # numpy
    "scipy~=1.2",           # scipy，科学计算库
    "tensorflow~=1.14.0",   # tensorlfow，深度学习框架
    # absl is a tensorflow dependency, but produces double logging before 0.8
    # should be removed once tensorflow requires absl > 0.8 on its own
    "absl-py>=0.8.0",
    # setuptools comes from tensorboard requirement:
    # https://github.com/tensorflow/tensorboard/blob/1.14/tensorboard/pip_package/setup.py#L33
    "setuptools >= 41.0.0",
    "tensorflow-probability~=0.7.0", # 概率编程，参见：https://blog.csdn.net/yH0VLDe8VG8ep9VGe/article/details/84985912
    "tensor2tensor~=1.14.0",   # https://github.com/tensorflow/tensor2tensor
    "apscheduler~=3.0",        # 安排定期执行的任务，https://www.cnblogs.com/zhaoyingjie/p/9664081.html
    "tqdm~=4.0",               # 一个快速，可扩展的Python进度条，https://blog.csdn.net/langb2014/article/details/54798823
    "networkx~=2.3",           # 网络分析库，https://www.cnblogs.com/kaituorensheng/p/5423131.html
    "fbmessenger~=6.0",        # 与facebook messenger api通信
    "pykwalify~=1.7.0",        # YAML JSON校验库
    "coloredlogs~=10.0",       # 用于带颜色的终端输出，https://coloredlogs.readthedocs.io/en/latest/
    "scikit-learn~=0.20.2",    # 机器学习库
    "ruamel.yaml~=0.15.0",     # YAML parser/emitter
    "scikit-learn~=0.20.0",    # duplicate
    "slackclient~=1.3",        # 用于和slack web api交互，https://pypi.org/project/slackclient/
    "python-telegram-bot~=11.0",  #  Telegram Bot API.的python接口，https://github.com/python-telegram-bot/python-telegram-bot
    "twilio~=6.0",                # Twilio API Python实现
    "webexteamssdk~=1.1",         # Webex Teams APIs
    "mattermostwrapper~=2.0",     # Mattermost 是一个 Slack 的开源替代品
    "rocketchat_API~=0.6.0",      # rocketchat_API
    "colorhash~=1.0",             # 针对对象的hash值生成颜色，https://pypi.org/project/colorhash/
    "pika~=1.0.0",                # RabbitMQ 客户端库
    "jsonschema~=2.6",            # 对json数据进行校验
    "packaging~=19.0",            # 打包
    "gevent~=1.4",                # https://www.liaoxuefeng.com/wiki/897692888725344/966405998508320，https://blog.csdn.net/freeking101/article/details/53097420
    "pytz~=2019.1",               # python时区设置
    "python-dateutil~=2.8",       # https://blog.csdn.net/handsomekang/article/details/10128671
    "rasa-sdk~=1.3.0",
    "colorclass~=2.2",            # 为控制台添加颜色
    "terminaltables~=3.1",        # 控制台输出表格
    "sanic~=19.6",                # sanic web框架
    "sanic-cors==0.9.9.post1",    # 实现跨域访问很方便的扩展
    "sanic-jwt~=1.3",             # 认证插件
    "aiohttp~=3.5",               # https://www.liaoxuefeng.com/wiki/1016959663602400/1017985577429536
    "questionary>=1.1.0",         # 用于构建漂亮的命令行输入提示
    "python-socketio>=4.3.1",     # python socket.io
    # the below can be unpinned when python-socketio pins >=3.9.3
    "python-engineio>=3.9.3",     # Engine.IO server and client
    "pydot~=1.4",                 # 数据可视化
    "async_generator~=1.10",      # 
    "SQLAlchemy~=1.3.0",          # 提供了SQL工具包及对象关系映射工具
    "kafka-python~=1.4",          # kafka python客户端
    "sklearn-crfsuite~=0.3.6",    # CRF suit
    "PyJWT~=1.7",                 # json网络令牌
    # remove when tensorflow@1.15.x or a pre-release patch is released
    # https://github.com/tensorflow/tensorflow/issues/32319
    "gast==0.2.2",                # 通用的抽象语法树
]

extras_requires = {
    "test": tests_requires,
    "spacy": ["spacy>=2.1,<2.2"],
    "mitie": ["mitie"],
    "sql": ["psycopg2~=2.8.2", "SQLAlchemy~=1.3"],
}

```

## 各目录功能介绍

```
- alt_requirements: 相关依赖
- binder: 额外的依赖
- data: 存放不同配置文件，不同训练数据格式的示例
- docker：docker相关
- doc：文档相关
- examples：示例
- rasa：代码路径
	- cli：command line interface相关
	- core：对话状态管理以及用户行为选择
	- importers：数据导入等相关
	- nlu：自然语言理解
	- utils：工具
	- __init__.py：用于将文件夹变为一个python模块，模块导入的时候会被调用，可以参见：https://www.cnblogs.com/Lands-ljk/p/5880483.html
	- __main__.py：setup中有如下定义，entry_points={"console_scripts": ["rasa=rasa.__main__:main"]},也就是rasa当做命令执行的时候，会执行rasa.__main__:main，即主程序入口
	- constants.py：用来定义常量
	- data.py：获取nlu，stories文件，提供文件类型校验
	- exceptions.py：自定义excepjtions
	- jupyter.py：用来支持jupyter中进行对话
	- model.py：模型相关的操作
	- run.py：运行rasa模型
	- server.py：web 服务相关
	- test.py：测试相关
	- train.py：用来训练
	- version.py：定义版本信息
- rasa_core：之前core的路径
- rasa_nlu：之前nlu的路径
- sample_configs：一些示例的配置
- scripts：ping slack的脚本
- tests：测试代码
```

