title: PHP RabbitMQ 教程（二） - 工作队列
categories: php
date: 2016-04-17 14:39:30
tags:  [php,rabbitmq]

---

### 工作队列

##### （使用[php-amqplib](https://github.com/php-amqplib/php-amqplib)库）

![](/images/rabbitmq/python-two.png)

在本教程[第一部分](/2016/02/28/php-rabbitmq-tutorial-one.html) 我们已经写完了从一个指定队列发送和接收消息的程序。在这一章节中，我们会创建一个工作队列（Work Queue）来分发耗时的任务给多个工作者（worker）。

工作队列（也被称为 任务队列-task queue）主要是避免立即执行资源密集型任务并且还要等待它执行完毕。相反，需要让任务稍后执行，我们把一个任务当做一条信息发送给队列，后台运行的工作者（worker）会取出任务并执行，当运行多个worker时任务会在它们之间共享。

这个概念在web应用中非常有用，可以在短暂的HTTP请求期间处理一些复杂的任务。

### 准备工作

在前面的部分我们发送了一条内容为“Hello World”的信息，现在我们会发送一些字符串，把这些字符串当做复杂的任务，我们并没有一个实际的任务，像是图片缩放，或者转换PDF文件，所以我们使用sleep方法来假设任务很繁忙。我们会在字符串中加入一些“.”来表示复杂复杂程度；每一个“.”表示需要耗时1秒，比如，“Hello ...”代表需要耗时3秒。

我们从上一节的基础上稍微改动了一下send.php，来允许消息可以从命令行发送，这个程序会发送任务到队列中，把它命名为new_task.php

```
$data = impllode(' ',array_slice($argv,1));
if(empty($data))$data = "Hello World";
$msg = new AMQPMessage($data,
	array('delivery_mode'=>2)#设置消息持久化，下边会讲到。
);
$channel->basic_publish($msg,'','task_queue');
echo "[x] Sent ",$data,"\n";
```

上一节的receive.php也需要一些改动：需要为消息中的每一个“.”模拟1秒的工作。它会从队列中取出消息并运行，把它命名为worker.php：

```
$callback = function($msg){
	echo "[x] Received ",$msg->body,"\n";
	sleep(substr_count($msg->body,'.'));
	echo "[x] Done","\n";
	$msg->delivery_info['channel]->basic_ack($msg->delivery_info['delivery_tag']);
};

$channel->basic_gos(null,1,null);
$channel->basic_consume('task_queue','',false,false,false,false,$callback);
```

注意我们伪造的任务需要花费时间（即发送的字符串中要有一些"."）

然后运行：

```
php new_task.php "A very hard task which takes two seconds.."
php wordker.php
```

### 轮询分发

使用工作队列的一个好处就是它能够并行的处理队列。如果有太多工作需要处理，只需要添加新的worker就可以了。

首先，我们试着同时运行两个worker.php，它们都会从队列接收到消息，但是到底是不是这样呢？我们看一下。

此时需要打开3个终端，其中两个运行worker.php，这两个就是我们的消费者 - C1和C2。

shell1

```
php worker.php
[*] Waiting for messages. To exit press CTRL+C
```
shell2

```
php worker.php
[*] Waiting for messages. To exit press CTRL+C
```

在第三个终端中我们会发送新的任务，消费者程序开始运行后就可以发送一些消息了。

shell3

```
php new_task.php First message.
php new_task.php Second message..
php new_task.php Third message...
php new_task.php Fourth message....
php new_task.php Fifth message.....
```

我们看一下发送给worker的是什么:

shell1

```
php worker.php
[*] Waiting for messages.To exit press CTRL+C
[x]Received 'First message.'
[x]Received 'Third message...'
[x]Received 'Fifth message.....'
```
shell2

```
php worker.php
[*] Waiting for messages.To exit press CTRL+C
[x]Received 'Second message.'
[x]Received 'Fourth message...'
```

RabbitMQ会默认按顺序把消息发送给下一个消费者，平均每个消费者都会得到一样多数量的消息，这种分发消息的方式叫做轮询。试着添加三个或更多个worker来运行。

### 消息响应

执行一个任务会消耗一定的时间，也许你想知道如果一个消费者在执行一个耗时较长的任务时但是在执行一部分的时候挂掉会发生什么。在我们当前的代码中，一旦RabbitMQ把消息分发给消费者便会立即从内存中移除。这种情况下，如果停止一个worker，它正在处理的消息就会丢失。同时其他所有发送给这个worker的还没有处理的消息也会丢失。

但是我们不想丢失任何任务，如果一个worker挂掉，需要把任务发送到另一个worker。

为了确保消息永不丢失，RabbitMQ支持消息响应（message acknowledgements），消费者会发送一个响应告诉RabbitMQ已经收到了某条消息，并且已经处理，这样RabbitMQ就可以删掉它了。

如果一个消费者程序在未发送响应之前挂掉了（频道关闭，链接关闭，或者TCP连接丢失），RabbitMQ会认为消息没有完全处理然后会重新推送到队列中。如果此时有其他的消费者程序在运行，RabbitMQ会很快把消息发送给另一个消费者。这样就可以确保消息不会丢失，即使worker偶尔挂掉。

消息是没有超时的概念的，当worker断开连接的时候，RabbitMQ会重新发送消息，这样在处理一个耗时较长的消息任务时就不会出现问题了。

消息响应默认是关闭的。可以通过设置basic_consume的第四个参数为false(true表示不开启应答)，然后在处理完任务的时候从worker发送一个正确的响应内容。

```
$callback = function($msg){
	echo "[x] Received ",$msg->body,"\n";
	sleep(substr_count($msg->body,'.'));
	echo "[x] Done","\n";
	$msg->delivery_info['channel]->basic_ack($msg->delivery_info['delivery_tag]);
};
$channel->basic_consume('task_queue','',false,false,false,false,$callback);
```

这样我们就可以确保当你CTRL+C杀掉一个正在处理消息的worker的时候，消息并不会丢失。在这个worker挂掉之后，所有未响应的消息就会发送。

### 忘了响应

一个很容易犯的错误就是忘了basic_ack，后果很严重。消息会在程序退出后重新发送（可能看起来像是随机返还 原文：which may look like random redelivery），但是如果它不释放未响应的消息，RabbitMQ就会占用越来越多的内存。

为了排除这种错误可以使用rabbitmqctl来打印messages_unacknowledges字段：

```
sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged

Listing queues ...
hello    0       0
...done.
```


### 消息持久化

我们已经学习了确保即使消费者程序挂掉，任务也不会丢失。但是任务还是会在RabbitMQ服务停止的时候丢失。

当RabbitMQ退出或崩溃，它会丢失之前所有的队列和消息，除非你特意告诉它。所以我们必须把队列和消息设为持久化。

首先，为了队列不丢失，需要把它声明为<i>持久化（durable）</i>，所以修改queue_declare的第三个参数为true：

```
$channel->queue_declare('hello',false,true,false,false);
```

尽管这行代码本身是正确的，但是仍然不会正确运行。因为在之前已经定义过一个非持久化的 hello 队列。RabbitＭＱ不允许使用不同参数重新定义一个已经存在的队列，它会返回一个错误。但是可以用一个快捷的方法去解决，定义一个不同名字的队列，比如 task_queue：

```
$channel->queue_declare('task_queue',false,true,false,false);
```

需要把生产者和消费者程序都设置为 true。

这时候，我们就可以确保在RabbitMQ重启之后task_queue队列不会丢失。现在需要设置消息持久化了 - 通过设置AMQPMessage的属性数组中消息属性 delivery_mode = 2来达到。

```
$msg = new AMQPMessage($data,
		array('delivery_mode'=>2) //设置消息持久化
	);
```

### 关于消息持久化的说明

设置消息持久化并不能完全保证消息不会丢失。这只是告诉让RabbitMQ要把消息保存到硬盘，但是从RabbitMQ接收到消息到保存完成仍然还有一个短暂的间隔时间。因为RabbitMQ并不是每一条消息都会使用fsync(2)，可能只是保存到缓存中而不是真正的写到磁盘里。并不能保证消息真正的持久化，但是对于简单的工作队列已经足够了。如果你需要更健壮的持久化，可以使用[publisher confirms](https://www.rabbitmq.com/confirms.html)机制。



### 公平分发

也许你注意到它仍没有像我们想的那样去派发任务，比如在两个worker的情况下，处理奇数消息的比较繁忙，处理偶数消息的比较轻松，一个worker不断的忙碌而另一个几乎不需要工作，但是RabbitmQ并不知道这些，并且继续一如既往的派发消息。

这是因为RabbitMQ在消息进入队列的时候只管去派发，并不管消费者未做出响应的消息数。它只是把每第n条消息发送给第n个消费者。

我们可以使用basic_qos方法，并设置prefetch_count = 1。这样是告诉RabbitMQ在同一时刻不要发送超过1条消息给一个worker，或者说，不要发送新的消息给worker直到它已处理完上一条消息并作出了响应。这样，它就会把消息发送给下一个空闲的worker了。

![](/images/rabbitmq/prefetch-count.png)

```
$channel->basic_qos(null,1,null);
```

### 注意队列长度

如果所有的worker都处于忙碌状态，队列就会填满，你需要留意，添加更多的worker，或者使用其他的策略。


### 整合

最终，new_task.php的代码如下：

```
<?php

require_once __DIR__ .'/verdor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

$connection = new AMQPStreamConnection('localhost',5672,'guest','guest');
$channel = $connection->channel();

$channel->queue_declare('task_queue',false,true,false,false);

$data = implode(' ',array_slice($argv,1));

if(empty($data)) $data = "Hello World!";

$msg = new AMQPMessage($data,
	array('delivery_mode'=>2) // 消息持久化
);
$channel->basic_publish($msg,'','task_queue');

echo "[x]Sent ", $data, "\n";

$channel->close();
$connection->close();
?>
```

[new_task.php源码](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/new_task.php)

worker.php

```
<?php

require_once __DIR__ .'/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;

$connection = new AMQPStreamConnection('localhost',5672,'guest','guest');
$channel = $connection->channel();

$channel->queue_declare('task_queue',false,true,false,false);

echo '[*] Waiting for messages.To exit press CTRL+C',"\n";

$callback = function($msg){
	echo "[x]Received ",$msg->body,"\n";
	sleep(substr_count($msg->body,'.'));
	echo "[x]Done","\n";
	$msg->delivery_info['chennel']->basic_ack($msg->delivery_info['delivery_tag']);
};

$channel->basic_qos(null,1,null);
$channel->basic_consume('task_queue','',false,false,false,false,$callback);

while(count($channel->callbacks)){
	$channel->wait();
}

$channel->close();
$connection->close();
?>
```

[worker.php](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/worker.php)

使用消息应答和prefetch_count=1后，就可以运行一个工作队列了，持久模式选项会在即使RabbitMQ重启的情况下保留任务。

现在我们可以继续学习第三部分的内容，学习如何发送相同的消息给多个消费者。


