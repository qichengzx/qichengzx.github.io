title: 利用Dropbox同步hexo的文章源文件并自动生成文章发布
layout: photo
date: 2015-11-13 14:12:35
tags:  [hexo,dropbox]
---

** 阅读之前默认为已在服务器安装hexo，也默认服务器可以访问Dropbox **

** 不建议服务器内存小于512M ，谁卡谁知道 **


### 最初的最初

先说下我最终完成的结构：

	1.本地有一个日志文件夹，用于存放md文件，或HTML文件，作为本地的Dropbox客户端同步的目录
	2.服务器home文件夹中init一套hexo程序，用于接收本地的md文件，和generate，minify
	
流程：

	1.本地撰写，本地客户端自动同步
	2.服务器Dropbox设置的同步目录接收本地的内容并cp到专门用于生成的hexo目录里
	3.服务器用于生成的hexo生成新日志，cp到web目录中
	
这么做的原因是，如果一共（是整个博客一共）只有一篇文章或几篇文章，那几乎没影响，如果，有几十篇了，据我观察，生成很耗时，可能会导致搜索引擎访问出现404，用户打开一个生成到一半的文件，用户打开一个木有样式的文章。


### STEP 0 安装Dropbox

[注册Dropbox](https://db.tt/mMNtRA6x)。


```
if( ! has_dropbox_account() ){
	return "请先注册个账户啊亲";
}
```

由于Dropbox默认安装在~，所以建议新建一个专用于同步的账户，如dbox。

```
adduser dbox
passwd dbox 
```

** 请记住密码。请记住密码。请记住密码。 **

然后，用新建的账户登录进去。

终于可以安装Dropbox了。

这里要区分一下你的系统，

** 32位: **

```
cd ~ && wget -O - "https://www.dropbox.com/download?plat=lnx.x86" | tar xzf -
```

** 64位: **

```
cd ~ && wget -O - "https://www.dropbox.com/download?plat=lnx.x86_64" | tar xzf -
```

然后就可以运行Dropbox的守护程序了

```
~/.dropbox-dist/dropboxd &
```

第一次在新的电脑上启动，会提示：

	此电脑尚未与任何 Dropbox 帐户关联...

	请访问 https://www.dropbox.com/cli_link_nonce?nonce=95cd317d2***** 来关联此设备。

	此电脑现在已与 Dropbox 关联。欢迎 yourname username


** 注意在出现“此电脑现在已与 Dropbox 关联。欢迎 your username”前不要ctrl+c 退出这个程序。 **

打开授权链接后会出现如下的提示，选择连接即可。

![](http://7b1hhm.com1.z0.glb.clouddn.com/hexo33B2BAAF-6996-41AA-BFA3-FC177106F62A.png)

一定要注意，不然你都会奇怪为啥再次启动这个进程的时候还会出现“尚未关联的提示”

验证成功后，会在当前用户的home目录中创建Dropbox目录，即Dropbox同步的目录。

此时，如果你打开了 top，那应该发现此时服务器的内存有点捉急了，原因是Dropbox这个进程占的很多，所以一般情况下killall dropbox 退出就好了，壕请随意。


### STEP 1 设置文件夹监测


**安装incron服务**
	
incron是Linux下一个监测文件变化的服务
	

```
apt-get install incron
yum install incron

```

** 设置开机启动 **

安装sysv-rc-conf，用于管理服务的启动

```
apt-get install sysv-rc-conf
sysv-rc-conf incron on

```

** 创建监测任务 **

先修改下incron的编辑器

```
vi /etc/incron.conf

```

（此时可能需要sudo权限，因为是dbox用户）在文件的最后一行，去掉editor = vi前的#，保存退出。

输入：incrontab -e


```
/home/dbox/Dropbox/yourfolder/ IN_ATTRIB,IN_MOVE /home/dbox/hexo.bash

```

第一个参数：用来接收Dropbox同步的文件夹

第二个参数：指监测的动作

第三个参数：处理脚本

监测的动作可以用：

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


可以选自己需要的动作

接下来写处理脚本
```
vi hexo.bash
```


```
cp -R /home/dbox/Dropbox/yourfolder/* /home/dbox/dropasync/source/_posts/;
cd /home/dbox/dropasync/ && hexo gm
cd ~ && ./.dropbox-dist/dropboxd

```

理论上到这里已经可以了。

~~但是实际上我本人在测试的时候，512内存的服务器会只剩下4M内存，然后再执行任何命令都提示
```-bash: fork: Cannot allocate memory```，持续很长时间，一开始以为是同时跑的进程太多导致内存不够，把没用的mongodb，redis都kill之后还是这样，后来干脆在执行这个脚本的时候把Dropbox的后台禁掉，也不行，之后想到了在这段脚本里echo一段数字到log中，执行的时候发现会写入多次，但是还是不清楚为什么会出现这种情况，进程里也是出现了两个或多个hexo，于是干脆就在这段脚本开始时先kill dropbox和hexo，在末尾再kill hexo。实际运行起来，内存还是会降到最低，但是持续时间明显减少到可以接受的程度。~~

---
2015-11-21 14:06:07 附截图
---

![](http://7b1hhm.com1.z0.glb.clouddn.com/hexoF2A913D3-14C5-4FC9-B177-500AC0434036.png)

---
2015-11-14 22:38:06 测试发现，上述带删除线的方法依然不行。内存依然会榨干很长时间。

---


至于为何出现这个情况，咱不可知。。。总不能完全是因为内存太小吧……

后续应该还会继续改进这个脚本。

**另外，在安装这个用于生成日志的hexo程序的时候也出现了npm killed的问题，查了资料发现也是因为内存不足，npm install -d 可以查看安装过程，如果出错可以用来定位到底是什么原因引起错误**


一些参考资料：

[Hexo 服务器端布署及 Dropbox 同步](http://lucifr.com/2013/06/02/hexo-on-cloud-with-dropbox-and-vps/)

[用Hexo+Vps搭建博客并用Dropbox同步自动发布](http://www.fanicy.com/2014/06/01/0001.hexowithvpsdropbox/)

[incrontab(5) - Linux man page](http://linux.die.net/man/5/incrontab)

[incron：linux下基于文件的事件触发](http://wlx.westgis.ac.cn/tag/incrontab/)

[Why is chkconfig no longer available in Ubuntu?](http://askubuntu.com/questions/221293/why-is-chkconfig-no-longer-available-in-ubuntu)

[Use incron to Trigger Action when File Changes 如果设置incrontab出现问题可以参考这篇](https://www.garron.me/en/linux/use-incron-rsync-dropbox-backup.html)
