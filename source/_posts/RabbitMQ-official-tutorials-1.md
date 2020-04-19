---
title: RabbitMQ官方系列教程[翻译] - 1. "Hello World!" (Python版)
date: 2019-09-21 00:02:42
categories: 消息队列
tags: [RabbitMQ,Python3]
---

![](https://kevin-blog-pic.oss-cn-shenzhen.aliyuncs.com/qsd0i.png!origin)
官方地址：https://www.rabbitmq.com/tutorials/tutorial-one-python.html

<!-- more -->

> ### 预先准备
>
> 这篇教程假定 RabbitMQ 已经完成[安装](https://www.rabbitmq.com/download.html)并且运行在 `localhost` 及标准端口 `5672` 上。如果你使用了一个不同的主机地址、端口或者凭据，则需要依此调整一下连接设置。

##  介绍

RabbitMQ是一个消息协商器(Message Broker，简称MQ)：它接受和转发消息。你可以把它当作是一个邮局：当你把想要寄出去的信件放到邮筒里时，你可以确信邮差最终会将你的信件送达你的收件人手上。在这个比喻中，RabbitMQ 就是那个邮筒、邮局以及邮差。

RabbitMQ 和邮局最主要的区别在于，RabbitMQ 处理的并不是纸质文件，而是接收、存储和转发二进制 blob 数据 - messages (消息)。

RabbitMQ 和消息传递会使用到一些专业术语。

* **Producing** 指的是发送。一个负责发送消息的程序就是一个 **producer** ：

![生产者](https://kevin-blog-pic.oss-cn-shenzhen.aliyuncs.com/12lv4.webp!origin)

* **Queue** 是邮筒在 RabbitMQ 中的名字。尽管消息会流经 RabbitMQ 和你的程序，但它们只能被储存在一个 queue (队列)中。队列只约束于主机内存和磁盘限制，它本质上是一个大的消息缓冲区。许多的 **producers** (生产者)可以将消息发送到一个队列中，许多的 **consumers** (消费者)也可以尝试从一个队列中获取数据。我们是这样表示一个队列的：

![队列](https://kevin-blog-pic.oss-cn-shenzhen.aliyuncs.com/b1lfl.webp!origin)

* **Consuming** 和接收的意思类似。**consumer** (消费者)是一个主要用来等待接收消息的程序：

![消费者](https://kevin-blog-pic.oss-cn-shenzhen.aliyuncs.com/8x9cc.webp!origin)

请注意，producer (生产者)、consumer (消费者)以及 broker (协商器)并非一定要存在于同一台主机上；实际上，在大多数的程序中，它们并没有放在一起。一个应用程序即可以是生产者，也可以是消费者。

## Hello World!

**(使用 Python 的 Pika 客户端)**

在教程的这个部分，我们将要用 Python 写两个小的程序：一个生产者(发送者)程序，用来发送单条信息；以及一个消费者(接收者)程序，用来接收消息并打印。这是消息传递的"Hello World"级程序。

在下图中，"P"是我们的生产者，"C"是我们的消费者。中间的盒子是一个队列 - 即 RabbitMQ 为消费者保留的一个消息缓存。

我们的总体设计看起来像这样：

![生产者、队列、消费者模型](https://kevin-blog-pic.oss-cn-shenzhen.aliyuncs.com/bp3u7.webp!origin)

生产者发送消息到名为"hello"的队列中。消费者则从这个队列中接收消息。

> ### RabbitMQ 知识库
>
> RabbitMQ 使用多种协议。这篇教程使用的是 AMQP 0-9-1，这是一个开放的、通用的消息传递协议。RabbitMQ 有许多用不同语言实现的客户端。在本教程系列中，我们将要使用的是 Pika 1.0.0，这是一个由RabbitMQ团队推荐的 Python 客户端。你可以使用 `pip` 包管理工具来安装它：
>
```bash
python -m pip install pika --upgrade
```

现在我们的 Pika 已经安装完成了，我们可以开始写一些代码了。

## Sending (发送)

![生产者发送模型](https://kevin-blog-pic.oss-cn-shenzhen.aliyuncs.com/dvm86.webp!origin)

我们的第一个程序 `send.py` 将会发送单条消息到队列中。我们要做的第一件事情就是与 RabbitMQ 服务器建立一条连接。

```python
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
```

我们现在已经连接上了本地机器的协商器，因此这里使用的是 `localhost` 。如果我们想要连接到另一台不同的机器上的协商器，我们只需要在这里指定它的域名或者 IP 地址。

接着，在发送消息之前，我们需要确保用于接收消息的队列确切存在。如果我们向一个不存在的位置发送了一条消息，RabbitMQ 会直接丢弃这条消息。让我们创建一个名为 **hello** 的队列用于发送消息：

```python
channel.queue_declare(queue='hello')
```

至此，我们已经准备好发送消息了。我们的第一条消息只包含一个字符串 **Hello World!** 并且我们想把它发送给我们的 **hello** 队列。

在 RabbitMQ 中，消息不能被直接发送给队列，它需要经过一个 **exchange (交换器)** 。但我们现在不要注意这些细节 - 你可以在[本教程系列的第三部分](https://www.rabbitmq.com/tutorials/tutorial-three-python.html)获取更多关于 **exchange (交换器)** 的内容。现在我们所需要知道的只是如何使用由空字符串标识的默认交换器。这个交换器很特殊 - 它允许我们确切地指定消息要被发送到哪个队列中去。目标队列的名字需要在 `routing_key` 这个参数中指定。

```python
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body='Hello World!')
print(" [x] Sent 'Hello World!'")
```

在退出程序之前，我们需要确保网络缓冲区被刷新并且我们的消息被确切地传递给了 RabbitMQ 。我们可以通过温柔地关闭连接来实现这一点。

```python
connection.close()
```

> ### 发送失败了！
>
> 如果这是你第一次使用 RabbitMQ ，并且没有看到那条 "Sent" 的输出消息，那么你可能会抓耳挠腮并好奇是哪里出了问题。有可能是协商器启动时已经没有足够可用的磁盘空间了(默认情况下它至少需要 200 MB的空闲空间)，因此它拒绝了接收消息。检查一下协商器的日志文件以确认并在必要情况下降低这个限制。[有关配置文件的文档](https://www.rabbitmq.com/configure.html#config-items)将会告诉你如何设置 `disk_free_limit` 。

## Receiving (接收)

![消费者接收模型](https://kevin-blog-pic.oss-cn-shenzhen.aliyuncs.com/w5kmn.webp!origin)

我们的第二个程序 `receive.py` 将要从队列中接收消息并将它们打印在屏幕上。

再一次的，我们首先要连接到 RabbitMQ 服务器。有关连接到 Rabbit 的代码跟前面是一样的。

下一步，像前面一样，需要确保目标队列确切存在。使用 `queue_declare` 来创建一个队列是幂等的 - 我们想运行这条命令多少次就可以运行多少次，因为只会创建一个队列。

```python
channel.queue_declare(queue='hello')
```

你可能会质问，为什么我们需要再次声明这个队列？我们已经在前面的代码中声明过了。如果我们能够确保目标队列确切存在，我们当然大可不必如此。比如我们假设 `send.py` 程序已经在前面运行过了。但是目前我们还不能确定需要先运行哪个程序。在这种情况下，在两个程序中重复地声明目标队列会是一个好的习惯。

> ### 列举出队列
>
> 你可能会想要看 RabbitMQ 中有哪些队列以及在队列中有多少消息。你可以使用 `rabbitmqctl` 工具(以被授权用户身份)来执行这个操作：
>
```bash
sudo rabbitmqctl list_queues
```
>
> 在 Windows 平台上则省略 sudo ：
>
```bash
rabbitmqctl list_queues
```

从队列中接收消息相比之下会更复杂一些。它的工作原理是为队列订阅一个 `callback (回调)` 函数。无论何时我们收到消息，这个 `callback (回调)` 函数都会被 Pika 库调用。在这个例子中，这个函数的作用是将消息中的内容打印到屏幕上。

```python
def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)
```

接着，我们需要告诉 RabbitMQ 这个特定的回调函数应该从我们的 hello 队列中接收消息：

```python
channel.basic_consume(queue='hello',
                      auto_ack=True,
                      on_message_callback=callback)
```

为了使这条命令能正常工作，我们需要确保要进行订阅操作的队列确切存在。幸运的是，对此我们非常自信 - 我们在上面的部分已经使用 `queue_declare` 创建了一个队列。

`auto_ack` 参数将在[后面部分](https://www.rabbitmq.com/tutorials/tutorial-two-python.html)进行讨论。

最终，我们进入了一个永无止境的循环，等待数据并在必要时运行回调函数。

```python
print(' [*] Waiting for messages. To exit press CTRL+C')
channel.start_consuming()
```

## 代码汇总

`send.py` ([源文件](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/send.py))

```python
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='hello')

channel.basic_publish(exchange='', routing_key='hello', body='Hello World!')
print(" [x] Sent 'Hello World!'")
connection.close()
```

`receive.py` ([源文件](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/receive.py))

```python
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='hello')


def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)


channel.basic_consume(
    queue='hello', on_message_callback=callback, auto_ack=True)

print(' [*] Waiting for messages. To exit press CTRL+C')
channel.start_consuming()
```

现在，我们可以在终端中尝试运行我们的程序。首先，让我们启动一个消费者程序，它将持续地运行以等待"货物"。

```bash
python receive.py
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'Hello World!'
```

现在让我们启动生产者程序。它会在每次运行完成后便停止。

```bash
python send.py
# => [x] Sent 'Hello World!'
```

欢呼吧！我们已经能够通过 RabbitMQ 发送我们的第一条消息了。你可能已经注意到了，`receive.py` 程序并没有退出。它会时刻准备着接收其他消息，直到使用 CTRL-C 组合键来停止。

在新的终端中尝试再次运行 `send.py` 程序。

**我们已经学习了如何向一个已经命名的队列发送消息并从中接收消息。是时候前往[第 2 部分](https://www.rabbitmq.com/tutorials/tutorial-two-python.html)去学习如何建立一个简单的 work queue (工作队列)了。**

## 生产[非]适用性免责声明

请记住，本文或其他教程都只是教程。教程一次展示一个新的概念，并且可能会故意过度简化某些东西而忽略了其它的事物。例如，为了简洁起见，有关连接管理、错误处理、连接恢复、并发和度量收集的内容都在很大程度上被忽略了。这样简化过的代码不应该被当作是已经完成的产品。

在开发您自己的应用程序之前，请先查看[文档](https://www.rabbitmq.com/documentation.html)的剩余部分。我们特别推荐以下部分：[Publisher Confirms and Consumer Acknowledgements](https://www.rabbitmq.com/confirms.html)， [Production Checklist](https://www.rabbitmq.com/production-checklist.html) 以及 [Monitoring](https://www.rabbitmq.com/monitoring.html) 。

## 获得帮助并提供反馈

如果您对本教程或有关 RabbitMQ 的其他内容感到疑问，请立即在 [RabbitMQ mailing list](https://groups.google.com/forum/#!forum/rabbitmq-users) 上提问。

## 帮助我们完善 3 版本以下的文档

如果你想对本网站做出改进，可以在 [Github](https://github.com/rabbitmq/rabbitmq-website) 上找到它的源码。只需要 fork 源仓库并提交一条 pull 请求。非常感谢！

