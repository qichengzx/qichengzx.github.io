title: PHP RabbitMQ 教程（四） - 路由
categories: php
date: 2016-04-24 14:16:41
tags:  [php,rabbitmq]()

——

### 路由
（使用[php-amqplib][2]）
在上一节中，我们创建了一个简单的日志系统（logging system）。我们已经可以广播日志消息到多个接收者了。

在本节中，我们要给它增加一个功能-使它能够只订阅消息的一个子集。比如，只把严重的错误信息写入到日志文件（存储到磁盘）中，但同时仍然会把所有日志信息输出到控制台中。

### 绑定

在上一节中我们已经创建了绑定（bindings），代码如下：

\`\`\`
\`
$channel-\>queue\_bind($queue\_name,’logs’);

\`\`\`
\`
绑定（bindings）是指交换器（exchange）和队列（queue）的关系。可以简单的理解为：这个队列对这个交换器中的消息感兴趣。

绑定的时候可以带一个额外的 routing\_key 参数。为了避免与$channel::basic\_publish的参数混淆，我们把它叫做 binding\_key，所以我们这样使用key创建一个绑定：

\`\`\`
\`$binding\_key = ‘black’;
$channel-\>queue\_bind($queue\_name,$exchange\_name,$binding\_key);
\`\`\`
\`
binding key的意义取决于交换器的类型。我们之前使用过的fanout类型的交换器，会忽略这个值。

### Direct 交换器

之前创建的日志系统分发所有消息到所有的消费者。我们打算扩展一下，使它可以过滤严重的消息。比如，我们只想在接收到严重错误的时候才写入到磁盘中，不在警告或普通的消息上浪费磁盘空间。

我们使用的是没有太多扩展性的fanout交换器，它仅能够简单的广播消息。

我们将要使用一个direct交换器代替。路由算法很简单-只有binding key完全匹配routing key的消息会进入队列。

为了说明，考虑如下的场景：

![][image-1]

在这个场景中，我们可以看到direct类型的交换器X有两个队列，第一个队列使用orange作为binding key，第二个队列有两个绑定，一个是black另一个是green。

在这个场景中，当routing key为orange的消息发送到交换器，将会被路由到队列Q1。routing key为black或green的消息将会发送到Q2。其他的消息则会被丢弃。

### 多个绑定
![][image-1]

使用相同的binding key绑定多个队列是合法的。我们会使用direct交换器代替fanout交换器来发布消息。

[2]:	https://github.com/php-amqplib/php-amqplib

[image-1]:	/images/rabbitmq/direct-exchange-multiple.png