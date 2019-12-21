+++
date="2019-12-21"
title="rasa对话系统demo实战 - 简单的火车票查询助手"
categories=["chatbot","rasa"]
tags=["rasa-example"]
+++

# rasa对话系统Demo实战 - 简单的火车票查询助手

##  描述

### 目的

基于rasa构建用于火车票查询的对话系统Demo。

### 对话示例

```
bot> 您好，欢迎使用火车票查询助手。
bot> 请问，您的出发时间是？
user> 明天上午11:00
bot> 请问，您的出发地点是？
user> 武汉
bot> 请问，您的目的地是？
user> 上海
bot> 您好，您需要查询的信息是：
     2019-11-01 11:00 武汉到上海的车票信息
     您的查询结果是：
     1. ...
     2. ...
```

## 详细实现

对话设计的相关内容可以参见：https://zhuanlan.zhihu.com/p/83203455。此处没有涉及很多的对话设计。

### 意图设计

针对火车票预订的对话系统，需要识别的用户意图有：

- 对话触发，start

### nlu设计

由于该示例作为简单的任务型对话系统，以form的形式进行，没有对nlu进行设计。

### stories设计

```
## happy path
* start
  - utter_welcome
  - ticket_form
  - form{"name": "ticket_form"}
  - form{"name": null}
  - action_show_result
```

### domain内容

```
intents:
  - start
  
slots:
  starting_time:
    type: unfeaturized
    auto_fill: false
  starting_loc:
    type: unfeaturized
    auto_fill: false
  arriving_loc:
    type: unfeaturized
    auto_fill: false

forms:
  - ticket_form
  
actions:
  - utter_welcome
  - action_show_result
  
templates:
  utter_welcome:
    - text: "您好，欢迎使用火车票查询助手。"
  utter_ask_starting_time:
    - text: "请问，您的出发时间是？"
  utter_ask_starting_loc:
    - text: "请问，您的出发地点是？"
  utter_ask_arriving_loc:
    - text: "请问，您的目的地是？"
  utter_default:
    - text: "我的大脑不够用了，请重试！"
```

### config实现

```
language: zh
pipeline:
  - name: JiebaTokenizer
  - name: CRFEntityExtractor
  - name: "rasa_ext.extractors.regex_date_extractor.RegexDateExtractor"
    entity_name: "starting_time"
  - name: CountVectorsFeaturizer
  - name: EmbeddingIntentClassifier
  
policies:
  - name: FormPolicy
  - name MemorizationPolicy
    max_history: 1
  - name: KerasPolicy
```

### RegexDateExtractor实现

此处说明为了介绍如何对Rasa的组件进行扩展，给出了基于正则表达式日期提取的Extractor实现，代码如下：

```python
from typing import Any, Dict, Optional, Text, List

from rasa.nlu.extractors import EntityExtractor
from rasa.nlu.model import Metadata
from rasa.nlu.training_data import Message, TrainingData

import rasa_ext.utils.date_parser as data_parser

class RegexDateExtractor(EntityExtractor):
    provides = ["entities"]
    
    defaults = {
        "entity_name": "time"
    }
    
    def __init__(
    	self,
        component_config: Optional[Dict[Text, Any]] = None,
    ) -> None:
        super(RegexDateExtractor, self).__init__(component_config)
        
    def process(self, message: Message) -> List[Dict[Text, Any]]:
        json_ents = []
        date_times = date_parser.time_extract(message.text)
        for date_time in date_times:
            json_ents.append({
                'start': -1,
                'end': -1,
                'value': date_time,
                'entity': self.component_config['entity_name']
            })
        message.set(
        	"entities", message.get("entities", []) + json_ents, add_to_output=True
        )
```

其中，date_parser的实现见参考：https://blog.csdn.net/lilong117194/article/details/81212961。

### action的实现

action文件中需要实现两个action，TicketForm和ActionShowResult，其中关于车票信息的查询见：https://zhuanlan.zhihu.com/p/27969976。具体如下：

**此处需要注意的是**：查票信息查询https://zhuanlan.zhihu.com/p/27969976中的实现，没法正常获取车票信息，cry。**请高手帮忙**

```python
import pdb
import logging
from typing import Dict, Text, Any, List, Union, Optional

from rasa_sdk import Tracker
from rasa_sdk import Action
from rasa_sdk.executor import CollectingDispatcher
from rasa_sdk.forms import FormAction
from rasa_sdk.events import Form, AllSlotsReset

import requests
import re

EventType = Dict[Text, Any]
logger = logging.getLogger(__name__)

class TicketForm(FormAction):
    """火车票查询的action"""
    
    def name(self) -> Text:
        return "ticket_form"

    @staticmethod
    def required_slots(tracker: Tracker) -> List[Text]:
        """A list of required slots that the form has to fill"""

        return ["starting_time", "starting_loc", "arriving_loc"]

    def submit(
        self,
        dispatcher: CollectingDispatcher,
        tracker: Tracker,
        domain: Dict[Text, Any],
        ) -> List[Dict]:
        """Define what the form has to do
           after all required slots are filled"""
        dispatcher.utter_template("utter_submit", tracker)
        return []

    def slot_mappings(self) -> Dict[Text, Union[Dict, List[Dict]]]:
        return {
            "starting_time": self.from_entity(entity="starting_time"),
            "starting_loc": self.from_text(),
            "arriving_loc": self.from_text()
        }


class ActionShowResult(Action):
    def __init__(self):
        requests.packages.urllib3.disable_warnings()
        # 12306的城市名和城市代码js文件url
        url = 'https://kyfw.12306.cn/otn/resources/js/framework/station_name.js?station_version=1.9018'
        r = requests.get(url,verify=False)
        pattern = u'([\u4e00-\u9fa5]+)\|([A-Z]+)'
        result = re.findall(pattern,r.text)
        self.station = dict(result)

    def name(self) -> Text:
        return "action_show_result"

    def run(
            self, dispatcher, tracker: Tracker, domain: Dict[Text, Any]
    ) -> List[Dict[Text, Any]]:
       """Execute the side effects of this action.

        Args:
            dispatcher: the dispatcher which is used to
                send messages back to the user. Use
                ``dipatcher.utter_message()`` or any other
                ``rasa_sdk.executor.CollectingDispatcher``
                method.
            tracker: the state tracker for the current
                user. You can access slot values using
                ``tracker.get_slot(slot_name)``, the most recent user message
                is ``tracker.latest_message.text`` and any other
                ``rasa_sdk.Tracker`` property.
            domain: the bot's domain
        Returns:
            A dictionary of ``rasa_sdk.events.Event`` instances that is
                returned through the endpoint
        """
        starting_time = tracker.get_slot('starting_time')
        starting_loc = tracker.get_slot('starting_loc')
        arriving_loc = tracker.get_slot('arriving_loc')

        prefix_info = '您好，您需要查询的信息是:\n {} {}到{}的车票信息\n'.format(starting_time, starting_loc, arriving_loc)
        
        list_starting_time = starting_time.split(' ')
        date_time = list_starting_time[0]
        hour_time = list_starting_time[1]

        pdb.set_trace()
        url = self.get_query_url(date_time, starting_loc, arriving_loc)
        info_list = self.query_train_info(url, hour_time)

        if type(info_list) == list:
            dispatcher.utter_message(prefix_info + '\n'.join(info_list))
        else:
            dispatcher.utter_message(prefix_info + info_list)

        return []

    def get_query_url(self, date_time, starting_loc, arriving_loc):
        '''
        返回调用api的url链接
        '''
        try:
            date = date_time
            from_station_name = starting_loc
            to_station_name = arriving_loc
            from_station=self.station[from_station_name]
            to_station = self.station[to_station_name]
        except:
            date,from_station,to_station='--','--','--'

        # api url 构造
        url = (
            'https://kyfw.12306.cn/otn/leftTicket/query?'
            'leftTicketDTO.train_date={}&'
            'leftTicketDTO.from_station={}&'
            'leftTicketDTO.to_station={}&'
            'purpose_codes=ADULT'
        ).format(date, from_station, to_station)
        logger.debug(url)

        return url

    def query_train_info(self, url, hour_time):
    '''
        查询火车票信息：
        返回 信息查询列表
        '''
        time_list = hour_time.split(':')
        total_minutes = int(time_list[0])*60 + int(time_list[1])

        info_list = []
        try:
            r = requests.get(url, verify=False)
            # 获取返回的json数据里的data字段的result结果
            raw_trains = r.json()['data']['result']

            for raw_train in raw_trains:
                # 循环遍历每辆列车的信息
                data_list = raw_train.split('|')

                # 车次号码
                train_no = data_list[3]
                # 出发站
                from_station_code = data_list[6]
                from_station_name = code_dict[from_station_code]
                # 终点站
                to_station_code = data_list[7]
                to_station_name = code_dict[to_station_code]
                # 出发时间
                start_time = data_list[8]
                start_time_vec = start_time.split(':')
                start_total_minutes = int(start_time_vec[0])*60 + int(start_time_vec[1])
                # 忽略超过120分钟的车次
                if abs(start_total_minutes - total_minutes) > 120:
                    continue

                # 到达时间
                arrive_time = data_list[9]
                # 总耗时
                time_fucked_up = data_list[10]
                # 一等座
                first_class_seat = data_list[31] or '--'
                # 二等座
                second_class_seat = data_list[30]or '--'
                # 软卧
                soft_sleep = data_list[23]or '--'
                # 硬卧
                hard_sleep = data_list[28]or '--'
                # 硬座
                hard_seat = data_list[29]or '--'
                # 无座
                no_seat = data_list[26]or '--'

                # 打印查询结果
                info = ('车次:{}\n出发站:{}\n目的地:{}\n出发时间:{}\n到达时间:{}\n消耗时间:{}\n座位情况：\n 一等座：「{}」 \n二等座：「{}」\n软卧：「{}」\n硬卧：「{}」\n硬座：「{}」\n无座：「{}」\n\n'.format(
                    train_no, from_station_name, to_station_name, start_time, arrive_time, time_fucked_up, first_class_seat,
                    second_class_seat, soft_sleep, hard_sleep, hard_seat, no_seat))

                info_list.append(info)

            return info_list
        except:
            return '12306查询失败，请重试'

```

**另外需要注意的是：在使用demo运行过程中，如果需要进行多次查询，中间需要执行`/restart`，将slot信息重置。**

## 参考

- 日期实体提取：https://blog.csdn.net/lilong117194/article/details/81212961
- 查票信息查询：https://zhuanlan.zhihu.com/p/27969976





