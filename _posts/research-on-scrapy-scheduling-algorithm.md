---
title: Scrapy 调度算法研究
date: 2024-06-14 15:15
tags:
---

## 背景

Scrapy 是基于 Python 的自动爬虫框架，用户通过编写 `Spider` 类来定义爬取的行为，即提取内容并产生若干个 `Item` 或者新的 `Reuqest`。根据 [Architecture overview](https://docs.scrapy.org/en/latest/topics/architecture.html)，`Spider` 实例会不断产生 `Request` 对象发送给 `Engine`，`Engine` 再转发给 `Scheduler` 进行调度，之后向 `Scheduler` 请求一个 `Request`，此时`Scheduler` 会根据某种调度算法返回一个 `Request`，`Engine` 再转发给 `Downloader` 下载。

[`Scheduler`](https://docs.scrapy.org/en/latest/topics/scheduler.html#default-scrapy-scheduler) 在初始化的时候有 3 个队列类的参数，分别是 `dqclass`、`mqclass`、`pqclass`，类型提示都是 `class`。文档提供的默认值也没有用。在 GitHub 上找到了源代码才发现前面两个都是在 [squeues.py](https://github.com/scrapy/scrapy/blob/master/scrapy/squeues.py#L159) 里动态定义的，而 `pqclass` 定义在 [pqueues.py](https://github.com/scrapy/scrapy/blob/master/scrapy/pqueues.py)。

## squeues.py

观察前面的几个 import 语句：

```py
import marshal
import pickle  # nosec
from os import PathLike
from pathlib import Path
from typing import TYPE_CHECKING, Any, Callable, Optional, Type, Union

from queuelib import queue
```

大致猜到这个文件作用大概是定义了一些常用的队列。[marshal](https://docs.python.org/zh-cn/3/library/marshal.html) 和 [pickle](https://docs.python.org/zh-cn/3/library/pickle.html) 都是和序列化相关的库。[queuelib](https://github.com/scrapy/queuelib) 定义了一些常用的队列。

稍微解读一下其中的函数：
- `_with_mkdir`：类装饰器，给 disk-based 队列用的，作用是确认文件夹的存在，若不存在则创建文件夹。
- `_serializable_queue`：类装饰器，在入队、出队的时候进行序列化、反序列化操作。
- `_scrapy_serialization_queue`：`_serializable_queue` 的 Scrapy 定制版，主要是把类型注解换成了 `Request`，然后在序列化的时候使用了 `Request.to_dict` 和 `request_from_dict` 方法。
- `_scrapy_non_serialization_queue`：`_scrapy_serialization_queue` 的无序列化操作版本。
- `_pickle_serialize`：对 `pickle.dumps` 可能抛出的异常简单封装了一下。

以上函数最终产出了以下几个类：

```py
# queue.*Queue aren't subclasses of queue.BaseQueue
_PickleFifoSerializationDiskQueue = _serializable_queue(
    _with_mkdir(queue.FifoDiskQueue), _pickle_serialize, pickle.loads  # type: ignore[arg-type]
)
_PickleLifoSerializationDiskQueue = _serializable_queue(
    _with_mkdir(queue.LifoDiskQueue), _pickle_serialize, pickle.loads  # type: ignore[arg-type]
)
_MarshalFifoSerializationDiskQueue = _serializable_queue(
    _with_mkdir(queue.FifoDiskQueue), marshal.dumps, marshal.loads  # type: ignore[arg-type]
)
_MarshalLifoSerializationDiskQueue = _serializable_queue(
    _with_mkdir(queue.LifoDiskQueue), marshal.dumps, marshal.loads  # type: ignore[arg-type]
)

# public queue classes
PickleFifoDiskQueue = _scrapy_serialization_queue(_PickleFifoSerializationDiskQueue)
PickleLifoDiskQueue = _scrapy_serialization_queue(_PickleLifoSerializationDiskQueue)
MarshalFifoDiskQueue = _scrapy_serialization_queue(_MarshalFifoSerializationDiskQueue)
MarshalLifoDiskQueue = _scrapy_serialization_queue(_MarshalLifoSerializationDiskQueue)
FifoMemoryQueue = _scrapy_non_serialization_queue(queue.FifoMemoryQueue)  # type: ignore[arg-type]
LifoMemoryQueue = _scrapy_non_serialization_queue(queue.LifoMemoryQueue)  # type: ignore[arg-type]
```

可以看到都是非常基本的 FIFO 或者 LIFO 队列。

## pqueues.py

这个文件定义了基于优先级的优先队列。主要定义了 `ScrapyPriorityQueue` 和 `DownloaderAwarePriorityQueue` 两个类。

### ScrapyPriorityQueue

`ScrapyPriorityQueue` 提供了 `push` 和 `pop` 两个方法，就像传统的队列一样，但是因为要考虑到优先级，所以具体实现上又与传统的队列有区别。

`ScrapyPriorityQueue` 实例拥有一个用于指示当前优先级的 `curprio` 属性，以及与不同优先级对应的若干个 `queue`。`push` 方法除了将 `request` 入队相应的队列之外，还会调整 `curprio` 为 `request.priority`。`pop` 方法会根据  `curprio` 从相应的队列出队 `request` 作为返回值，但是在返回之前还会判断一下队列是否为空，如果是则重新调整 `curprio` 为可用的最小值。

### DownloaderAwarePriorityQueue

`DownloaderAwarePriorityQueue` 没太看懂，主要是因为 `Downloader` 没有文档。不过从 `DownloaderAwarePriorityQueue` 的文档中可以知道，它的机制是把持有下载最少的域名的 `request` 最先出队。

