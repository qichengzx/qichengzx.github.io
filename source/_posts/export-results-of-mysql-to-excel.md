title: MySQL命令行导出结果集到Excel
categories: mysql
date: 2015-11-21 21:24:35
tags:  [mysql,excel]

---

![](http://7b1hhm.com1.z0.glb.clouddn.com/hexo1631785_mysql.gif)

某个需求，为了简单单独建表存储，而且没有相关的后台管理方法，无法查看，或导出，但是又突然出现需要导出这个需求。

所以突然想到MySQL是否带了这种直接导出到文件中的方法，查资料后发现果然有。

只需要在命令行中执行如下查询即可。

```
SELECT * FROM my_table INTO OUTFILE 'file.csv' FIELDS TERMINATED BY ',';
```

有几点需要注意：

	0.文件将在服务器上生成，所以需要需要MySQL进程有生成文件的权限。
	1.必须为一个不存在的文件名。
	
其实还有一些功能，或者说是说明，这里并没有写。


官方文档：

[13.2.9.1 SELECT ... INTO Syntax](http://dev.mysql.com/doc/refman/5.5/en/select-into.html)