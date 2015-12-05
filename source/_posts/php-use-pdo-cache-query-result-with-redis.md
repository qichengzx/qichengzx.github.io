title: PHP使用Redis作为缓存使用PDO读取MySQL数据
date: 2015-12-05 22:37:09
tags:  [php,pdo,redis,mysql]
categories: php
---

![](http://7b1hhm.com1.z0.glb.clouddn.com/hexo1414772222.jpg)

图片转自：[Using PDO::fetchAll – Examples with codes and output results](http://blog.encodez.com/blog/using-pdofetchall-examples-with-codes-and-output-results)

一个简单的例子。

**实现功能：**
	
	1.使用PDO读取数据
	
	2.使用Redis缓存结果
	
	3.再次查询时会从Redis查询，减少MySQL查询
	
没有注意代码的逻辑，仅实现思路。

```
<?php

//一上来肯定是配置pdo的连接信息。其实这步在这个示例中可以放到第一个else中。
$dsn = 'mysql:dbname=node;host=127.0.0.1';
$user = 'root';
$password = '';

//连接Redis
$redis= new Redis();
$redis->connect('127.0.0.1',6379);

//接收查询参数
$id = intval($_GET['id']);

//设置在Redis中存储的KEY
$MY_NODE_KEY_ = 'TEST_PDO_REDIS_ID_';

//拼接KEY和查询ID，读取Redis，
$cache = $redis->get($MY_NODE_KEY_.$id);

//用来插入log的时间参数
$date = date("Y-m-d H:i:s");
if($cache){
	//Redis缓存存在则直接输出
	print_r(json_decode($cache,true));
	//并记录log
	error_log($date."---read from redis \r\n", 3, './debug.txt');
}else{
	//缓存不存在，则连接PDO
	try {
    	$pdo = new PDO($dsn, $user, $password);
    	error_log($date."---Connection Succcess \r\n", 3, './debug.txt');
		
		//查询
	    $query = $pdo -> query("select * from news where id = '$id'");
		//设置结果集为数组
	    // [PDOStatement::fetch](http://php.net/manual/zh/pdostatement.fetch.php)
	    $query->setFetchMode(PDO::FETCH_ASSOC);
	    
	    $rs = $query->fetch();
	
	    if (is_array($rs)) {
	    	//查询完成，以json格式写入Redis中。
			$redis->set($MY_NODE_KEY_.$rs['id'],json_encode($rs));
			print_r($rs);
			error_log($date.'read from mysql', 3, './debug.txt');
	    }
	    
	//PDO连接出错
	} catch (PDOException $e) {
		//输出错误信息，并记录log中
    	echo 'Connection failed: ' . $e->getMessage();
    	error_log($date."---Connection failed \r\n", 3, './debug.txt');
	}
}

```

写的很糙，逻辑已经简单到没有逻辑了。

一句话说一下思路就是：
	先看有没有缓存
	木有就查数据库，写入缓存
	

参考资料：
	[PHP 数据对象](http://php.net/manual/zh/book.pdo.php)
	[PHP PDO的简单使用(query(),exec(),prepare(),Transaction,行锁)](http://www.cnblogs.com/zemliu/archive/2012/05/08/2490953.html)