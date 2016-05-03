title: PHP RabbitMQ 教程（六） - 远程调用
categories: php
date: 2016-05-03 14:35:20
tags:  [php,rabbitmq]

---

### 远程调用

在[第二节](/2016/04/17/php-rabbitmq-tutorial-two.html)中，我们学习了如何使用工作队列在多个worker中分发耗时任务。

但是如果我们需要在远程运行一个函数并等待返回结果怎么办？这是两码事，这个模式通常被称为远程过程调用（Remote Procedure Call，RPC）。

