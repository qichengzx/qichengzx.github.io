title: 使用Dropbox同步hexo文章
date: 2015-12-27 21:47:09
tags:  [dropbox,hexo]
---
![](/images/dropbox/20130702165945-558672261.jpg)

继之前[那次失败的尝试](https://www.qichengzx.com/2015/11/13/dropbox-sync-hexo-and-autobuild-itself.html)之后（只在当时写的时候实验过几次，每次都以服务器卡死结束），后来在又多了几篇日志之后连generate也不能愉快的完成了。索性就在本地生成然后git push到服务器。

现在想更激进一些，git只管理日志以外的东西，比如hexo的升级，或模板的调整和日志源文件。而生成的静态文件直接通过Dropbox客户端同步到服务器。

话不多说。

以下为前提：

	本地已安装hexo，和Dropbox客户端，并且客户端的同步目录已经选择到hexo的目录。
	服务器已安装dropbox服务，及相应的用户。


Dropbox的同步目录选hexo根目录或public都行，只是在服务器的处理脚本那同步修改下就行了。

以下内容假设已在服务器添加dbox用户用于dropbox服务的同步处理。并且也已经设置了与dropbox账户的关联。

#### 启动dropboxd

用dbox用户登录服务器。

然后，启动dropboxd进程。

```
~/dropbox-dist/dropboxd &
```

#### 设置文件夹监测

##### 先安装incron服务。

```
apt-get install incron
yum install incron
```

##### 开机启动

安装sysv-rc-conf，用于管理服务的启动

```
apt-get install sysv-rc-conf
sysv-rc-conf incron on
sysv-rc-conf --list //用于查看所有服务的状态

```

##### 创建监测服务

先修改下incron的编辑器

```
sudo vi /etc/incron.conf

```

在文件的最后一行，去掉editor = vi前的#，保存退出。

输入：

```
incrontab -e
```

如果当前登录的不是dbox用户，可以使用
```
incrontab -udbox -e
```

然后输入：

```
/home/dbox/Dropbox/yourfolder/ IN_ATTRIB,IN_MOVE /home/dbox/dbox.sh

```
第一个参数：用来接收Dropbox同步的文件夹

第二个参数：指监测的动作

第三个参数：处理脚本

监测的动作可以用：

```
IN_ACCESS，即文件被访问
IN_MODIFY，文件被 write
IN_ATTRIB，文件属性被修改，如 chmod、chown、touch 等
IN_CLOSE_WRITE，可写文件被 close
IN_CLOSE_NOWRITE，不可写文件被 close
IN_OPEN，文件被 open
IN_MOVED_FROM，文件被移走,如 mv
IN_MOVED_TO，文件被移来，如 mv、cp
IN_CREATE，创建新文件
IN_DELETE，文件被删除，如 rm
IN_DELETE_SELF，自删除，即一个可执行文件在执行时删除自己
IN_MOVE_SELF，自移动，即一个可执行文件在执行时移动自己
IN_UNMOUNT，宿主文件系统被 umount
IN_CLOSE，文件被关闭，等同于(IN_CLOSE_WRITE | IN_CLOSE_NOWRITE)
IN_MOVE，文件被移动，等同于(IN_MOVED_FROM | IN_MOVED_TO)
#上面所说的文件也包括目录。
```

##### 处理同步后的文件的脚本

```
cd /home/dbox/
vi dbox.sh

cd /home/dbox/Dropbox/yoursite/
cp -R public/ /var/www/yoursite/

```
最后一句要注意看你本地同步了哪些内容，还要注意与网站的目录对应。

还要注意dbox.sh要有执行权限，和yoursite的写入权限。

至此，完成。


