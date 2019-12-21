+++
date="2019-12-21"
title="rasa source - 命令行接口实现源码分析"
categories=["chatbot","rasa"]
tags=["rasa-source"]
+++

# rasa source - 命令行接口实现源码分析

关于命令行介绍可以参见：https://rasa.com/docs/rasa/user-guide/command-line-interface/

源码地址位于：https://github.com/RasaHQ/rasa

argparse使用参见：https://pymotw.com/3/argparse/index.html，https://docs.python.org/zh-cn/3/library/argparse.html#module-argparse

关于直接在浏览器上进行代码浏览，可以安装**sourcegraph**插件，在这之前可以先安装**谷歌访问助手**。

## \__main__.py分析

`__main__.py`为程序执行的入口文件，从该文件开始分析，该文件的main函数如下：

```python
def main() -> None:
    # Running as standalone python application
    import os
    import sys

    parse_last_positional_argument_as_model_path()
    arg_parser = create_argument_parser()
    cmdline_arguments = arg_parser.parse_args()

    # 下面的代码为其他逻辑省略
    # ...
```

主要的实现见`create_argument_parse()`，代码如下：

```python
def create_argument_parser() -> argparse.ArgumentParser:
    """Parse all the command line arguments for the training script."""

    parser = argparse.ArgumentParser(
        prog="rasa",
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
        description="Rasa command line interface. Rasa allows you to build "
        "your own conversational assistants 🤖. The 'rasa' command "
        "allows you to easily run most common commands like "
        "creating a new bot, training or evaluating models.",
    )

    parser.add_argument(
        "--version",
        action="store_true",
        default=argparse.SUPPRESS,
        help="Print installed Rasa version",
    )

    parent_parser = argparse.ArgumentParser(add_help=False)
    add_logging_options(parent_parser)   # 添加日志记录等级
    parent_parsers = [parent_parser]

    subparsers = parser.add_subparsers(help="Rasa commands")

    # 下面的具体实现都位于cli目录下面
    scaffold.add_subparser(subparsers, parents=parent_parsers)
    run.add_subparser(subparsers, parents=parent_parsers)
    shell.add_subparser(subparsers, parents=parent_parsers)
    train.add_subparser(subparsers, parents=parent_parsers)
    interactive.add_subparser(subparsers, parents=parent_parsers)
    test.add_subparser(subparsers, parents=parent_parsers)
    visualize.add_subparser(subparsers, parents=parent_parsers)
    data.add_subparser(subparsers, parents=parent_parsers)
    x.add_subparser(subparsers, parents=parent_parsers)

    return parser
```

## cli分析

cli完成的主要功能包括两点：

1. 从命令行中获取运行需要的参数
2. 执行对应的函数

关于第1点，所有对应的参数介绍可以参见：https://rasa.com/docs/rasa/user-guide/command-line-interface/。

关于第二点，下面将列出可能被执行的函数，在项目中全局搜索`set_defaults`：

- `rasa/cli/data.py`
	- `data_parser.set_defaults(func=lambda _: data_parser.print_help(None))`
	- `convert_parser.set_defaults(func=lambda _: data_parser.print_help(None))`
	- `convert_nlu_parser.set_defaults(func=convert.main)`： 与命令 `rasa data convert nlu` 对应
	- `split_parser.set_defaults(func=lambda _: split_parser.print_help(None))`
	- `nlu_split_parser.set_defaults(func=split_nlu_data)` ：与命令 `rasa data split nlu` 对应
	- `validate_parser.set_defaults(func=validate_files) `：与命令 `rasa data validate` 对应
- `rasa/cli/interactive.py`
  - `interactive_parser.set_defaults(func=interactive)`：与命令`rasa interactive`对应
  - `interactive_core_parser.set_defaults(func=interactive_core)`：与命令`rasa interactive core`对应
- `rasa/cli/run.py`
  - `sdk_subparser.set_defaults(func=run_actions)`：与命令`rasa run`对应
- `rasa/cli/scaffold.py`
  - `scaffold_parser.set_defaults(func=run)`：与命令`rasa init`对应
- `rasa/cli/shell.py`
  - `shell_parser.set_defaults(func=shell)`：与命令`rasa shell`对应
  - `shell_nlu_subparser.set_defaults(func=shell_nlu)`：与命令`rasa shell nlu`对应
- `rasa/cli/test.py`
  - `test_core_parser.set_defaults(func=test_core)`：与命令`rasa test core`对应
  - `test_nlu_parser.set_defaults(func=test_nlu)`：与命令`rasa test nlu`对应
  - `test_parser.set_defaults(func=test)`：与命令`rasa test`对应
- `rasa/cli/train.py`
  - `train_core_parser.set_defaults(func=train_core)`：与命令`rasa train core`对应
  - `train_nlu_parser.set_defaults(func=train_nlu)`：与命令`rasa train nlu`对应
  - `train_parser.set_defaults(func=train)`：与命令`rasa train`对应
- `rasa/cli/visualize.py`
  - `visualize_parser.set_defaults(func=visualize_stories)`：与命令`rasa visualize`对应
- `rasa/cli/x.py`
  - `shell_parser.set_defaults(func=rasa_x)`：与命令`rasa x`对应

## 后续

后续的文章，将分别从这些命令入手，对源码进行解析。

