title: PHP RabbitMQ 教程（五） - 主题
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

### Topic 交换器

发送到 topic 交换器的消息不能随意设置 routing_key - 它必须是一个单词列表，以'.'分隔。单词可以是任何内容，但是通常会具体说明消息的功能。一些有效的routing key示例：样："stock.usd.nyse"，"nyse.vmw"，"quick.orange.rabbit"。routing key 可以是任何长度的你喜欢的单词，最大255个字节。

binding key 也必须是同样的格式，topic交换器的逻辑和direct交换器类似 - 带有特定routing key的消息会被派发到所有绑定了binding key的队列，然而对于binding key依然有两个重要的特殊情况：

	*可以代替一个单词

	#可以代替0个或多个单词

下图比较好的解释了这个情况：

![](/images/rabbitmq/python-five.png)	

在这个例子中，我们准备全部发送描述动物的消息。这些消息带有由三个单词(两个点号分隔)组成的routing key，其中第一个单词表示速度，第二个表示颜色，第三个表示种类："<speed>.<colour>.<species>"。

我们创建三个绑定：Q1的binding key为"\*.orange"，Q2的binding key为"\*.\*.rabbit"和"lazy.\#"。

这些绑定可以概括为：

	Q1 关注所有orange的动物
	
	Q2 想知道所有关于兔子（rabbits）和懒惰动物（lazy animals）的消息。

routing key为"quick.orange.rabbit"的消息会被发送到两个队列，"lazy.orange.elephant"也会被发送到这两个队列。而"quick.orange.fox"则只会发送到第一个队列，"lazy.brown.fox"会被只发送到第二个队列。"lazy.pink.rabbit"会只被发送到第二个队列一次，即使它匹配两个绑定。"quick.brown.fox"不匹配任何绑定，所以会被丢弃。

如果打破规则，发送一条带有一个或四个单词，如"orange"或"quick.orange.male.rabbit"会怎么样？好吧，消息会丢失，因为它不匹配任何一个绑定。

但是，"lazy.orange.male.rabbit"这种消息，即使它有4个单词，依然会匹配最后一个绑定，然后被发送到第二个队列。

#### topic 交换器

```
topic 交换器非常强大，可以表现得跟其他交换器一样。

当一个队列的binding key为"#"时，它会接收所有消息，忽略routing key，像fanout交换器一样。

当绑定中不存在"*"和"#"时，topic交换器会表现的跟direct交换器一样。

```

### 整合

我们准备在日志系统中使用topic交换器。假定日志的routing key由两个单词："<facility>.<severity>"组成。

代码与上一节的几乎一致。

emit_log_topic.php：

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->exchange_declare('topic_logs', 'topic', false, false, false);

$routing_key = isset($argv[1]) && !empty($argv[1]) ? $argv[1] : 'anonymous.info';
$data = implode(' ', array_slice($agrv, 2));
if(empty($data)) $data = "Hello Wrold!";

$msg = new AMQPMessage($data);

$channel->basic_publish($msg, 'topic_logs', $routing_key);

echo " [x] Sent ",$routing_key,':',$data," \n";

$channel->close();
$connection->close();

?>
```

receive_logs_topic.php：

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->exchange_declare('topic_logs', 'topic', false, false, false);

list($queue_name, ,) = $channel->queue_declare("", false, false, true, false);

$binding_keys = array_slice($argv, 1);
if( empty($binding_keys) ){
  file_put_contents('php://stderr', "Usage: $argv[0] [binding_key]\n");
  exit(1);
}

foreach($binding_keys as $binding_key){
  $channel->queue_bind($queue_name, 'topic_logs', $binding_key);
}

echo ' [*] Waiting for logs. To exit press CTRL+C', "\n";

$callback = function($msg){
  echo ' [*] ',$msg->delivery_info['routing_key'], ':', $msg->body, "\n";
}

$channel->basic_consume($queue_name, '', false, true, false, false, $callback);

while(count($channel->callbacks)){
  $channel->wait();
}

$channel->close();
$connection->close();

?>
```

接收所有日志：

```
php receive_logs_topic.php '#'
```

接收所有来自"kern"的日志：

```
php receive_logs_topic.php "kern.*"
```

只接收"致命(critical)"日志：

```shell
php receive_logs_topic.php "*.critical"
```

创建多个绑定：

```shell
php receive_logs_topic.php "kern.*" "*.critical"
```

发布routing key为"kern.critical"的日志就输入：

```shell
php emit_log_topic.php "kern.critical" "A critical kernel error"
```

注意此代码并没有做路由或捆绑的例子，也许你想试一下两个以上的routing key参数。

一些问题：

​	["*"会匹配routing key为空的消息吗？](http://www.rabbitmq.com/tutorials/tutorial-five-php.html#teaser_answer_1)

​	["#.*"会匹配内容为".."的消息吗？会匹配一个单词的消息吗？](http://www.rabbitmq.com/tutorials/tutorial-five-php.html#teaser_answer_2)

​	["a.*.#"和"a.#"的区别是什么？](http://www.rabbitmq.com/tutorials/tutorial-five-php.html#teaser_answer_3)

[emit_log_topic.php完整代码](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/emit_log_topic.php) [receive_logs_topic.php完整代码](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/receive_logs_topic.php)

下一步，在第六节中学习像远程过程调用一样完成消息往返。