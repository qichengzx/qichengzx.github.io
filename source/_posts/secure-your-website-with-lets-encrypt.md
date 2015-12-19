title: 使用Let's Encrypt制作数字证书
date: 2015-12-19 16:38:37
tags:  [https,ssl]
---
![](https://s.qichengzx.com/img/201512/https.png)

Let's Encrypt是一个新的CA机构，提供非常简单并且免费的TLS／SSL证书服务。

可谓万众期待。

说下我的环境：
	
	Ubuntu 14.04.3
	Nginx 1.4.6
	

首先你要有个域名，其次要有个服务器。

### STEP 0 － 安装Let's Encrypt 客户端
 
##### 安装git

先更新下系统

```
sudo apt-get update
sudo apt-get -y install git
```

##### Clone Let's Encrypt

```
sudo git clone https://github.com/letsencrypt/letsencrypt /opt/letsencrypt
```

这里把下载的文件放到了/opt/letsencrypt文件夹

### STEP 1－生成证书

首先需要关闭80端口

```
sudo nginx -s stop
```

接下来运行Let's Encrypt

```
cd /opt/letsencrypt
./letsencrypt-auto certonly --standalone
```

注意：需要超级用户权限，所以需要输入密码

之后会出现提示框，需要输入邮箱，用于接收提醒和，如果不幸丢失了Key，找回的时候也需要邮箱。

根据提示进行操作即可，接下来啥同意用户协议。

下一步则是输入你需要生成证书的域名了，如果需要一个证书给多个域名使用，这些域名则要全部输入，（example.com和www.example.com是不同的域名），域名之间用空格，逗号，左斜杠 分隔。

成功后会有如下提示：

```
IMPORTANT NOTES:

 - If you lose your account credentials, you can recover through
   e-mails sent to sammy@digitalocean.com
 - Congratulations! Your certificate and chain have been saved at
    /etc/letsencrypt/live/example.com/fullchain.pem. Your
   cert will expire on 2016-03-15. To obtain a new version of the
   certificate in the future, simply run Let's Encrypt again.
 - Your account credentials have been saved in your Let's Encrypt
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Let's
   Encrypt so making regular backups of this folder is ideal.
 - If like Let's Encrypt, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le

```

证书位置：/etc/letsencrypt/live/example.com/fullchain.pem

过期时间：2016-03-15

至于过期时间为什么这么短则是出于安全考虑。不宜使用同一个证书太长时间。

##### 证书文件：

上一步成功后，会有如下4个文件生成。


    cert.pem: 证书
    chain.pem: The Let's Encrypt chain certificate（不知道什么鬼）
    fullchain.pem: cert.pem and chain.pem combined
    privkey.pem: 证书私钥
    
```
sudo ls /etc/letsencrypt/live/your_domain_name
```

### STEP 2－配置Web服务器(Nginx)

证书生成完毕，现在可以配置Web服务器了。

编辑配置文件：

```
sudo vi /etc/nginx/sites-available/default
```

在server里，增加如下代码：

```
listen 443 ssl;
server_name example.com;
ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
```

如果之前是绑定的80端口，直接改为443即可，后边增加 ssl 。

如果想使用最安全的SSL协议，增加如下代码：

```
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_prefer_server_ciphers on;
ssl_ciphers AES256+EECDH:AES256+EDH:!aNULL;
```

最后，在server区块后再增加一段，用于从HTTP跳转到HTTPS。

```
server {
    listen 80;
    server_name example.com;
    rewrite ^/(.*) https://example.com/$1 permanent;
}

```

保存，退出。

重启Nginx服务器。

```
sudo nginx -s reopen
```

现在已经可以通过HTTPS访问网站了。

原文中还有设置自动生成证书的部分，出于懒的原因此处不再写。

参考资料：
[How To Secure Nginx with Let's Encrypt on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-14-04)