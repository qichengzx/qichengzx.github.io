title: crontab实现定时备份数据库
categories: linux
date: 2016-01-10 18:52:55
tags:  [crontab,mysql,linux]

---

![](https://s.qichengzx.com/att/img/201601/crontab.jpg)

crontab命令之前写过了，在[Linux crontab 访问PHP URL完成定时任务](http://segmentfault.com/a/1190000003953826)，今天写了一个用来备份数据库的脚本。

主要会用到以下几个命令：

####mysqldump

参考文章：[mysqldump导入导出数据库总结 ](http://blog.csdn.net/shellching/article/details/8129687)


####创建.sh文件：

```
cd ~
vi backup.sh
```
backup.sh内主要内容如下：

```
mysqldump -hlocalhost -uroot -p'root' --databases database1 | gzip > /var/backups/databases-database1`date +'%Y%m%d_%H%M%S'`.sql.gz
```

首先用mysqldump命令

```
	1.连接数据库
	2.选择要备份数据库
	3.选择存储备份文件的方式，这里使用了gzip了生成一个压缩包
```

根据文档，如果想备份所有数据库，可以使用

```
mysqldump -hlocalhost -uroot -p'root' --all-databases | gzip > /var/backups/databases-all-database`date +'%Y%m%d_%H%M%S'`.sql.gz
```
保存。

有备份，就会有备份后的处理，显而易见的问题是备份多了会比较占空间，并且也用不到那么多备份。所以备份完成删除掉一段时间以前的就可以了。这步也可以在备份前做，无所谓。

这里又用到了find命令

参考文章：[Linux中find常见用法示例](http://www.cnblogs.com/wanqieddy/archive/2011/06/09/2076785.html)

####删除之前的备份文件

在刚才的backup.sh中继续输入：

```
cd /var/backups/
rm -rf `find . -name '*.sql.gz' -mtime +10`

```

这句命令有两部分，

第一部分是删除命令：```rm -rf```。就是那句一定要慎用的命令了，

第二部分是找到：当前目录，名字以'.sql.gz'结尾的，更改时间在10天以前的文件。

'.'表示当前目录，由于上一句是```cd /var/backups/```，所以这里使用当前目录即可。

'-name '和'-mtime'参数是find命令的条件。

具体的说明，和其他条件可以参考前边说到的文章。

到这里基本上备份脚本就完成了。

但是为了什么时候突然想看一下日志，或者备份出错的时候查问题，还可以在脚本里加上记录日志的命令：

####日志

```
echo 'Begin Backup Database At :' `date +'%Y-%m-%d %H:%M:%S'`
```
这里又用到了date。

参考文章：[Linux下date命令，格式化输出，时间设置 ](http://blog.csdn.net/jk110333/article/details/8590746/)

脚本保存之后，记得添加执行权限；

```
sudo chmod +x backup.sh
```

接下来就是在系统里添加crontab任务了。

```
cd /etc/
vi crontab

```

在文件末尾，加上

```
m  h dom mon dow user	command
00 5 * * * root /home/yourname/backup.sh >> /var/log/backup.log
```

这样，backup.sh里的echo就会输出到/var/log/backup.log中了。

Over。


