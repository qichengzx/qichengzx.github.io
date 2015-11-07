title: vps 搭建个人git服务器
date: 2015-11-07 23:34:10
tags: vps git 
---


以下内容主要来自
	[How To Set Up a Private Git Server on a VPS](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-private-git-server-on-a-vps)

之前在DigitalOcean买了个5刀的vps，本来是想搞VPN的，但是没成功，后来把个人博客放这了，后来又觉得有点浪费，索性重新启用这个域名来写技术文章。

本来是用的WP的但是也一直没写，后来又想折腾别的程序试一下，就选了现在的[hexo](https://hexo.io)，昨天在自己电脑上安装了，也可以写文章页可以默认 `hexo server` 运行，然后，我愉快的把命令行关掉之后，就傻了。

后来也在官方文档和Google里看有什么办法能让它在后台运行，官方说可以 `hexo s &`，然并卵，官方和Google都说可以用 `forever` `pm2` ，一样然并卵，今天早晨起来继续弄的时候，觉得还是放弃吧，既然有public文件夹，还是用nginx去解析吧。这个都是后话了。

下边说怎么在vps上安装git服务器，昨天和今天上午也看了一些资料一直也不成功。晚上从DigitalOcean社区里看到这篇文章，然后就成功了。


#### 0 在本地生成ssh key
`
	ssh-keygen -C "youremail@mailprovider.com" 
	Generating public/private rsa key pair.
	Enter file in which to save the key (/home/flynn/.ssh/id_rsa):
	Enter passphrase (empty for no passphrase):
	Enter same passphrase again: 
	Your identification has been saved in foo_rsa.
	Your public key has been saved in foo_rsa.pub.
`
注意替换成自己的邮箱，可以一路回车，也可以在 Enter passphrase 的时候输入一个密码保护一下。如果已经生成过了可以 **略过这步**。
#### 1 在服务器添加git用户
首先切换到root用户，`su -` 。
然后添加git用户。（用户名不一定是git，但是习惯上用这个名）

`useradd git`

设置密码

`passwd git`

输入两次密码即可，用户创建完毕。我在操作的时候并没有在/home/下创建git用户的目录。

所以可能需要自己手动创建

`cd /home/`

`mkdir git `

`sudo chown -R -v git:git git/`

现在可以安装git服务了。


    CentOS/Fedora: yum install git
    Ubuntu/Debian: apt-get install git

DO文档说可以这样，但是部分资料里写要 install git-core，但是因为之前安装过git-core，所以不确定是不是DO文档上是正确的，**所以此处DO的文档可能不准确。**（不确定）

#### 2 把本地的ssh key添加到服务器的允许访问列表里

登录进服务器，切换到git用户。

`su git`

然后创建git用户的.ssh文件夹及authorized_keys文件，

`mkdir ~/.ssh && touch ~/.ssh/authorized_keys`


&&的意思是执行完第一个命令后紧接着执行第二个命令，分开写一样可以，连着写逼格更高。

然后回到本地的命令行里，

`cat .ssh/id_rsa.pub | ssh git@123.45.56.78 "cat >> ~/.ssh/authorized_keys"`

第一个cat 后的文件地址取决于你电脑上这个文件的路径，后边是说把前边你本地的id_rsa.pub的内容写到远程服务器上git 这个用户的.ssh/authorized_keys文件里，也就是前边创建的那个文件。

所以，不出意外的话，此时可以在服务器上运行

`cat ~/.ssh/authorized_keys ` 就可以看到你本地电脑生成的key了，不出意外的话结尾应该是你生成key填写的邮箱。

#### 3 初始化本地仓库

在服务器上任何一个位置，执行

`git init --bare project.git`，这样，在你的开发机和服务器环境是就可以用到这个project.git了。

（为已存在的git项目）设置远程仓库的URL

`git remote set-url origin git@git.droplet.com:project.git`

git@git.droplet.com:project.git 替换成你的git用户名，你的ip或域名，及你的服务器上git的文件夹

如果是一个新的仓库，是这样

`git init && git remote add origin git@git.droplet.com:project.git`

git@git.droplet.com:my-project.gi 只是服务器上


### 总结：

	所以实际上上边的流程是
	
	0. 在服务器上比如home/git 目录里git init --bare project.git
	1. 在自己电脑，某个目录，git init 或者git clone git@git.droplet.com:project.git
	2. 在服务器上比如www目录，git init 或git clone git@git.droplet.com:project.git
	
	不出意外，这样就可以了。
	
	本地添加一个新文件，git add . , git commit -m "test" ,git push origin master，登录服务器，切换到www目录，git pull origin master，就可以了。
	
	
参考资料:

[How To Set Up a Private Git Server on a VPS](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-private-git-server-on-a-vps)

[Git on the Server - Setting Up the Server](https://git-scm.com/book/it/v2/Git-on-the-Server-Setting-Up-the-Server)