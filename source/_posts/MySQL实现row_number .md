title: MySQL实现row_number
categories: mysql
date: 2016-02-03 18:25:26
tags:  [mysql]

---

在本教程中，我们将在MySQL中实现一个非常实用的row_number功能。

row_number是一个返回数据排序编号的排名方法，从1开始。我们经常需要用到row_number去生成某些报表，不幸的是，MySQL并不像MSSQL，Oracle一样支持这个方法。在MySQL中如果想实现这个功能需要临时变量。

为了在MySQL中实现row_number，需要在查询中实用临时变量，下图中从employees表中查询出5条数据，并且从1开始，为每一行添加了编号(row number)。

```
SET @row_number = 0;
 
SELECT 
    (@row_number:=@row_number + 1) AS num, firstName, lastName
FROM
    employees
LIMIT 5;
```

![](/images/mysql-row_number-session-variables.jpg)

在上面的查询中：

	首先，我们定义了一个变量叫做row_number,初始值为0，row_number是以@为前缀的一个临时变量。
	
	然后，在查询中，我们为这个变量每次+1，LIMIT分句是为了限制返回的结果数，此处设为5.
	
另一个方法是使用临时变量作为派生表，联合主表。看下边的查询语句：

```
SELECT 
    (@row_number:=@row_number + 1) AS num, firstName, lastName
FROM
    employees,(SELECT @row_number:=0) AS t
LIMIT 5;

```

需要注意，派生表必须使用别名，以使查询语句在语法上没有问题。

由于时间关系（回家过年），暂时到这里，文中部分内容可能翻译的有点不太顺，可以直接看下原文，原文中还有一部分讲分组查询中实用row_number。

这个之后再写。
	
原文地址：[MySQL row_number, This Is How You Emulate It](http://www.mysqltutorial.org/mysql-row_number/)
	