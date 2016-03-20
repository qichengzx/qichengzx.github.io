title: PHP RabbitMQ 教程（二） - 工作队列
categories: php
date: 2016-03-07 14:20:17
tags:  [php,rabbitmq]

---

## 工作队列

##### （使用php-amqplib库）

在本教程[第一部分](/) 我们已经写完了发送消息，和从一个指定队列接收消息。在这一部分，我们会创建一个工作队列（Work Queue）来分发耗时的多个workers。

工作队列（也被称为 任务队列）的主要思想是避免立即执行资源密集型任务并且还要等待它执行完毕。相反，需要让任务稍后执行，我们包装一个任务作为一条信息发送给队列，后台运行的工作进程会弹出任务并最终执行，当运行多个workers时任务会被共享。

这个概念在web应用无法在短暂的HTTP请求期间处理复杂的任务时尤其有用。

## 准备

在前面的部分我们发送了一条内容为“Hello World”的信息，现在我们发送字符串代表复杂的任务，我们并没有一个实际的任务，像是重置图片的大小，或者生成PDF文件，所以我们使用sleep方法来假设任务很繁忙。我们会在字符串中加入一些“.”来表示它很复杂；每一个“.”表示需要花费一秒的工作，比如，“Hello ...”代表需要花费3秒的工作。

我们从上一节的基础上稍微改动了一下send.php，来允许消息可以从命令行发送，这个程序会加入到队列中，所以命名为new_task.php

```
$data = impllode(' ',array_slice($argv,1));
if(empty($data))$data = "Hello World";
$msg = new AMQPMessage($data,
						array('delivery_mode'=>2)#make message persistent
						);
$channel->basic_publish($msg,'','task_queue');
echo "[x] Sent ",$data,"\n";
```

上一节的receive.php也需要一些变化：需要去为消息中的每一个“.”伪造一秒的工作。它会从队列中弹出消息，然后运行，所以命名为worker.php：

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

注意我们伪造的任务需要花费时间（即字符串中要有一些"."）

然后在运行：

```
php new_task.php "A very hard task which takes two seconds.."
php wordker.php
```

## 循环分发















