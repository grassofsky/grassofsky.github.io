+++
date="2019-12-21"
title="rasa对话系统如何实现联想输入"
categories=["chatbot","rasa"]
tags=["rasa-extension"]
+++

# rasa对话系统如何实现联想输入

## rasa 文集

[rasa文章导引（用于收藏）](https://zhuanlan.zhihu.com/p/88112269)

## 联想输入

**问题描述**：输入关键词，从候选的问题中选择出相似度最大的前n个词。

**具体示例**：

候选问题如下：

- 笔记本死机了怎么办？
- 计算机死机了怎么办？
- 电脑卡死了？
- 电脑用着突然卡死了
- 手机死机了
- 电脑不能上网了
- 电脑死机了

提的问题如下：我的电脑死机了怎么办？

联想词搜索原理：https://blog.csdn.net/DusonBlog/article/details/52661237。

下面Demo使用倒排索引进行实现，参见：https://blog.csdn.net/u011239443/article/details/60604017

```python
import jieba # 使用结巴分词器
import numpy

corpus = [
    "笔记本死机了怎么办",
    "计算机死机了怎么办",
    "电脑卡死了",
    "电脑用着突然卡死了",
    "手机死机了",
    "电脑不能上网了",
    "电脑死机了"
]

def search_related_questions(question):
    # 网上很容易就可以搜索到停用词表
    stopwords = ['我', '了', '着', '的', '怎么办']
    vocabulary = set()

    corpus_tokens = []
    for sentence in corpus:
        # 分词
        tokens = jieba.cut(sentence)
        # 去除停用词
        tokens_without_stops = [token for token in tokens if token not in stopwo                                  rds]
        corpus_tokens.append(tokens_without_stops)
        for word in tokens_without_stops:
            vocabulary.add(word)

    vocabulary = sorted(list(vocabulary))

    search_dict = {}
    for word in vocabulary:
        search_dict[word] = []
        for i, tokens in enumerate(corpus_tokens):
            count = tokens.count(word)
            if count != 0:
                search_dict[word].append([i, count])

    # 进行检索
    question_tokens = jieba.cut(question)
    question_tokens_without_stops = [token for token in question_tokens if token                                   not in stopwords]

    # 相关的问题有
    related_question = []
    for token in question_tokens_without_stops:
        if token in search_dict.keys():
            related_question += search_dict[token]

    # 相关问题匹配到的次数
    related_question_dict = {}
    for id_count_pair in related_question:
        if id_count_pair[0] in related_question_dict:
            related_question_dict[id_count_pair[0]] += id_count_pair[1]
        else:
            related_question_dict[id_count_pair[0]] = id_count_pair[1]

    # 进行排序
    sorted_question = sorted(related_question_dict.items(), key=lambda item:item                                  [1], reverse=True)

    # 输出相似问题排名
    related_question_str = []
    for question in sorted_question:
        related_question_str.append(corpus[question[0]])
    return related_question

if __name__ == '__main__':
    question = "我的电脑死机了怎么办"
    search_related_questions(question)
```

## 将rasa和联想输入结合

联想输出，通常情况下是针对question-answer系统。下面针对如何将联想输入结合到rasa 给出一定思路。可以在`RestInput`添加相关实现，类似如下：

```python
# file: rasa/core/channels/channel.py
class RestInput(InputChannel):
    def blueprint(
        self, on_new_message: Callable[[UserMessage], Awaitable[None]]
    ) -> Blueprint:
        ...
        @custom_webhook.route("/suggest", methods=["POST"])
        async def suggest(request: Requkest) -> HTTPResponse:
            sender_id = await self._extract_sender(request)
            # 用户输入的消息
            text = self._extract_message(request)
            related_questions = search_related_questions(text)
                return response.json({"suggest": related_questions})
        ....
```









