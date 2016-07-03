docker配置PHP(beta)



### Docker

 Docker大火，一个新的小项目也要用一下，前前后后看了一段时间也折腾了一段时间，算是对docker配置PHP环境有了个初步的认识，但是难免会有些疏漏或错误。

仅作为记录。

[docker hub上的PHP仓库](https://hub.docker.com/_/php/)，包含了多个版本的PHP，如fpm，cli，Apache等，以下示例使用7-fpm-alpine。



docker构建一个image，一般会使用Dockerfile，这个文件里指定了FOM，指定了需要安装