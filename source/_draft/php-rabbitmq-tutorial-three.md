title: PHP RabbitMQ 教程（三） - 发布/订阅
categories: php
date: 2016-03-16 21:46:18
tags:  [php,rabbitmq]

---

我们在上一节创建了一个工作队列，并假定队列对应的任务传送给了某个客户端。在这一章节我们会做一些完全不一样的东西--我们会发送一条消息到多个消费者，也称之为“发布/订阅”模式。

为了说明这个模式，我们会创建一个简单的日志系统，它由两个程序组成--第一个是发送日志信息，第二个是接收日志并打印。

In our logging system every running copy of the receiver program will get the messages. 这样就可以运行一个接收端就把日志保存到硬盘里，同时运行另一个接收端去实时显示日志到屏幕。

本质上，日志内容是广播给所有的接收端的。

### 交换
在之前的章节中我们从一个队列里发送和接收消息，现在该把完整的RabbitMQ消息模型介绍给大家了。

让我们快速的回看一遍在之前的章节中的内容：

生产者是一个用来发送消息的程序

队列是一个存储消息的缓冲区

消费者是一个接收消息的程序

RabbitMQ消息模型的核心思想是，生产者永远不会直接发送给任何消息队列，实际上，生产者一般情况下甚至不知道消息应该发送给哪个队列。

生产者只能发送消息到交换器中，交换器非常简单。一方面从生产者接收消息，另一方面把消息推送到队列中。交换器必须知道如何处理接收到的消息，是推送到某个队列？推送到多个队列？还是丢弃这条消息。这个规则通过交换器类型(exchange type)来指定。

@图1

这里是交换器的几个类型：direct,topic,headers,fanout。这里我们主要关注最后一个--fanout，我们创建一个类型为 fanout 的交换器，命名为 logs。

```
$channel->exchange_declare('logs','fanout',false,false,false);
```

fanout交换器非常简单，你可以从名称中猜出它的功能，它把所有接收到的消息广播给所有它知道的队列，这也正是我们的日志程序需要的功能。

现在，可以发布消息到这个队列。

```
$channel->exchange_declare('logs','fanout',false,false,false);
$channel->basic_publish($msg,'logs');
```

### 临时队列


