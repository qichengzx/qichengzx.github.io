title: Mac brew php7.1环境下安装Yaf
categories: php
date: 2016-11-02 17:57:38
tags:  [php]

---

![](/images/php/think201-emergence-of-php7.jpg)

开发机一直使用brew来安装PHP及其他的环境，今天把PHP升到7.1，由于7.1版本下还没有[yaf](http://www.laruence.com/manual/)的源，所以无法使用brew安装，只能编译安装了。

首先下载yaf，解压，进入目录。

```
git clone git@github.com:laruence/yaf.git

$(brew --prefix homebrew/php/php71)/bin/phpize

./configure --with-php-config=$(brew --prefix homebrew/php/php71)/bin/php-config

make && make install

make test

```

$(brew --prefix homebrew/php/php71) 即 brew info php71结果中的path值。

由于brew安装PHP会在php.ini同级目录创建conf.d目录，并把扩展的配置文件写在这里，一目了然知道都安装了哪些扩展，所以也以同样方式在此目录创建ext-yaf.ini。

make install 后会显示，具体路径可能会不一样。

```
Installing shared extensions:     /usr/local/Cellar/php71/7.1.0-rc.5_9/lib/php/extensions/no-debug-non-zts-20160303/
```
这个目录即扩展.so的存放目录。下边会用到。

```
[yaf]
extension="/usr/local/opt/php71/lib/php/extensions/no-debug-non-zts-20160303/yaf.so"
yaf.environ="dev"
;yaf.use_namespace = 1
```

至此，重启php-fpm就可以了。

#### 
图片来自：[Emergence of PHP7](https://think201.com/blog/2016/emergence-of-php7/)