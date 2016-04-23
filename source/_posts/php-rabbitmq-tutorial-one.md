title: PHP RabbitMQ 教程（一） - 介绍
categories: php
date: 2016-02-28 11:10:03
tags:  [php,rabbitmq]

---

## 准备工作
### 先决条件

本教程先决条件是RabbitMQ已经安装并正在以5672端口运行在 localhost，如果你使用了不同的域，端口，用户，密码，连接配置需要适当改变。

### 获得帮助

如果在本教程中遇到问题，可以通过邮件列表进行联系。

### 介绍

RabbitMQ是一个消息代理，它的本质是，从producers（生产者）接收消息，然后发送给consumers（消费者），在这个过程中，可以根据自己的配置规则使用路由，缓冲区，保存消息。

通常的，RabbitMQ，信息传送（messaging），使用一些专业术语。（RabbitMQ, and messaging in general, uses some jargon.）

	>生产（Producing）仅仅意味着发送，发送信息的程序叫做生产者（producers），以下图表示：
	
![](/images/rabbitmq/producer.png)
	
	>队列就是一个信箱的名字，存在于RabbitMQ内部，虽然消息在RabbitMQ和你的应用之间传输，但是只能存在于队列里，队列没有大小限制，它可以存储尽可能多的消息，本质上它是一个无限大的缓冲区，多个producers（生产者）可以通过一个队列发送消息，多个consumers（消费者）也可以尝试从一个队列接收消息，队列以下图表示，队列的名字在图的上边：

![](/images/rabbitmq/queue.png)	

	>consumers（消费者）的意思与接收相似，消费者主要是等待接收消息的程序，以下图表示：
	
![](/images/rabbitmq/consumer.png)	

需要注意的是，生产者，消费者，和代理，不需要一定在一台机器上，事实上在大多数情况下他们确实不在一台机器上。

### “Hello World”

（使用[php-amqplib](https://github.com/php-amqplib/php-amqplib)库）

在这一部分，我们使用PHP写两段程序，一个生产者发送一条消息，一个消费者接收消息并打印出来。我们会忽略一些php-amqplib API的细节，从简单的事情开始学习，这是一段内容为“Hello World”的消息。

在下边的示意图中，“P”是生产者，“C”是消费者，中间的盒子是队列 — 一个RabbitMQ代表消费者的消息缓冲区。

![](/images/rabbitmq/python-one.png)

#### php-amqplib库

RabbitMQ支持很多协议，本教程包含AMQP 0-9-1，一个开放，通用信息协议，RabbitMQ支持Python、Ruby、.NET、Java、JMS、C、PHP、ActionScript、XMPP、STOMP等多种语言（详见[这里](http://www.rabbitmq.com/devtools.html)），在本教程中我们使用php-amqplib，使用[Composer](https://getcomposer.org/doc/00-intro.md) 管理依赖。

添加一个composer.json文件到你的项目目录。

```
{
    "require": {
        "php-amqplib/php-amqplib": "2.5.*"
    }
}
```

如果你已经安装了 Composer ，可以运行如下的代码：

```
composer.phar install
```

这是一个Windows系统下的Composer安装文件。

现在我们已经安装了php-amqplib，可以写程序了。

### 发送

![](/images/rabbitmq/sending.png)

新建一个send.php作为发送端，receive.php作为接收端，发送端会连接RabbitMQ，发送一条信息，然后退出。

在send.php中，需要引用php-amqplib库，和使用其中的一些必要的类。

```
require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

```

接下来，建立到RabbitMQ服务器的连接：

```
$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

```

这里我们使用socket进行连接，处理协议和鉴定，这样就已经连接到了本机的代理，如果想要连接不同的主机，只要更改localhost为该主机的名称或IP地址即可。

下一步，建立频道，大部分API的工作都在这完成。

要想发送信息，需要声明一个队列，之后可以向这个队列里发布消息。

```
$channel->queue_declare('hello', false, false, false, false);

$msg = new AMQPMessage('Hello World!');
$channel->basic_publish($msg, '', 'hello');

echo " [x] Sent 'Hello World!'\n";

```

声明队列是幂等的 — 它仅在不存在的时候才会被创建，如果存在也不会受影响。消息内容是一个字节数组（byte array），所以可以发送任何内容。

最后，关闭频道和连接。

```
$channel->close();
$connection->close();
```

[这是send.php类的完整内容。](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/send.php)

### 发送失败

如果这是第一次使用RabbitMQ，并且没有看到“Sent”信息（即“ [x] Sent 'Hello World!”），也许你抓耳挠腮的想知道为什么出错了，也许是代理没有足够的硬盘空间（默认情况下需要至少1G的空间）导致拒绝接收信息。检查日志文件，有必要的花调低限值。[这个配置文件](http://www.rabbitmq.com/configure.html#config-items)文档将会展示给你如何设置disk_free_limit。

### 接收

收件人，与发送者只发送一条消息不同，接收者会一直运行以监听信息并输出。

![](/images/rabbitmq/receiving.png)

receive.php中与send.php中的 include和use 部分的代码一样。

```
require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;

```

设置连接与send.php一样，打开连接和频道，命名一个队列，需要注意的是，队列名需要与send.php所发布的队列的名字一致。

```
$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->queue_declare('hello', false, false, false, false);

echo ' [*] Waiting for messages. To exit press CTRL+C', "\n";

```

注意，我们在此声明了一个队列，因为有可能会在send程序开启前先开启receive程序，我们想要确保在试着接收消息之前队列就已经存在了。

下一步，告诉服务器去从队列传送消息，我们会定义一个用于从服务器接收消息的函数，记住，消息会异步的从服务器发送到客户端。

```
$callback = function($msg) {
  echo " [x] Received ", $msg->body, "\n";
};

$channel->basic_consume('hello', '', false, true, false, false, $callback);

while(count($channel->callbacks)) {
    $channel->wait();
}
```
此处使用while方法，当收到消息时，会把收到的消息传入到$callback方法里。

[这是receive.php类的全部内容。](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/receive.php)

现在我们可以运行两段脚本了，在命令行里，执行sender程序。

```
php send.php

```

然后，执行receiver程序

```
php receive.php

```

receiver程序会把通过sender程序发送的内容打印出来，receiver程序会一直运行，监听新消息（使用ctrl+c停止），所以试着运行sender程序从另一个命令行。

如果想查看队列，可以运行rabbitmqctl list_queues。

Hello World！

查看第二部分，建立一个简单的队列。