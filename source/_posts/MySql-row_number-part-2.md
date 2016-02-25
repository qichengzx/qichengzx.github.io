title: MySQL实现row_number第二部分
categories: mysql
date: 2016-02-24 17:58:24
tags:  [mysql]

---
接上篇：[MySQL实现row_number](https://www.qichengzx.com/2016/02/03/MySQL%E5%AE%9E%E7%8E%B0row_number%20.html)

### 为分组增加row number

row_number的分析函数怎么样？（原文：How about  row_number “over partition by” functionality? ），比如，如果想为每一个组增加row number，并且在每一个新的分组重置它。

查看下图中payments表。

```
SELECT
    customerNumber, paymentDate, amount
FROM
    payments
ORDER BY customerNumber;
```

![](/images/mysql-row-number/mysql-row_number-payments-table.jpg)


设想下，为每一条customer数据增加一个当前行的编号，并且每当customer的number字段变化的时候行编号都被重置。

为了实现这个，需要使用两个临时变量，一个作为当前行编号，另一个用来存 上一条customer number与当前行的number进行对比，查询语句如下。（这句有点绕，可能翻译的不太准确，请以原文为准。To achieve this, you have to use two session variables, one for the row number and the other for storing the old customer number to compare it with the current one as the following query:）

```
SELECT 
    @row_number:=CASE
        WHEN @customer_no = customerNumber THEN @row_number + 1
        ELSE 1
    END AS num,
    @customer_no:=customerNumber as CustomerNumber,
    paymentDate,
    amount
FROM
    payments
ORDER BY customerNumber;
```

在这段查询代码中，我们使用了CASE声明，如果customer的number不变，就给row_number变量+1，否则，重置为1，结果如下：

![](/images/mysql-row-number/mysql-row_number-per-group.jpg)

与为每一行记录生成row_number一样，也可以使用派生表（derived table）和交叉连接（cross join）生成同样的结果。

```
SELECT 
    @row_number:=CASE
        WHEN @customer_no = customerNumber THEN @row_number + 1
        ELSE 1
    END AS num,
    @customer_no:=customerNumber as CustomerNumber,
    paymentDate,
    amount
FROM
    payments,(SELECT @customer_no:=0,@row_number:=0) as t
ORDER BY customerNumber;
```

本教程中，我们展示了如何在MySQL中模仿row_number方法。


原文地址：[MySQL row_number, This Is How You Emulate It](http://www.mysqltutorial.org/mysql-row_number/)
	