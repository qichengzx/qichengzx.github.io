title: PHP RabbitMQ 教程（六） - 远程调用
categories: php
date: 2016-05-03 14:35:20
tags:  [php,rabbitmq]

---

### 远程调用

在[第二节](/2016/04/17/php-rabbitmq-tutorial-two.html)中，我们学习了如何使用工作队列在多个worker中分发耗时任务。

但是如果我们需要在远程运行一个函数并等待返回结果怎么办？这是两码事，这个模式通常被称为远程过程调用（Remote Procedure Call，RPC）。

在本节中我们准备使用RabbitMQ构建一个RPC系统：一个客户端和一个可扩展的RPC服务端。由于我们没有值得分发的耗时任务，我们准备创建一个假的返回[斐波那契数列](https://en.wikipedia.org/wiki/Fibonacci_number)的RPC服务。

### 客户端接口

我们创建一个简单的客户端类来说明RPC服务如何使用。这个类会展示call方法如何发送一个RPC请求并且阻塞，直到接收到返回值。

```php
$fibonacci_rpc = new FibonacciRpcClient();
$response = $fibonacci_rpc->call(30);
echo "[.] Got ", $response, "\n";
```

####  RPC注意事项

尽管RPC在计算机学中很常见，但它十分挑剔。当程序员不知道是否是调用一个本地的方法还是一个很慢的RPC会出现这个问题。这样的困惑便导致不可预测的系统并增加不必要的调试复杂性。比起简化的软件，误用RPC会导致不可维护的无头绪代码。

记住刚才的内容，考虑下面的建议：

​	确保可以明显的看出哪个方法调用的是本地的哪个是远程的。

​	系统文档化。让组件之间的依赖变得清晰可见。

​	错误处理。当RPC服务长时间关闭客户端该作何反应？

如果有疑问，则尽量避免使用RPC。如果可以话，你应该使用异步管道——而不是RPC——像阻塞，结果被异步推送到下个计算阶段。

###  回调队列

通常在RabbitMQ上做RPC很简单。客户端发送请求消息，服务端回复消息。为了接收响应消息，我们需要在请求中附带一个"callback"队列地址，我们可以使用默认的队列。来试一试：

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

这样又带来一个新的问题，在队列接收到响应时，并不知道属于哪个请求。这也正是correlation_id属性要发挥的作用。我们为每一个请求的设定一个唯一的correlation_id值，然后，当在回调队列接收到消息时会查看它的属性，基于此，我们就可以把响应和请求进行匹配。如果发现一个未知的correlation_id值，可以安全的忽略掉这条消息 - 因为它不属于任何请求。

也许你会问，为什么应该忽略回调队列里的未知消息，而不是返回一个错误？因为服务可能会出现紊乱的情况，虽然不太可能，但是如果发生这种情况，RPC服务会在发送完响应后挂掉，但是还没有进行消息确认。如果发生了，重启RPC服务后会再次处理这个请求。这就是为什么在客户端我们必须适当的处理重复请求，而RPC服务最好的幂等的。

### 总结

![](/images/rabbitmq/python-six.png)

RPC工作流程：

​	当客户端开始运行时会创建一个匿名独有回调队列。

​	RPC请求中，客户端消息带有两个属性：reply_to用来设置回调队列，correlation_id用来唯一标识每一个请求。

​	请求被发送到rpc_queue队列。

​	RPC worker（又称worker）在队列中守护，等待新请求。当请求到达，它会进行处理，然后把结果以消息的形式发送回客户端的队列，队列名便是客户端消息带有的reply_to的值。

​	客户端等待回调队列中的数据。当消息到达，检查它的correlation_id的值。如果符合客户端发送给RPC服务器中请求的值，客户端会返回响应内容到应用中。

### 整合

斐波那契方法：

```
function fib($n){
  if($n == 0)
  	return 0;
  if($n == 1)
  	return 1;
  return fib($n-1) + fib($n-2);
}
```

定义完斐波那契方法。假定它仅接受数字类型的输入。（别期望它能处理大的数字，它很可能非常慢的处理完。）

RPC服务处理程序rpc_server.php：

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->queue_declare('rpc_queue', false, false, false, false);

fuction fib($n){
  if($n == 0)
    return 0;
  if($n == 1)
    return 1;
  
  return fib($n-1) + fib($n-2);
}

echo " [x] Awaiting RPC requests\n";

$callback = function($req){
  $n = intval($req->body);
  echo " [.] fib(",$n,")\n";
  
  $msg = new AMQPMessage(
  	(string)fib($n),
    array('correlation_id' => $req->get('correlation_id'))
  );
  
  $req->delivery_info['channel']->basic_publish($msg, '', $req->get('reply_to'));
  $req->delivery_info['channel']->basic_ack($req->delivery_info['delivery_tag']);
};

$channel->basic_qos(null, 1, null);
$channel->basic_consume('rpc_queue', '', false, false, false, false, $callback);

while(count($channel->callbacks)){
  $channel->wait();
}

$channel->close();
$connection->close();

?>
```

服务端代码相当简单：

​	和以往一样我们会从创建连接，频道，和声明队列开始

​	也许我们想要运行更多的进程。为了在多个服务器之间负载均衡，需要在$channel.basic_qos中设置prefech_count；

​	我们使用basic_consume访问队列。然后进入while循环等待请求消息，处理，然后返回响应消息。

RPC客户端 rpc_client.php：

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

class FibonacciRpcClient{
	private $connection;
	private $channel;
	private $callback_queue;
	private $response;
	private $corr_id;
  
	public function __construct(){
		$this->connection = new AMQPStreamConnection(
		  'localhost', 5672, 'guest', 'guest'
		);
		$this->channel = $this->connection->channel();
		list($this->callback_queue, ,) = $this->channel->queue_declare("", false, false, true, false);
		$this->channel->basic_consume(
			$this->callback_queue, '', false, false, false, false,
			array($this, 'on_response'));
	}
  
 	public function on_response($req){
 		if($req->get('correlation_id') == $this->corr_id){
 			$this->response = $req->body;
 		}
 	}
  	
	public function call($n){
		$this->response = null;
		$this->corr_id = uniqid();

		$msg = new AMQPMessage(
  		(string) $n,
  		array(
  			'correlation_id' => $this->corr_id,
  			'reply_to' => $this->callback_queue)
  		);
    )
      
    $this->channel->basic_publish($msg, '', 'rpc_queue');
    while(!$this->response){
    	$this->channel->wait();
    }
    return intval($this->response);
  }
};

$fibonacci_rpc = new FibonacciRpcClient();
$response = $fibonacci_rpc->call(30);
echo " [.] Got ", $response, "\n";

?>
```

现在可以查看示例的完整代码了。[rpc_client.php](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/rpc_client.php) 和 [rpc_server.php](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/rpc_server.php)。

现在RPC服务端可以运行了：

```shell
php rpc_server.php
[x] Awiting RPC resquests
```

接收斐波那契数列运行：

```
php rpc_client.php
[x] Requesting fib(30)
```

这里展现的并不是RPC服务的唯一可能实现，但它有一些重要的优势：

​	如果RPC服务太慢，可以按比例增加运行数量。试试在新控制台裕兴第二个rpc_server.php服务。

​	在客户端，RPC要求只发送和接收一条消息。不能有像队列声明一样的异步调用。结果就是，对于单一的RPC请求，客户端仅需要一个网络往返。

现在的代码还是过于简单，并没有想解决更复杂（更重要）的问题，比如：

​	要是没有服务端守护运行，客户端作何反应？

​	RPC客户端是否需要设置超时？

​	如果服务端引发异常，是否该把它发送到客户端？

​	处理前阻止无效消息（如检查范围，类型）进入？

如果想尝试，可以在[rabbitmq-management](https://www.rabbitmq.com/plugins.html) 里找到一些有用的查看队列的插件。



原文地址：[Remote procedure call (RPC)](https://www.rabbitmq.com/tutorials/tutorial-six-php.html)