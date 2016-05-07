title: PHP RabbitMQ 教程（三） - 发布/订阅
categories: php
date: 2016-04-23 19:58:17
tags:  [php,rabbitmq]

---

### 发布/订阅

我们在[上一节](/2016/04/17/php-rabbitmq-tutorial-two.html)创建了一个工作队列，并假定队列对应的任务传送给了某个客户端。在这一章节我们会做一些完全不一样的东西--我们会发送一条消息到多个消费者，也称之为“发布/订阅”模式。

为了说明这个模式，我们会创建一个简单的日志系统（logging system，以下简称日志系统），它由两个程序组成--第一个是发送日志信息，第二个是接收日志并打印。

日志系统的每一个运行的接收端程序都会接收信息，这样就可以运行一个接收端就把日志保存到硬盘里，同时运行另一个接收端去实时显示日志到屏幕。

本质上，日志内容是广播给所有的接收端的。

### 交换器

在之前的章节中我们从一个队列里发送和接收消息，现在该把完整的RabbitMQ消息模型介绍给大家了。

让我们快速的回看一遍在之前的章节中的内容：

	>生产者是一个用来发送消息的程序

	>队列是一个存储消息的缓冲区

	>消费者是一个接收消息的程序

RabbitMQ消息模型的核心思想是，生产者永远不会直接发送给任何消息队列，实际上，生产者一般情况下甚至不知道消息应该发送给哪个队列。

生产者只能发送消息到交换器中，交换器非常简单。一方面从生产者接收消息，另一方面把消息推送到队列中。交换器必须知道如何处理接收到的消息，是推送到某个队列？推送到多个队列？还是丢弃这条消息。这个规则通过交换器类型(exchange type)来指定。

![](/images/rabbitmq/exchanges.png)

这里是交换器的几个类型：direct,topic,headers,fanout。这里我们主要关注最后一个--fanout，创建一个类型为 fanout 的交换器，命名为 logs。

```
$channel->exchange_declare('logs','fanout',false,false,false);
```

fanout交换器非常简单，你可以从名称中猜出它的功能，它把所有接收到的消息广播给所有它知道的队列，这也正是我们的日志系统需要的功能。


#### 列出交换器

可以使用rabbitmqctl 命令列出服务器上的所有交换器：

```
sudo rabbitmqctl list_exchanges

Listing exchanges ...
        direct
amq.direct      direct
amq.fanout      fanout
amq.headers     headers
amq.match       headers
amq.rabbitmq.log        topic
amq.rabbitmq.trace      topic
amq.topic       topic
logs    fanout
...done.
```

结果中有一些amq.*和一些未命名的交换器，这是一些默认创建的交换器，它们不太可能是现在需要用到的。

#### 未命名交换器

在之前的章节中我们对交换器一无所知，直到可以发送消息给队列。大概是因为我们当时正在使用一个以空字符串“”定义的默认的交换器。

回想一下之前怎么发布消息：

```
$channel->basic_publish($msg,'','hello');
```

这里就是使用默认或者说未命名的交换器：消息被routing_key的值
Here we use the default or nameless exchange: messages are routed to the queue with the name specified by routing_key, if it exists. The routing key is the second argument to basic_publish

现在，可以发布消息到这个队列。

```
$channel->exchange_declare('logs','fanout',false,false,false);
$channel->basic_publish($msg,'logs');
```

### 临时队列

也许你还记得在之前我们使用了一个指定的队列（还记得 hello 队列 和 task_queue 队列吗？）。可以命名一个队列是至关重要的--我们需要指定一个worker到同一个队列。当想让生产者和消费者使用同一个队列时给队列命名是非常重要的。

但是在我们的日志系统中情况不同了，我们想要接收所有的消息，不仅仅是其中的一部分，我们关心的是最新的消息而不是旧的，因此需要做两件事。

首先，当连接到RabbitMQ时，需要一个空的队列，可以手动创建一个名字随机的队列，或者，更好的办法是，让服务器为我们随机选一个队列名字。

其次，一旦与消费者失去连接，队列需要自动删除。

在[php-amqplib](https://github.com/php-amqplib/php-amqplib)中，当我们创建了一个名字为空的队列时，实际上是创建了一个被生成了名字的非持久化的队列。

```
list($queue_name, ,) = $channel->queue_declare("");
```

方法执行后，$queue_name变量包含了一个RabbitMQ生成的字符串。比如也许是这样的：amq.gen-JzTY20BRgKO-HjmUJj0wLg。

当连接被关闭的时候，队列也会被删掉，因为队列是独有的。

### 绑定(Bindlings)

![](/images/rabbitmq/bindings.png)

我们已经创建了一个fanout类型的交换器和一个队列。现在需要让交换器发送消息给队列。交换器和队列之间的关系称之为绑定(binding)

```
$channel->queue_bind($queue_name,'logs');
```

现在开始，logs 交换器会把消息附加到队列中。

#### 列出绑定（Listing bindings）

可以使用 rabbitmqctl list_bindings列出所有存在的正在使用的绑定。

### 整合

![](/images/rabbitmq/python-three-overall.png)

发送日志消息的生产者，与之前的代码看起来没什么不同，最重要的变化是现在想要发送消息到我们的 logs 交换器中，需要在发送时提供一个routing_key，但是在 fanout类型的交换器中这个值是可以忽略的。下边是emit_log.php的代码。

```php
<?php

require_once __DIR__ .'/verdor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

$connection = new AMQPStreamConnection('localhost',5672,'guest','guest');
$channel = $channel->channel();

$channel->exchange_declare('logs','fanout',false,false,false);

$data = implode(' ',array_slice($argv,1));

if(empty($data)) $data = "info:Hello World";
$msg = new AMQPMessage($data);

$channel->basic_publish($msg,'logs');

echo "[x]Sent ",$data,"\n";

$channel->close();
$connection->close();
?>
```

([emit_log.php](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/emit_log.php))

如你所见，建立连接后声明了交换器，这一步是必须的，因为发送消息到一个不存在的交换器是被禁止的。

如果还没有队列绑定到交换器，信息会丢失，但是这对于我们是可以的，如果没有消费者监听，我们可以安全的丢弃消息。

receive_logs.php：

```php
<?php

require_once __DIR__ .'/vendor/autoload.php';
use PhpAmqpLib\Connection\QMAPStreamConnection;

$connection = new AMQPStreamConnection('localhost',5672,'guest','guest');
$channel = $connection->channel();

$channel->queue_bind($queue_name,'logs');

echo '[*] Waiting for logs. To exit press CTRL+C',"\n";

$callback = function($msg){
	echo '[x]',$msg->body,"\n";
};

$channel->basic_consume($queue_name,'',false,true,false,false,$callback);

whild(count($channel->callbacks)){
	$channel->wait();
}

$channel->close();
$connection->close();
?>
```
([receive_logs.php](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/receive_logs.php))

如果想保存日志到文件中，可以在命令中输入

```
php receive_logs.php > logs_from_rabbit.log
```

如果想在屏幕上查看日志，新打开一个终端并运行：

```
php receive_logs.php
```

发送日志：

```
php emit_log.php
```

使用 rabbitmqctl list_bindings 可以确认代码确实创建了绑定和队列，当两个receive_logs.php在运行的时候会看到类似这样的：

```
sudo rabbitmqctl list_bindings
Listing bindings ...
logs    exchange        amq.gen-JzTY20BRgKO-HjmUJj0wLg  queue           []
logs    exchange        amq.gen-vso0PVvyiRIL2WoV3i48Yg  queue           []
...done.

```

对于结果的解释很简单，logs交换器中的数据发送到两个服务器指定的队列，而这正是我们要实现的。

想要弄明白怎样去监听部分消息，转到第四部分。

原文地址：[Publish/Subscribe](https://www.rabbitmq.com/tutorials/tutorial-three-php.html)