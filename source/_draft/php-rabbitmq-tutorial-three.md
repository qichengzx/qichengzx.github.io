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
