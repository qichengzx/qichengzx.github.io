title: node连接MySQL并读取数据
date: 2015-12-03 21:38:37
tags:  [node,mysql]
categories: javascript
---


![](http://7b1hhm.com1.z0.glb.clouddn.com/hexo201312030901.jpg)

文章部分代码与[之前整理的一篇文章](http://segmentfault.com/a/1190000002995355)一样。

本文主要实现node连接MySQL并读取指定表的数据输出到ejs模板中。

MySQL是一款非常常用的开源数据库，[npm中也有MySQL的包](https://www.npmjs.com/package/mysql)。


###MySQL测试库的表结构：

```
DROP TABLE IF EXISTS `news`;
CREATE TABLE `news` (
  `id` int(11) NOT NULL DEFAULT '0',
  `title` varchar(45) NOT NULL DEFAULT '',
  `createtime` int(11) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

LOCK TABLES `news` WRITE;
INSERT INTO `news` VALUES (1,'news one',1449064787),(2,'news two',1449064790),(3,'news three',1449064900);
UNLOCK TABLES;
```

### 安装Nodejs MySQL包

```
npm install mysql
```

### server.js

```
//server.js
var express = require('express');
var mysql = require('mysql');
var moment = require('moment');//用来格式化UNIX时间戳
var app = express();
```
配置MySQL数据库

```
var connection = mysql.createConnection({
	host:'localhost',
	user:'root',
	password:'root',
	database:'node'
});
```
连接数据库，并在成功或失败时输出log

```
connection.connect(function(err){
	if(!err){
		console.log('Database is connected...\n\n');
	}else{
		console.log('Error Connecting Database...\n\n');
	}
});
```
定义表

```
var TABLE = 'news';
```
设定模板引擎

```
app.set('view engine','ejs');
```
这就是很普通的路由，表示接收访问首页的请求

```
app.get('/',function(req,res){
	//定义一组数据
	var data = [
					{ name : 'Bloody Mary' , drunkness : 3 },
					{ name : 'Martini' , drunkness : 5 },
					{ name : 'Scotch' , drunkness : 10 }
				];

	//MySQL查询
	connection.query(
		//普通的SQL
		'select * from ' + TABLE,
		//查询回调
		function(err,results,fields){
			if(err){
				//输出错误
				throw err;
			}
			
			if(results){
				console.log(results);
				//express render一个页面
				res.render('pages/index',{
					title:'test',
					results:results,
					data:data,
					moment:moment
				});
			}
		}
	)

});

app.listen(8888);
console.log('8888 is the magic port');
```

### index.ejs

```
<% include ../partials/head %>

<body class="container">
	<main>
		<div class="jumbotron">
			<h2>data from static</h2>
			<ul>
				<% data.forEach(function(d) { %>
				<li>
					<%= d.name %>
					<span><%= d.drunkness %></span>
				</li>
			    <% }); %>
			</ul>
		</div>
		<div class="jumbotron">
			<h2>data from mysql</h2>
			<ul>
				<% results.forEach(function(result) { %>
				<li>
					<%= result.id %>
					<span><%= result.title %></span>
					<span><%= moment(result.createtime*1000).format('YYYY-MM-DD, hh:mm:ss') %></span>
				</li>
			    <% }); %>
			</ul>
		</div>
		
	</main>

	<footer>
		<% include ../partials/footer %>
	</footer>
	
</body>
</html>
```
至此，就可以正常输出从数据库里读取出来的数据了。

**小插曲**

之前在练习的时候，require MySQL 之后npm install 只安装了require的几个包，昨天再写的时候装了一大堆。
另外之前在使用moment的时候并没有在render页面的时候传入moment，直接就可以用了，昨天也报错了。
