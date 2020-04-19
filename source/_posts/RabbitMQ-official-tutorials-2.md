---
title: RabbitMQ官方系列教程[翻译] - 2. Work queues (Python版)
date: 2019-09-21 05:09:52
categories: 消息队列
tags: [RabbitMQ,Python3]
---

![](https://kevin-blog-pic.oss-cn-shenzhen.aliyuncs.com/qsd0i.png!origin)
官方地址：https://www.rabbitmq.com/tutorials/tutorial-two-python.html

<!-- more -->

> ### 预先准备
>
> 这篇教程假定 RabbitMQ 已经完成[安装](https://www.rabbitmq.com/download.html)并且运行在 `localhost` 及标准端口 `5672` 上。如果你使用了一个不同的主机地址、端口或者凭据，则需要依此调整一下连接设置。

## 预先准备

与其他 Python 教程一样，我们将使用版本号为 [1.0.0](https://pika.readthedocs.io/en/stable/) 的 [Pika](https://pypi.python.org/pypi/pika) RabbitMQ 客户端。

![工作队列模型](https://kevin-blog-pic.oss-cn-shenzhen.aliyuncs.com/god0z.webp!origin)

## 本章重点

在[第一篇教程](https://www.keviny.site/2019/09/21/RabbitMQ-official-tutorials-1/)里，我们写了向一个已命名的队列发送并从中接受消息的两个程序。在这篇教程里，我们将创建一个 **Work Queue (工作队列)** ，该队列用于在多个 **workers (工人)** 之间分配耗时任务。

Work Queues (工作队列)(又名：**Task Queues (任务队列)**)的核心思想是，避免立即执行资源密集型任务并且不得不等待它完成的情形。相反的，我们安排任务稍后再完成。我们将 **task (任务)** 封装成一条消息，并将它发送到队列中去。在后台运行的工人进程将抛出任务并且最终执行这项作业。当你运行了很多工人进程时，任务会在它们之间共享。

这个概念在web应用程序中尤其有用，这些应用程序无法在一个短的HTTP请求窗口期间处理一个复杂的任务。

在这系列教程的前面部分我们发送了一条包含 "Hello World!" 字符串的消息。现在我们将发送代表复杂任务的字符串。我们并没有现实世界的任务，例如要调整图片的大小或着要渲染pdf文件，因此让我们使用 `time.sleep()` 函数来使自己假装很忙。我们将字符串中点的数量当作其复杂度；每个点作为"工作"的其中一秒。举个例子，一个形如 `Hello...` 的假任务会占用 3 秒钟的时间。

我们将稍微修改一下前面例子中 **send.py** 的代码以允许从命令行中发送任意消息。该程序会将任务安排到我们的工作队列中，所以我们将它命名为 `new_task.py` ：

```python
import sys

message = ' '.join(sys.argv[1:]) or "Hello World!"
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body=message)
print(" [x] Sent %r" % message)
```

我们的老脚本 **receive.py** 也需要做一些改变：它需要将消息主体中的每一个点伪造成 1 秒钟的工作。它会从队列中抛出消息并执行任务，因此我们将它命名为 `worker.py` 。

```python
import time

def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)
    time.sleep(body.count(b'.'))
    print(" [x] Done")
```

## 循环调度

使用任务队列的优点之一是能够轻松地并行化工作。如果我们正在积压工作，那么我们只需要增加更多的工人，这样就可以轻松扩展了。

首先，让我们试着同时运行两个 `worker.py` 脚本。它们都会从队列中获取消息，但究竟是怎样的呢？让我们拭目以待。

你需要打开三个控制台程序。其中两个运行 `worker.py` 脚本。这些控制台程序就是我们的两个消费者 - C1 和 C2。

```bash
# shell 1
python worker.py
# => [*] Waiting for messages. To exit press CTRL+C
```

``` bash
# shell 2
python worker.py
# => [*] Waiting for messages. To exit press CTRL+C
```

在第三个控制台程序中，我们将要发布新的任务。一旦你启动了消费者程序，你就可以发布一些消息：

```bash
# shell 3
python new_task.py First message.
python new_task.py Second message..
python new_task.py Third message...
python new_task.py Fourth message....
python new_task.py Fifth message.....
```

让我们看看传递给工人们的是什么吧：

```bash
# shell 1
python worker.py
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'First message.'
# => [x] Received 'Third message...'
# => [x] Received 'Fifth message.....'
```

```bash
# shell 2
python worker.py
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'Second message..'
# => [x] Received 'Fourth message....'
```

默认情况下，RabbitMQ 会按顺序将每条消息发送给下一个消费者。平均而言，每一个消费者都会接收到相同数量的消息。这种分发消息的方式称为循环。使用三个工人或更多进行尝试。

## 消息确认

执行任务会需要几秒钟。你可能会好奇，如果一个消费者启动了一个长耗时的任务并且在仅完成了部分任务的情况下死掉，会发生什么情况。就我们现在的代码而言，一旦 RabbitMQ 将消息传递给消费者，它会立即将这条消息标记为删除。在这种情况下，如果你杀死了一个工人，我们将会丢失其正在处理的信息。我们还会丢失所有发送给该特定工人旦尚未处理的消息。

但我们不想丢失任何任务。如果一个工人死了，我们想把任务传递给其他工人。

RabbitMQ 支持[消息确认](https://www.rabbitmq.com/confirms.html)以确保任务永远不会丢失。消费者会发送回一条确认消息以通知 RabbitMQ 一条特定的消息已经被接收和处理，这样 RabbitMQ 就可以自由地删除这条消息。

如果一个工人死了(其通道已关闭、连接已关闭或者 TCP 连接丢失)而没有发送一条确认消息，RabbitMQ 就会知道该消息并没有被完全处理，从而将其重新放入队列中。如果同时还有其他消费者在线，它会快速的将该消息重新传递给其他消费者。这样就可以确保即使有工人偶然死亡也不会有任何消息丢失。

消息没有超时时间；当有消费者死亡时，RabbitMQ 会将消息重新传递。即使处理一条消息会花费非常漫长的时间也没关系。

默认情况下，[手动消息确认](https://www.rabbitmq.com/confirms.html)已经打开。在前面的例子中，我们通过 `auto_ack=True` 标签显式地关闭了它们。当我们完成了一个任务时，是时候移除这个标签并让工人发送适当的确认消息了。

```python
def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)
    time.sleep( body.count('.') )
    print(" [x] Done")
    ch.basic_ack(delivery_tag = method.delivery_tag)

channel.basic_consume(queue='hello', on_message_callback=callback)
```

使用此代码，我们可以确保即使你在一个工人处理消息时使用 CTRL+C 来杀死他，也不会丢失任何东西。在工人死亡不久后，所有未经确认的消息都会被重新传递。

确认消息必须在收到消息的同一通道中发送。试图使用不同的通道进行确认将导致通道级协议的异常。请参阅[有关确认的文档指南](https://www.rabbitmq.com/confirms.html)以了解更多信息。

> ### 忘记确认
>
> 错过 `basic_ack` 是一个常见的错误。这是个简单的错误，但后果却很严重。当你的客户端退出时，消息会被重新传递(看起来像随机传递一样)，但 RabbitMQ 会消耗越来越多的内存，因为其无法释放任何未经确认的消息。
>
> 为了调试这种错误，你可以使用 `rabbitmqctl` 来打印 `messages_unacknowledged` 字段：

```bash
sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
```

> 在 Windows 平台上，去掉 sudo ：

```bash
rabbitmqctl.bat list_queues name messages_ready messages_unacknowledged
```

## 消息持久性

我们已经学习了如何确保即使在消费者死亡的情况下，任务也不会丢失。但若 RabbitMQ 服务器停止工作，我们的任务仍然会丢失。

在 RabbitMQ 退出或崩溃时，它会丢失其中的队列和消息，除非你让它不要这样做。为了确保消息不会丢失，需要做两件事：我们需要将队列和消息都标记为持久。

首先，我们需要确保 RabbitMQ 永远不会丢失我们的队列。为了实现这一点，我们需要将此队列声明为 **durable (持久)** 。

```python
channel.queue_declare(queue='hello', durable=True)
```

尽管此命令本身是正确的，但在我们的设置中，它并不会生效。这是因为我们已经声明了一个名为 `hello` 但不持久的队列。RabbitMQ 不允许使用不同的参数重新定义一个现有队列，并且将向尝试进行此项操作的程序返回一个错误。但是有一个快速解决此问题的方法 - 让我们声明一个使用不同名称的队列，比如 `task_queue` ：

```python
channel.queue_declare(queue='task_queue', durable=True)
```

这个 `queue_declare` 更改需要同时应用于生产者和消费者代码。

此时，我们可以确定即使 RabbitMQ 重新启动，`task_queue` 队列也不会丢失。现在我们需要将我们的消息标记为持久消息 - 通过赋予一个值为 `2` 的 `delivery_mode` 属性。

```python
channel.basic_publish(exchange='',
                      routing_key="task_queue",
                      body=message,
                      properties=pika.BasicProperties(
                         delivery_mode = 2, # make message persistent
                      ))
```

> ### 关于消息持久性的说明
>
> 将消息标记为持久并不完全能保证消息不会丢失。尽管其告知 RabbitMQ 将消息保存到磁盘，但 RabbitMQ 接收到消息并尚未完成保存时仍会有一个短暂的时间窗口。而且，RabbitMQ 并不会对每一条消息都执行 `fsync(2)` -- 它可能只是被保存到缓存中而并没有真正写入到磁盘上。持久性的保证并不强，但这对于简单的任务队列来说已经绰绰有余了。如果你需要更强有力的保证，可以使用[发布者确认机制](https://www.rabbitmq.com/confirms.html)。

## 公平派遣

你可能已经注意到了，调度仍然无法完全按照我们的要求来进行。例如，在有两个工人的情况下，当所有单数消息都很庞大而双数消息都很轻量时，一个工人会一直非常忙碌，而另一个工人却几乎不做什么工作。好吧，RabbitMQ 对此一无所知，它依然只会平均分配消息。

出现这种情况的原因是，RabbitMQ 在消息进入队列的时候只是将其分配出去。它并不会查看消费者未确认消息的数量。它只会盲目地将每第 n 条消息发送给第 n 个消费者。

![公平派遣模型](https://kevin-blog-pic.oss-cn-shenzhen.aliyuncs.com/8gjj6.webp!origin)

为了克服这一问题，我们可以将 `basic.qos` 方法与 `prefetch_count=1` 设置一起使用。这会告诉 RabbitMQ 一次不要给工人一条以上的消息。换言之，在工人完成处理并确认上一条消息前不要再给它分配一条新的消息。而是将消息分配给下一个不忙的工人。

```python
channel.basic_qos(prefetch_count=1)
```

> ### 关于队列大小的说明
>
> 如果所有的工人都处于忙碌状态，你的队列就已经满了。你需要注意这一点，可能采用增加新工人的方法或着设置[消息的TTL](https://www.rabbitmq.com/ttl.html)。

## 代码汇总

`new_task.py` ([source](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/new_task.py))

```python
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='task_queue', durable=True)

message = ' '.join(sys.argv[1:]) or "Hello World!"
channel.basic_publish(
    exchange='',
    routing_key='task_queue',
    body=message,
    properties=pika.BasicProperties(
        delivery_mode=2,  # make message persistent
    ))
print(" [x] Sent %r" % message)
connection.close()
```

`worker.py` ([source](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/worker.py))

```python
#!/usr/bin/env python
import pika
import time

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='task_queue', durable=True)
print(' [*] Waiting for messages. To exit press CTRL+C')


def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)
    time.sleep(body.count(b'.'))
    print(" [x] Done")
    ch.basic_ack(delivery_tag=method.delivery_tag)


channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue='task_queue', on_message_callback=callback)

channel.start_consuming()
```

使用消息确认机制以及设置 `prefetch_count` 参数可以建立一个工作队列。持久化选项使得任务即使在 RabbitMQ 重启的情况下也依然能存活。

**是时候前往[第 3 部分](https://www.rabbitmq.com/tutorials/tutorial-three-python.html)去学习如何将相同的消息发送给多个消费者了。**

## 生产[非]适用性免责声明

请记住，本文或其他教程都只是教程。教程一次展示一个新的概念，并且可能会故意过度简化某些东西而忽略了其它的事物。例如，为了简洁起见，有关连接管理、错误处理、连接恢复、并发和度量收集的内容都在很大程度上被忽略了。这样简化过的代码不应该被当作是已经完成的产品。

在开发您自己的应用程序之前，请先查看[文档](https://www.rabbitmq.com/documentation.html)的剩余部分。我们特别推荐以下部分：[Publisher Confirms and Consumer Acknowledgements](https://www.rabbitmq.com/confirms.html)， [Production Checklist](https://www.rabbitmq.com/production-checklist.html) 以及 [Monitoring](https://www.rabbitmq.com/monitoring.html) 。

## 获得帮助并提供反馈

如果您对本教程或有关 RabbitMQ 的其他内容感到疑问，请立即在 [RabbitMQ mailing list](https://groups.google.com/forum/#!forum/rabbitmq-users) 上提问。

## 帮助我们完善 3 版本以下的文档

如果你想对本网站做出改进，可以在 [Github](https://github.com/rabbitmq/rabbitmq-website) 上找到它的源码。只需要 fork 源仓库并提交一条 pull 请求。非常感谢！

