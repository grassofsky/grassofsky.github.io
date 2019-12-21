+++
date="2019-12-21"
title="rasa source - 初始化源码分析"
categories=["chatbot","rasa"]
tags=["rasa-source"]
+++

# rasa source - 初始化源码分析

执行命令为：`rasa init`。

对应的入口函数为：`rasa/cli/scaffold.py: run`。函数代码如下：

```python
def run(args: argparse.Namespace) -> None:
    import questionary

    print_success("Welcome to Rasa! 🤖\n")
    if args.no_prompt:
        print (
            "To get started quickly, an "
            "initial project will be created.\n"
            "If you need some help, check out "
            "the documentation at {}.\n".format(DOCS_BASE_URL)
        )
    else:
        print (
            "To get started quickly, an "
            "initial project will be created.\n"
            "If you need some help, check out "
            "the documentation at {}.\n"
            "Now let's start! 👇🏽\n".format(DOCS_BASE_URL)
        )

    path = (
        questionary.text(
            "Please enter a path where the project will be "
            "created [default: current directory]",
            default=".",
        )
        .skip_if(args.no_prompt, default=".")   # 参数传递了no_prompt，直接使用默认值，不进行提示
        .ask()
    )

    # 如果path不存在，询问并进行创建
    if not os.path.isdir(path):
        _ask_create_path(path)

    # 如果path为空，path不是目录，退出
    if path is None or not os.path.isdir(path):
        print_cancel()

    # 如果没有传递no_prompt，并且目录下还有东西，则询问进行重写
    if not args.no_prompt and len(os.listdir(path)) > 0:
        _ask_overwrite(path)
	
    # 重点函数实现：初始化项目
    init_project(args, path)
```

`init_project`实现如下：

```python
def print_train_or_instructions(args: argparse.Namespace, path: Text) -> None:
    import questionary

    print_success("Finished creating project structure.")

    should_train = questionary.confirm(
        "Do you want to train an initial model? 💪🏽"
    ).skip_if(args.no_prompt, default=True)
	
    # 进行询问确定是否需要训练初始模型
    if should_train:
        print_success("Training an initial model...")
        config = os.path.join(path, DEFAULT_CONFIG_PATH)
        training_files = os.path.join(path, DEFAULT_DATA_PATH)
        domain = os.path.join(path, DEFAULT_DOMAIN_PATH)
        output = os.path.join(path, create_output_path())

        args.model = rasa.train(domain, config, training_files, output)

        print_run_or_instructions(args, path)

    else:
        print_success(
            "No problem 👍🏼. You can also train a model later by going "
            "to the project directory and running 'rasa train'."
            "".format(path)
        )


def print_run_or_instructions(args: argparse.Namespace, path: Text) -> None:
    from rasa.core import constants
    import questionary

    should_run = (
        questionary.confirm(
            "Do you want to speak to the trained assistant on the command line? 🤖"
        )
        .skip_if(args.no_prompt, default=False)
        .ask()
    )

    if should_run:
        # provide defaults for command line arguments
        attributes = [
            "endpoints",
            "credentials",
            "cors",
            "auth_token",
            "jwt_secret",
            "jwt_method",
            "enable_api",
            "remote_storage",
        ]
        # 设置训练的默认参数
        for a in attributes:
            setattr(args, a, None)

        args.port = constants.DEFAULT_SERVER_PORT

        shell(args)
    else:
        if args.no_prompt:
            print (
                "If you want to speak to the assistant, "
                "run 'rasa shell' at any time inside "
                "the project directory."
                "".format(path)
            )
        else:
            print_success(
                "Ok 👍🏼. "
                "If you want to speak to the assistant, "
                "run 'rasa shell' at any time inside "
                "the project directory."
                "".format(path)
            )


def init_project(args: argparse.Namespace, path: Text) -> None:
    create_initial_project(path)
    print ("Created project directory at '{}'.".format(os.path.abspath(path)))
    print_train_or_instructions(args, path)


def create_initial_project(path: Text) -> None:
    from distutils.dir_util import copy_tree
	# 将提前准备好的资源文件，拷贝到对应的目录下
    copy_tree(scaffold_path(), path)


def scaffold_path() -> Text:
    import pkg_resources
	# 将模块的位置和initial_project目录名进行组合，形成资源目录名
    return pkg_resources.resource_filename(__name__, "initial_project")

```

## 补充知识点

- questionary使用说明：https://pypi.org/project/questionary/

## 后续

下一步将介绍如何对初始化示例进行训练。

