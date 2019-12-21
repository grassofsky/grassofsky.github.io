+++
date="2019-12-21"
title="对话系统rasa - tracker"
categories=["chatbot","rasa","翻译"]
tags=["rasa-doc"]
+++

# 对话系统rasa - tracker

```python
class rasa.core.trackers.DialogueStateTracker(sender_id, slots, max_event_history=None)
```

用来维护对话状态。

max_event_history字段只提供这些最后的事件，它可以在tracker_store中设置

```python
applied_events()
```

返回所有应应用的操作- 不返回已还原事件。

`Return type: List [ Event ]`

```python
as_dialogue()
```

返回包含所有对话的Dialogue对象。

这可以被序列化，然后用来精确地恢复这个跟踪器的状态。

`Return type: Dialogue`

```python
change_form_to(form_name)
```

激活或失活一个form

`Return type: None`

```python
clear_followup_action()
```

清除执行时的后续操作。

`Return type: None`

```python
copy()
```

创建这个tracker的副本。

```python
current_slot_values()
```

返回当前设置的slots的值

`Return type: Dict [ str, Any ]`

```python
current_state(event_verbosity=<EventVerbosity.NONE: 1>)
```

作为对象返回当前tracker状态

`Return type: Dict [ str, Any ]`

```python
events_after_latest_restart()
```

返回最近重启的event列表

`Return type: List [ Event ]`

```python
export_stories(e2e=False)
```

将tracker作为story以Rasa core story格式导出

返回dumped的tracker

`Return type: str`

```python
export_stories_to_file(export_path='debug.md')
```

将tracker作为story导出到文件

`Return type: None`

```python
classmethod from_dict(sender_id, events_as_dict, slots=None, max_event_history=None)
```

从dump创建tracker

dump应该是dumped event的数组。当恢复tracker的时候，这些events将会重新执行来重新创建状态

`Return type: DialogueStateTracker`

```python
generate_all_prior_trackers()
```

返回当前tracker之前tracker的生成器。

结果数组表示每个action之前的trackers。

`Return type: Generator[DialogueStateTracker, None, None]`

```python
get_last_event_for(event_type, action_names_to_exclude=None, skip=0)
```

获取对应类型的，已经被执行过了的最近的event

**参数**：

- `event_type`: 你想要查找的event类型
- `action_names_to_exclude`: ActionExecuted类型的events将从结果中排除。可以被用来skip `action_listen` 事件。
- `skip`: 用来跳过返回一个event之前的n个可能结果。

**返回**：返回匹配的event，否则返回None

`Return type: Optional [Event]`

```python
get_latest_entity_values(entity_type)
```

在最新消息中为传递的实体名找到实体值。

如果你仅对第一个给定类型的实体感兴趣，使用`next(tracker.get_latest_entity_values("my_entity_name"), None)`。如果没有找到entity，返回None。

`Return type: Iterator [ str ]`

```python
get_latest_input_channel()
```

获取最新的UserUttered事件的input_channel的名字。

`Return type: Optional [ str ]`

```python
get_slot(key)
```

获取slot的值

`Return type: Optional [ Any ]`

```python
idx_after_latest_restart()
```

返回事件列表中最近重新启动的idx。

如果对话没有被重启，返回0.

`Return type: int`

```python
init_copy()
```

创建新的状态tracker，使用相同的初始化值。

`Return type: 	DialogueStateTracker`

```python
is_paused()
```

当前tracker是否被暂停的状态。

`Return type: bool`

```python
last_executed_action_has(name, skip=0)
```

返回上一个ActionExecuted event是否有特定的名字

**参数**：

- name - 需要匹配的名字
- skip - 跳过n个可能的结果

**返回**：如果上一次执行的action的名字是name，返回True，否则为False

`Return type: bool`

```python
past_states(domain)
```

根据历史生成该tracker的过去的状态。

`Return type: deque`

```python
recreate_from_dialogue(dialogue)
```

使用串行对话框更新跟踪器状态。

这将使用trackerstore中持久化的状态。如果在调用此方法之前跟踪器为空，则最终状态将与从中创建对话的跟踪器相同。

`Return type: None`

```python
reject_action(action_name)
```

通知active form已被拒绝

`Return type: None`

```python
replay_events()
```

根据一系列events更新tracker

`Return type: None`

```python
set_form_validation(validate)
```

触发form校验

`Return type: None`

```python
set_latest_action_name(action_name)
```

设置最新操作名称并重置表单验证和拒绝参数

`Return type: None`

```python
travel_back_in_time(target_time)
```

创建具有特定时间戳状态的新跟踪器。

将创建一个新的跟踪器，并重放传递时间戳之前的所有事件。将包括恰好在目标时间发生的事件。

`Return type: DialogueStateTracker`

```python
trigger_followup_action(action)
```

在执行当前操作后触发另一个操作。

`Return type: None`

```python
update(event, domain=None)
```

根据事件修改跟踪器的状态。

`Return type: None`

