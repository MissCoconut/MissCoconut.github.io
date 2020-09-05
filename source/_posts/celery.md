---
title: Celery 任务调度
date: 2020-08-18 02:09:49
tags: 
- python
- 分布式
---

## 1. Celery 

Celery 是一个 Python 分布式任务调度模块，支持实时处理异步消息队列，稳定高效易上手。 

### 模块架构

Celery 模块架构如下：

![image-20200817010421489](https://i.loli.net/2020/08/17/Nd4bmazQgtkCvKV.png)

<!-- more-->

### 1 - 任务生产者 (task producer)

调用 Celery 提供的 API，产生任务并交给任务队列处理的应用，一段 py 代码、一个 API 请求都可。

### 2 - 任务调度器 (celery beat)

Celery beat 是 Celery 系统自带的任务生产者，用于执行预先定义的定时任务。

### 3 - 任务代理 (broker)

消息队列中间件，Celery 官方推荐使用 rabbitmq，redis 也可做消息队列，在对稳定性要求不严格时可选。
### 4 - 任务消费方 (celery worker)

真正执行工作任务的。一个 worker 对应一个进程。Celery 几乎用了各种能用上的异步手段来提高并发任务的性能。

Celery 支持分布式部署和横向扩展，可以通过在多个节点增加 worker 的数量来增加系统的高可用性，也可以在不同节点上分配执行不同任务的 worker 来达到模块化的目的。

### 5 - 结果保存

Celery 支持任务处理完后将状态信息和结果的保存，以供查询。Celery 内置支持 rpc, Django ORM，Redis，RabbitMQ 等方式来保存任务处理后的状态信息。另外，Celery 还支持各种参加的序列化方式。

### 安装

`pip install -U Celery`

## 2. 本地任务调用

hello celery，这里参考官方文档，任务在本地，了解 Celery 调度任务的流程。

为了简单，这里 broker (消息队列中间件) 和 backend (保存任务结果的地方) 都用 redis，存放在不同的库里。

### 消费者进行消费

1. 启动消息队列中间件；
1. 消费者定义异步任务方法；
2. 启动 Celery，Celery 会去完成连接消息队列中间件、创建消息队列、监听队列，等等操作。


```python
# celery_task.py
# Celery task 消费者
import celery
import time

backend = "redis://0.0.0.0:xxxx/1"
broker = "redis://0.0.0.0:Xxxx/2"

coco_celery = celery.Celery("coconut", backend=backend, broker=broker)


# 模拟耗时操作
@coco_celery.task
def do_sth(s):
    time.sleep(s)
    return s


@coco_celery.task
def do_sth_b(s):
    time.sleep(s)
    return s
```

命令行启动 Celery 

 `celery worker -A celery_tasks -l info`

![Image](https://i.loli.net/2020/08/17/OlvauFGtZchHwWs.png)


### 生产者发消息

1. 配置 Celery ，指定 broker & backend ，指定任务队列（可选）；
2. import 进来要执行的任务，然后调用 `delay()`方法。

```python
# producer_task.py
from CeleryTest.celery_task import do_sth, do_sth_b

result = do_sth.delay(3)
print(f"worker task end {result}")

result2 = do_sth_b.delay(2)
print(f"worker task2 end {result2}")
```

一个`delay ()` 方法，Celery 会自动把异步任务函数名、参数等等发送到消息队列。生产者直接运行这个 py 文件就行了。

### 踩坑

生产者发送消息后，拿到了任务 id，但是消费者这边报错：

```
[2020-08-17 02:09:24,644: ERROR/MainProcess] Received unregistered task of type 'CeleryTest.celery_task.do_sth'.
The message has been ignored and discarded.

Did you remember to import the module containing this task?
Or maybe you're using relative imports?

Please see
http://docs.celeryq.org/en/latest/internals/protocol.html
for more information.

The full contents of the message body was:
b'[[3], {}, {"callbacks": null, "errbacks": null, "chain": null, "chord": null}]' (78b)
Traceback (most recent call last):
  File "d:\workspace\cocotest\venv2\lib\site-packages\celery\worker\consumer\consumer.py", line 562, in on_task_received
    strategy = strategies[type_]
KeyError: 'CeleryTest.celery_task.do_sth'
```

生产者这边 import 了这两个任务的方法，还是报错找不到这两个 task。

仔细看 `[2020-08-17 02:09:24,644: ERROR/MainProcess] Received unregistered task of type 'CeleryTest.celery_task.do_sth'.` 和前边消费者启动时候 Celery 找到的任务名不一样....

问题在目录结构上：

![image-20200817021434940](https://i.loli.net/2020/08/17/xnVfSr1vMokpeLJ.png)

现在的目录结构是这样，然鹅消费者启动 Celery 的时候命令是  `celery worker -A celery_tasks -l info` 改成，`celery worker -A CeleryTest.celery_task -l info` 之后解决。

![image-20200817023126097](https://i.loli.net/2020/08/17/k4iYhUmgCOErzKq.png)

### 要注意的地方

Celery 使用中需要特别注意目录结构，确保 Celery 能识别出来定义的任务。

典型的目录结构如：

![image-20200818020535964](https://i.loli.net/2020/08/18/cHT3YCt4gGy7l9P.png)

其中 1 放实际的任务，可以分包组织，注意保证文件名是 `tasks.py`；

2 保存 Celery 的各种配置；

3 中生成 Celery app对象，加载配置等。

```python
# main.py
from celery import Celery

app = Celery("mytasks")

app.config_from_object("CeleryTest.config")

app.autodiscover_tasks([
    "CeleryTest.recv_task.do_sth",
    "CeleryTest.send_task.do_sth_b"
])

# Celery 根目录下启动
# Celery -A CeleryTest.main worker -l=info
```

## 3. 远程任务调用

实际生产中，worker 方法 和 调度方法 很可能跑在不同的机器上，在这种情况下，Celery 通过 broker 实现远程任务调度。

使用起来也比较简单，首先给生产者和消费者指定相同的 broker 和 backend，然后使用 Celery 的任务队列功能规定好任务队列；生产者同样的方法发送消息，worker 在各个节点上跑起来后会自动执行。

### 生产者实现

- Celery 配置

```python
# worker.py 定义 celery app 
celery_app = Celery('celery',
                 broker=BROKER_CONN_URI,
                 backend=BACKEND_CONN_URI,
                 include=['celery.tasks']
                 )

# tasks.py 中定义任务队列
from kombu import Queue

CELERY_TIMEZONE='Asia/Shanghai'

celery_app.conf.task_queues = (
    Queue("task_a", routing_key='default'),
    Queue("task_b", routing_key='default'),
    Queue("task_c", routing_key='default')
)

CELERY_ROUTES = (
   [
     # 将 task_a 任务分配至队列 task_a
       ("remote_proj.tasks.task_a", {"queue": "task_a"}), 
     # 将 task_b 任务分配至队列 task_b
       ("remote_proj.tasks.task_b", {"queue": "task_b"}), 
     # 将 task_c 任务分配至队列 task_c
       ("remote_proj.tasks.task_c", {"queue": "task_c"})
   ],
)

CELERY_RESULT_SERIALIZER = "json"  # 指定任务结果序列化方式
CELERY_TASK_RESULT_EXPIRES = 60 * 60 * 24  # 任务结果过期时间
```

- 调用远程任务

```python
async def add_task(queue_name: str, params: dict) -> bool:
  """
  :return task ID
  """
   	task = kcelery.send_task("task_name",
                             kwargs=params,
                             queue="my_task_queue")
    return task
```

之后命令行启动 Celery

```
celery -A celery.worker worker -Q my_task_queue  --loglevel=info
```

### 消费者实现

```python
# worker.py 
# 定义 celery app , broker 和 backend 写和生产者一致的
celery_app = Celery('celery',
                 broker=BROKER_CONN_URI,
                 backend=BACKEND_CONN_URI,
                 include=['celery.tasks']
                 )

# 约定一致的 队列名、任务名
celery_app.conf.task_queues = (
   Queue("task_a", routing_key='default'),
    Queue("task_b", routing_key='default'),
    Queue("task_c", routing_key='default')
)

# 定义具体的任务
@celery_app.task(name="task_a")
def task_a(arg_a: str) -> str:
		time.sleep(5) # 模拟耗时操作
    return "result"
```

启动 celery 监听队列等待任务

```
celery -A ceelry.worker worker -Q scan_task --loglevel=info     
```



## 4. flower

flower 是一个可视化的监控 Celery 任务队列中任务状态的库，安装使用都比较简单。

```
pip install flower
celery flower -A celery.tasks --port=5555      
```
之后打开浏览器看 http://0.0.0.0:5555 即可。


## 参考链接

https://zhuanlan.zhihu.com/p/43768308