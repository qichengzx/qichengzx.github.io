title: PHP RabbitMQ 教程（二） - 工作队列
categories: php
date: 2016-03-07 14:20:17
tags:  [php,rabbitmq]

---

## 工作队列

##### （使用php-amqplib库）

在本教程[第一部分](/) 我们已经写完了从一个指定队列发送和接收消息的程序。在这一章节中，我们会创建一个工作队列（Work Queue）来分发耗时的任务给多个工作者（workers）。

工作队列（也被称为 任务队列-task queue）主要是避免立即执行资源密集型任务并且还要等待它执行完毕。相反，需要让任务稍后执行，我们把一个任务当做一条信息发送给队列，后台运行的工作者（workers）会取出任务并执行，当运行多个workers时任务会在它们之间共享。

这个概念在web应用中非常有用，可以在短暂的HTTP请求期间处理一些复杂的任务。

## 准备

在前面的部分我们发送了一条内容为“Hello World”的信息，现在我们会发送一些字符串，把这些字符串当做复杂的任务，我们并没有一个实际的任务，像是图片缩放，或者转换PDF文件，所以我们使用sleep方法来假设任务很繁忙。我们会在字符串中加入一些“.”来表示复杂复杂程度；每一个“.”表示需要耗时1秒，比如，“Hello ...”代表需要耗时3秒。

我们从上一节的基础上稍微改动了一下send.php，来允许消息可以从命令行发送，这个程序会发送任务到队列中，把它命名为new_task.php

```
$data = impllode(' ',array_slice($argv,1));
if(empty($data))$data = "Hello World";
$msg = new AMQPMessage($data,
						array('delivery_mode'=>2)#make message persistent
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

然后在运行：

```
php new_task.php "A very hard task which takes two seconds.."
php wordker.php
```

## 轮询分发

使用工作队列的一个好处就是它能够并行的处理队列。如果有太多工作需要处理，只需要添加新的worker就可以了。

首先，我们试着同时运行worker.php，它们都会从队列接收到消息，但是到底是不是这样呢？我们看一下。

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

在第三个终端中我们会发送新的任务，一旦消费者程序已经在运行，就可以发送一些消息了。

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

## 消息响应

执行一个任务会消耗一定的时间，也许你想知道如果一个消费者在执行一个耗时较长的任务然后执行一部分的时候挂掉会发生什么。在我们当前的代码中，一旦RabbitMQ把消息分发给消费者便会立即从内存中移除。这种情况下，如果停止一个worker，它正在处理的消息就会丢失。同时其他所有发送给这个worker的还没有处理的消息也会丢失。

但是我们不想丢失任何任务，如果一个worker挂掉，需要让任务发送到另一个worker。

为了确保消息永不丢失，RabbitMq支持消息响应（message acknowledgements），消费者会发送一个应答告诉RabbitMQ已经收到了特定的消息，并且已经处理，这样RabbitMQ就可以删掉它了。

如果一个消费者程序在未发送应答之前挂掉了（频道关闭，链接关闭，或者TCP连接丢失），RabbitMQ会认为消息没有完全处理并且会重新推送到队列中。如果此时有其他的消费者程序在线上运行，它会很快发送给另一个消费者。这样就可以确保消息不会丢失，即使worker偶尔挂掉。

There aren't any message timeouts; RabbitMQ will redeliver the message when the consumer dies. It's fine even if processing a message takes a very, very long time.

消息响应默认是关闭的。现在可以通过设置basic_consume的第四个参数为false(true表示不开启应答)，然后在处理完任务的时候从worker发送一个合适的应答内容。

```
$callback = function($msg){
	echo "[x] Received ",$msg->body,"\n";
	sleep(substr_count($msg->body,'.'));
	echo "[x] Done","\n";
	$msg->delivery_info['channel]->basic_ack($msg->delivery_info['delivery_tag]);
};
$channel->basic_consume('task_queue','',false,false,false,false,$callback);
```

这样我们就可以确保当你CTRL+C杀掉一个正在处理消息的worker的时候，消息并不会丢失。很快在这个worker挂掉之后，未应答的消息就会发送给RabbitMQ（即RabbitMQ收不到应该收到的应答消息）。

### 忘掉应答

这是一个通用的错误


## 消息持久化

我们已经学习了确保即使消费者程序挂掉，任务也不会丢失。但是任务还是会在RabbitMQ服务停止的时候丢失。

当RabbitMQ退出或崩溃，它会丢失之前所有的队列和消息，除非你特意告诉它。所以我们必须把队列和消息设为持久化。

首先，需要确保RabbitMQ永不丢失队列。需要把它声明为<i>持久化（durable）</i>，所以修改queue_declare的第三个参数为true：

```
$channel->queue_declare('hello',false,true,false,false);
```

尽管这行代码本身是正确的，但是仍然不会正确运行。因为在之前已经定义过一个非持久化的 hello 队列。RabbitＭＱ不允许使用不同参数重新定义一个已经存在的队列，它会返回一个错误。但是可以用一个快捷的方法去解决，定义一个不同名字的队列，比如 task_queue：

```
$channel->queue_declare('task_queue',false,true,false,false);
```

需要把生产者和消费者都设置为 true。

这时候，我们就可以确保在RabbitMq重启之后task_queue队列不会丢失。现在需要设置消息持久化了 - 通过设置AMQPMessage的属性数组中消息属性 delivery_mode = 2来达到。

```
$msg = new AMQPMessage($data,
		array('delivery_mode'=>2) //设置消息持久化
	);
```

### Note on message persistence


## 公平分发

也许你注意到它仍没有像我们想的那样去派发任务，比如在两个worker的情况下，处理奇数消息的比较繁忙，处理偶数消息的比较轻松，一个worker不断的忙碌而另一个几乎不需要工作，但是RabbitmQ并不知道这些，并且继续一如既往的派发消息。

这是因为RabbitMQ在消息进入队列的时候只管去派发，并不注意消费者未做出响应的消息数。它只是把第n条消息发送给第n个消费者。

