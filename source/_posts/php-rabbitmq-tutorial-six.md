title: PHP RabbitMQ 教程（六） - 远程调用
categories: php
date: 2016-05-03 14:35:20
tags:  [php,rabbitmq]

---

### 远程调用

在[第二节](/2016/04/17/php-rabbitmq-tutorial-two.html)中，我们学习了如何使用工作队列在多个worker中分发耗时任务。

但是如果我们需要在远程运行一个函数并等待返回结果怎么办？这是两码事，这个模式通常被称为远程过程调用（Remote Procedure Call，RPC）。

在本节中我们准备使用RabbitMQ构建一个RPC系统：一个客户端和一个可扩展的RPC服务端。由于我们没有值得分发的耗时任务，我们准备创建一个假的RPC服务，来返回[斐波那契数列](https://en.wikipedia.org/wiki/Fibonacci_number)。

### 客户端接口

我们创建一个简单的客户端类来说明RPC服务如何使用。这个类会揭露call方法如何发送一个RPC请求并且阻塞，直到接收到返回值。

```php
$fibonacci_rpc = new FibonacciRpcClient();
$response = $fibonacci_rpc->call(30);
echo "[.] Got ", $response, "\n";
```

####  RPC备注

虽然RPC在计算机学中很常见，它经常被批评。当一个程序员没有意识到



###  回调队列

通常在RabbitMQ上做RPC很简单。客户端发送一个请求消息，服务器端回复消息。为了接收响应消息，我们需要在请求中附带一个"callback"队列地址，我们可以使用默认的队列。来试一试：

```php
list($queue_name, ,) = $channel->queue_declare("", false, false, true, false);

$msg = new AMQPMessage(
	$payload,
	array('reply_to' => $queue_name));

$channel->basic_publish($msg, '', 'rpc_queue');

# ... then code to read a response message from the callback_queue
```



####  消息属性

AMQP协议定义了14个消息属性。大部分不常用，下面的除外：

​	delivery_mode：值为2时表示持久化，1为临时的。也许你还记得这个属性来自第二节。

​	content_type：编码格式，比如经常用的JSON格式，良好的做法是设置为：application/json。

​	reply_to：通常用来定义回调队列名称。

​	correlation_id：用来关联RPC的响应和请求。

### Correlation Id

在上面的方法中我们建议为每一个RPC请求创建一个回调队列。这样非常低效，但是幸运的是有更好的办法 - 我们可以为每一个客户端创建一个单独的回调队列。

这样又带来一个新的问题，在队列接收到响应时，并不知道响应属于哪个请求。这也正是correlation_id属性要发挥的作用。我们为每一个请求的设定一个唯一的值，然后，当在回调队列接收到消息时会查看它的属性，基于此，我们就可以把响应和请求进行匹配。如果发现一个未知的correlation_id值，可以安全的忽略掉这条消息 - 因为它不属于任何请求。

也许你会问，为什么应该忽略回调队列里的未知消息，而不是返回一个错误？因为