+++
date="2019-12-21"
title="对话系统rasa - Lock Stores （翻译）"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统rasa - Lock Stores （翻译）

rasa使用票证锁定机制来确保按正确的顺序处理给定会话id的传入消息，并在消息处于活动处理状态时锁定会话。这意味着多个rasa服务器可以作为复制服务并行运行，并且客户端在发送给定会话id的消息时不一定需要地址相同的节点。

## 目录

- Lock Stores
  - InMemoryLockStore(default)
  - RedisLockStore

## InMemoryLockStore(default)

**描述**：`InMemoryLockStore` 是默认的lock store。它在单个进程中维护会话锁。

**注意**：当多个RASA服务器并行运行时，不应使用此锁存储。

**配置**：使用`InMemoryLockStore` 不需要额外的配置。

## RedisLockStore

**描述**：RedisLockStore使用redis作为持久层来维护会话锁。这是运行RASA服务器集的推荐lock store。

**配置**：

1 启动rasa实例

2 在endpoints.yml中添加配置，如下：

```
lock_store:
    type: "redis"
    url: <url of the redis instance, e.g. localhost>
    port: <port of your redis instance, usually 6379>
    password: <password used for authentication>
    db: <number of your database within redis, e.g. 0>
```

3  利用该配置启动rasa core server，如：`rasa run -m models --endpoints endpoints.yml`

**参数**：

- url（默认：localhost）：redis实例的url
- port（默认：6379）：redis运行的端口
- db（默认：0）：redis数据库的数量
- password（默认：None）：用于认证的密码（None就是不需要认证）
- use_ssl（默认：False）：是否需要对通信加密

## 原文链接

https://rasa.com/docs/rasa/api/lock-stores/