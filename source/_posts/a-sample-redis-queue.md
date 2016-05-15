title: 一个简单的php+redis队列示例
categories: php
date: 2016-05-15 14:19:52
tags:  [redis,php,队列]

---

![](/images/redis/redis-300dpi.png)

虽然RabbitMQ的坑年前就开始填了，但是并没有机会在项目中实际使用，机缘巧合，换工作后第一个比较重要的事就是做一个直播的页面，如果数据直接插入或读取自数据库，数据库端的压力就太大了，当时RabbitMQ还没看完，而其他的队列程序更是没有用过，只是稍微对Redis熟悉些，于是就使用了Redis做。

[Redis](http://redis.io/)比较常见的是作为数据缓存工具使用，数据存储在内存中，减少了数据库的连接和查询，效率高，又方便。

而其实Redis也可以用来做消息队列。

### 发送

发送端首先当然是做好数据的接收和检测，比如为空的情况下给个默认值或返回错误。

特殊的情况下可能还要正则处理。

处理完插入到Redis的list([列表](http://www.redis.cn/topics/data-types-intro.html#lists))中。如果插入失败抛出异常。

```php
<?php

$redis = new Redis();
$redis->connect('127.0.0.1');

//如果redis需要认证
$redis->auth('redispasswd');

//命令行中使用，可以使用把信息作为参数的方式
$data = $argv[1];
if(empty($data)) $data = "Hello World..!":

try{
	//便于调试，在信息后边加上当前时间
	$data .= '-at-'.date('Y-m-d H:i:s');
	$redis->LPUSH('redis_queue_1',json_encode($data));
}catch(Exception $e){
	echo $e->getMessage()."\n";
}
```

#### 执行

```shell
php push.php "testinfo"
```

### 接收

由于消息是源源不断发送到队列中，所以接收端程序一般会以后台进程的方式运行。

```php
<?php
$redis = new Redis();
$redis->connect('127.0.0.1');

$redis->auth('redispasswd');

while (true) {

	try {
		$data = $redis->rPopLPush('redis_queue_1','redis_queue_1_bak')."\n";
		if($data){
			echo $data;
		}else{
			echo "没有数据\n";
		}
	} catch (Exception $e) {
		echo $e->getMessage()."\n";
	}

	//每秒读取1次
	sleep(1);
}
```

#### 执行

```shell
php get.php
```

### 更进一步

如果想把数据写入到MySQL中，该怎么办？

大多数情况下数据最终是需要持久化的，仅仅存在于redis中，也许内存会不够呢。

改造一下get.php。

```php
<?php
$redis = new Redis();
$redis->connect('127.0.0.1');

$redis->auth('redispasswd');

$dsn = 'mysql:dbname=test;host=127.0.0.1';

try{
	$pdo = new PDO($dsn,'root','mysqlpasswd',array(PDO::MYSQL_ATTR_INIT_COMMAND => 'SET NAMES utf8'));
}catch(PDOExcetion $e){
	echo $e->getMessage()."\n";
}

while(true){
	$data = $redis->rPopLPush('redis_queue_1','redis_queue_1_bak');
	if($data){
		$sth = $pdo->prepare("INSERT INTO tb_test(msg,time) VALUES (:msg,:time)");
		
		$time = time();
		
		$sth->bindParam(':msg',$data,PDO::PARAM_STR);
		$sth->bindParam(':time',$time,PDO::PARAM_INT);
		
		$sth->execute();
		$id = $pdo->lastInsertId();
      
		echo "插入成功，ID=$id \n";
	}else{
		echo "没有数据 \n";
	}
	
	//实际任务中可能需要尽可能快的把数据导入到MySQL中
	sleep(1);
}
```



#### 注意

​	这里使用了[PDO](http://php.net/manual/zh/book.pdo.php);

​	[bindParam](http://php.net/manual/zh/pdostatement.bindparam.php)方法的第一个参数对应insert语句中的对应顺序的值，如第一个bindParam方法对应第一个value值。

​	bindParam方法第二个参数为实际的值，传入变量或固定值，传入方法调用时如time()会提示：

Strict standards: Only variables should be passed by reference in /www/get.php on line 24。