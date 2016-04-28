title: PHP RabbitMQ 教程（五） - 话题
categories: php
date: 2016-04-27 18:27:03
tags:  [php,rabbitmq]
---

### 主题

([使用php-amqplib](https://github.com/php-amqplib/php-amqplib))

在[上一节](/2016/04/24/php-rabbitmq-tutorial-four.html)我们改善了日志系统(logging system，以下简称日志系统)，为了替代fanout类型的交换器，我们使用了一个direct类型的交换器，带来的好处是可以有选择的接收日志。

虽然使用direct交换器改善了系统，但是仍然有局限性 - 它不能根据多个条件进行路由。

在日志系统中，我们也许不仅仅想订阅严重等级的日志，也想订阅基于消息发布源的内容。也许你已经知道这个概念来自于UNIX的[syslog](http://en.wikipedia.org/wiki/Syslog)工具，基于严重性(info/warn/crit...)和设备路由日志(auth/cron/kern...)的工具。

这可以提高灵活性 - 我们也许只想要监听来自'cron'的关键错误而不是来自'kern'的全部日志。

为了在我们的日志系统上实现这个功能，需要学习一个更复杂的 topic 交换器。

### Topic交换器

发送到 topic 交换器的消息不能随意设置 routing_key - 它必须是一个单词列表，以'.'分隔。单词可以是任何内容，但是通常会具体说明消息的功能。一些有效的routing key示例：样："stock.usd.nyse"，"nyse.vmw"，"quick.orange.rabbit"。routing key 可以是任何长度的你喜欢的单词，最大255个字节。

binding key 也必须是同样的格式，topic交换器背后的逻辑和direct交换器类似 - 带有特定routing key的消息会被派发到所有绑定了binding key的队列，然而对于binding key依然有两个重要的特殊情况：

​	*可以代替一个单词

​	\#可以代替0个或多个单词

下图比较好的解释了这个情况：

![](/rabbitmq/python-five.png)	

在这个例子中，我们准备全部发送描述动物的消息。这些消息带有由三个单词(两个点号分隔)组成的routing key，其中第一个单词表示速度，第二个表示颜色，第三个表示种类："<speed>.<colour>.<species>"。

我们创建三个绑定：Q1的binding key为"\*.orange"，Q2的binding key为"\*.\*.rabbit"和"lazy.\#"。

这些绑定可以概括为：

​	Q1 关注所有orange的动物

​	Q2 想知道所有关于兔子（rabbits）和懒惰动物（lazy animals）的消息。









