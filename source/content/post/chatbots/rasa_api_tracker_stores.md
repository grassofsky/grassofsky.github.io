+++
date="2019-12-21"
title="对话系统rasa - tracker Stores （翻译）"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统rasa - tracker Stores （翻译）

所有的对话都存储在tracker store中。rasa core提供了不同存储类型的开箱即用实现。如果要使用其他存储，还可以通过扩展TrackerStore类来构建自定义跟踪器存储。

## 目录

- Tracker Stores
  - InMemoryTrackerStore(default)
  - SQLTrackerStore
  - RedisTrackerStore
  - MongoTrackerStore
  - Custom Tracker Store

## InMemoryTrackerStore(default)

**描述**：InMemoryTrackerStore是默认的tracker store。如果没有配置其他的tracker store就会使用该store。它将对话历史保存在内存中。**注意**：当重启rasa core的时候，所有的对话记录都会丢失。

**配置**：要想使用InMemoryTrackerStore并不需要额外的配置。

## SQLTrackerStore

**描述**：`SQLTrackerStore`用来将对话历史存储到SQL数据库中。这种方式存储的trackers，可以通过sender_id，timestamp，action name， intent name和typename进行查找。 

**配置**：

为了设置Rasa Core使用SQL，需要进行下面的配置：

1. 在endpolints.yml中添加配置：

   ```
   tracker_store:
       type: SQL
       dialect: "sqlite"  # the dialect used to interact with the db
       url: ""  # (optional) host of the sql db, e.g. "localhost"
       db: "rasa.db"  # path to your db
       username:  # username used for authentication
       password:  # password used for authentication
       query: # optional dictionary to be added as a query string to the connection URL
         driver: my-driver
   ```

2. 利用该配置启动rasa core server，如：`rasa run -m models --endpoints endpoints.yml`

**参数**：

- domain(默认：None)，与该tracker store关联的domain对象
- dialect（默认：sqlite）：用于与SQL后端通信的方言。具体查看：[SQLAlchemy docs](https://docs.sqlalchemy.org/en/latest/core/engines.html#database-urls) 
- host（默认：None）：SQL服务器的URL
- port（默认：None）：SQL服务器的端口
- db（默认：rasa.db）：被使用的database的路径
- username（默认：None）：用于认证的用户名
- password（默认：None）：用于认证的密码
- event_broker（默认：None）：将事件发布到对应的代理
- login_db（默认：None）：初始连接到的备用数据库名称，并创建由db指定的数据库（仅限postgresql）
- query（默认：None）：连接时传递给方言和/或dbapi的选项字典

## RedisTrackerStore

**描述**：RedisTrackerStore用来将对话历史存储到[Redis](https://redis.io/)。Redis是一个快速的内存key-value存储，同时也支持数据持久化。

**配置**：

1. 启动redis实例

2. 在endpoints.yml文件中添加配置，如下：

   ```
   tracker_store:
       type: redis
       url: <url of the redis instance, e.g. localhost>
       port: <port of your redis instance, usually 6379>
       db: <number of your database within redis, e.g. 0>
       password: <password used for authentication>
       use_ssl: <whether or not the communication is encrypted, default `false`>
   ```

3. 利用该配置启动rasa core server，如：`rasa run -m models --endpoints endpoints.yml`

**参数**：

- url（默认：localhost）：redis实例的url
- port（默认：6379）：redis服务运行的端口
- db（默认：0）：redis数据库的数目
- password（默认：None）：用于认证的密码（None，意味着不需要认证）
- record_exp（默认：None）：记录过期（秒）

## MongoTrackerStore

**描述**：MongoTrackerStore用来将对话历史存储到[Mongo](https://www.mongodb.com/)。Mongo是免费的开源的跨平台的面向文本的NoSQL数据库。

**配置**：

1. 启动MongoDB实例

2. 在endpoints.yml文件中添加配置，如下：

   ```
   tracker_store:
       type: mongod
       url: <url to your mongo instance, e.g. mongodb://localhost:27017>
       db: <name of the db within your mongo instance, e.g. rasa>
       username: <username used for authentication>
       password: <password used for authentication>
       auth_source: <database name associated with the user’s credentials>
   ```

3. 利用该配置启动rasa core server，如：`rasa run -m models --endpoints endpoints.yml`

**参数**：

- url（默认：mongodb://localhost:27017)：mongodb的url
- db（默认：rasa）：被使用的数据库的名字
- username（默认：0）：用于认证的用户名
- password（默认：None）：用于认证的密码
- collection（默认：conversations）：收集的用来存储对话的名字
- auth_source（默认：admin）：与用户凭据关联的数据库名称

## Custom Tracker Store

**描述**：如果您需要一个跟踪器存储，而不是现成的，您可以实现自己的。这是通过扩展基类trackerstore来完成的。`class rasa.core.tracker_store.TrackerStore(domain, event_broker=None)`。

**配置**：

1. 集成TrackerStore基类。你的构造函数中需要提供url参数

2. 在endpoints.yml文件中添加如下配置：

   ```
   tracker_store:
     type: path.to.your.module.Class
     url: localhost
     a_parameter: a value
     another_parameter: another value
   ```

## 原文链接

https://rasa.com/docs/rasa/api/tracker-stores/