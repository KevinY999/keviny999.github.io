---
title: RabbitMQ官方系列教程[翻译] - 3. Publish / Subscribe (Python版)
date: 2019-09-22 00:46:31
categories: 消息队列
tags: [RabbitMQ,Python3]
---

![](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/qsd0i.png)
官方地址：https://www.rabbitmq.com/tutorials/tutorial-three-python.html

<!-- more -->

> ### 预先准备
>
> 这篇教程假定 RabbitMQ 已经完成[安装](https://www.rabbitmq.com/download.html)并且运行在 `localhost` 及标准端口 `5672` 上。如果你使用了一个不同的主机地址、端口或者凭据，则需要依此调整一下连接设置。

## 预先准备

与其他 Python 教程一样，我们将使用版本号为 [1.0.0](https://pika.readthedocs.io/en/stable/) 的 [Pika](https://pypi.python.org/pypi/pika) RabbitMQ 客户端。

## 本章重点

在[上一篇教程](https://www.keviny.site/2019/09/21/RabbitMQ-official-tutorials-2/)里我们建立了一个工作队列。工作队列背后的假设是，每个任务都会被传递给一个工人。在这部分，我们会做一点完全不一样的事 -- 我们会将一条消息发送给多个消费者。这种模式称为 **"publish/subscribe (发布/订阅)"** 。

为了说明这种模式，我们要构建一个简单的日志系统。它由两个程序组成 -- 第一个程序会发出日志消息，第二个程序则接收并打印这些消息。

在我们的日志系统中，接收程序的每个运行副本都将收到消息。这样，我们就能够运行一个接收器并将日志定向到磁盘中；同时运行另一个接收器并在屏幕上查看日志。

从本质上说，发布出去的日志消息会被广播给所有的接收者。

## 交换器

在本教程的前面部分我们向一个队列发送消息并从中接收消息。现在是时候来介绍一下 Rabbit 中完整的消息传递模型了。

让我们快速回顾一下在前面的教程中介绍过的内容：

* **producer (生产者)** 是用来发送消息的用户应用程序。
* **queue (队列)** 是用来存储消息的缓存。
* **consumer (消费者)** 是用来接收消息的用户应用程序。

RabbitMQ 中消息传递模型的核心思想是生产者永远不会直接将任何消息发送到队列中。实际上，生产者常常完全都不知道消息是否会被传递到一个队列中去。

相反的，生产者只能将消息发送到 **exchange (交换器)** 中。交换器是一个非常简单的东西。一方面，它从生产者那里接收消息，另一方面，它将消息推送到队列中去。交换器必须确切地知道该如何处理接收到的消息。是否应该将其附加到一个特定的队列中？是否应该将其附加到许多的队列中？还是应该丢弃它呢？其中的规则由 **exchange type (交换器类型)** 来定义。

![交换器模型](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/41y33.webp)

有几种可用的交换器类型：`direct`，`topic`，`headers` 和 `fanout` 。我们主要来看最后一个 -- fanout 。让我们使用这种类型创建一个交换器，并称之为 `logs` ：

```python
channel.exchange_declare(exchange='logs',
                         exchange_type='fanout')
```

这种 fanout 交换器非常简单。正如你从这名字中可能会猜到的，它只是将所有它接受到的消息广播给所有它知道的队列。而这正是我们的日志记录器所需要的。

> ### 列举出交换器
>
> 为了在服务器上列出交换器，你可以使用那个非常有用的工具 `rabbitmqctl` ：

```bash
sudo rabbitmqctl list_exchanges
```

> 在列表中可能会有一些形如 `amq.*` 的交换器及默认(未命名)的交换器。这些是默认创建的，但此刻你可能不太需要用到。
>
> ### 默认交换器
>
> 在本教程的前面部分我们对交换器一无所知，但仍然可以给队列发送消息。之所以可以这样是因为我们使用了一个由空字符串(`""`)来标识的默认交换器。
>
> 回想一下之前我们是怎样发布消息的：

```python
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body=message)
```

> `exchange` 参数是交换器的名称。空字符串标识的是默认或未命名的交换器：消息被路由到由 `routing_key` 指定的队列中去(如果该队列存在的话)。

现在，我们可以改为将消息发布到命名的交换器中：

```python
channel.basic_publish(exchange='logs',
                      routing_key='',
                      body=message)
```

## 临时队列

你可能还记得，之前我们使用已经指定了名称的队列(还记得 `hello` 和 `task_queue` 嘛？)。能够命名队列对我们来说是至关重要的 -- 我们需要将工人指向同一个队列。当你想要在生产者和消费者之间共享队列时，给队列命名是非常重要的。

但对于我们的日志记录器来说并不是这样的。我们希望监听所有的日志消息而不仅仅是其中的一部分。我们也只对当前正在发送的消息感兴趣，而非旧的消息。为了解决这一问题我们需要两样东西。

首先，无论何时我们连接到 Rabbit ，我们都需要一个全新的空队列。为此，我们可以创建一个随机命名的队列，或者，更好的 - 让服务器为我们选择一个随机的队列名称。为此，我们可以给 `queue_declare` 提供一个空的 `queue` 参数。

```python
result = channel.queue_declare(queue='')
```

此时，`result.method.queue` 中包含了一个随机的队列名称。例如，它可能看起来像 `amq.gen-JzTY20BRgKO-HjmUJj0wLg` 。

其次，一旦消费者连接关闭，队列就应该被删除。对此有一个 `exclusive` 标志：

```python
result = channel.queue_declare(queue='', exclusive=True)
```

你可以在[队列指南](https://www.rabbitmq.com/queues.html)上了解到更多有关 `exclusive` 标志和其他队列属性的内容。

## 绑定

![绑定模型](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/caxhq.png)

我们已经创建了一个 fanout 交换器和一个队列。现在我们需要告诉交换器将消息发送到我们的队列中。其中交换器和队列的关系称为绑定。

```python
channel.queue_bind(exchange='logs',
                   queue=result.method.queue)
```

从现在起，`log` 交换器会将消息附加到我们的队列中。

> ### 列举出绑定
>
> 你大概能猜到使用如下的方式列举出已经存在的绑定：

```bash
rabbitmqctl list_bindings
```

## 代码汇总

![总体架构模型](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/0f8f9.png)

用于发送日志消息的生产者程序跟前面教程中的看起来没有很大的不同。其中最重要的变化是，我们现在希望将消息发布到名为 `log` 的交换器而非未命名的交换器。在发送消息时我们需要提供 `routing_key` ，但对于 `fanout` 交换，它的值将被忽略。

`emit_log.py` ([source](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/emit_log.py))

```python	
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='logs', exchange_type='fanout')

message = ' '.join(sys.argv[1:]) or "info: Hello World!"
channel.basic_publish(exchange='logs', routing_key='', body=message)
print(" [x] Sent %r" % message)
connection.close()
```

如你所见，在建立连接之后，我们才声明了这个交换器。这一步是必要的，因为将消息发布到一个不存在的交换器是被禁止的。

如果没有队列绑定到交换器上，那么消息将会丢失，但这对于我们来说是没问题的。如果没有消费者在监听，我们可以安全地丢弃该消息。

`receive_logs.py` ([source](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/receive_logs.py))

```python
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='logs', exchange_type='fanout')

result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue

channel.queue_bind(exchange='logs', queue=queue_name)

print(' [*] Waiting for logs. To exit press CTRL+C')

def callback(ch, method, properties, body):
    print(" [x] %r" % body)

channel.basic_consume(
    queue=queue_name, on_message_callback=callback, auto_ack=True)

channel.start_consuming()
```

我们完成了。如果你想将日志保存为一个文件，只需要打开一个控制台并输入：

```bash
python receive_logs.py > logs_from_rabbit.log
```

如果你想在屏幕上查看日志，请生成一个新的终端并运行：

```bash
python receive_logs.py
```

当然，要发送日志请输入：

```bash
python emit_log.py
```

使用 `rabbitmqctl list_bindings` 命令，你可以验证代码是否确切地按照我们的要求创建了绑定和队列。随着两个 `receive_logs.py` 程序的执行，你应该可以看到一些类似于这样的东西：

```bash
sudo rabbitmqctl list_bindings
# => Listing bindings ...
# => logs    exchange        amq.gen-JzTY20BRgKO-HjmUJj0wLg  queue           []
# => logs    exchange        amq.gen-vso0PVvyiRIL2WoV3i48Yg  queue           []
# => ...done.
```

关于此结果的解释很简单：数据从 `logs` 交换器进入到两个由服务器分配名称的队列中。而这正是我们的意图。

想了解如何监听消息的一部分，请前往[教程的第 4 部分](https://www.rabbitmq.com/tutorials/tutorial-four-python.html)。

## 生产[非]适用性免责声明

请记住，本文或其他教程都只是教程。教程一次展示一个新的概念，并且可能会故意过度简化某些东西而忽略了其它的事物。例如，为了简洁起见，有关连接管理、错误处理、连接恢复、并发和度量收集的内容都在很大程度上被忽略了。这样简化过的代码不应该被当作是已经完成的产品。

在开发您自己的应用程序之前，请先查看[文档](https://www.rabbitmq.com/documentation.html)的剩余部分。我们特别推荐以下部分：[Publisher Confirms and Consumer Acknowledgements](https://www.rabbitmq.com/confirms.html)， [Production Checklist](https://www.rabbitmq.com/production-checklist.html) 以及 [Monitoring](https://www.rabbitmq.com/monitoring.html) 。

## 获得帮助并提供反馈

如果您对本教程或有关 RabbitMQ 的其他内容感到疑问，请立即在 [RabbitMQ mailing list](https://groups.google.com/forum/#!forum/rabbitmq-users) 上提问。

## 帮助我们完善 3 版本以下的文档

如果你想对本网站做出改进，可以在 [Github](https://github.com/rabbitmq/rabbitmq-website) 上找到它的源码。只需要 fork 源仓库并提交一条 pull 请求。非常感谢！