title: 用Docker部署Golang Beego框架应用
categories: golang
date: 2016-10-21 10:02:02
tags: [go]

---

Docker是什么就不说了。
Golang是什么也不说了。
Beego是什么就更不用说了。

最近Beego项目完成，研究怎么部署。因为Docker部署起来更简单更快速，所以就说下怎么在docker里部署beego应用。

#### 写在前面

假设你的应用路径为 /go/app；
假设已配置好docker的相关东西。
假设使用 [godep](https://github.com/tools/godep) 作为依赖管理工具
示例中开放端口为80，需要与app.conf中的端口一致，可以自行修改。

#### 配置

在 /go/app 目录新建Dockerfile。

```
FROM golang:1.7.1-alpine

MAINTAINER youremail <youremail@xxx.com>

RUN apk add --update go git

ADD ./ /go/src/app

RUN cd /go/src/app \
	&& go get github.com/astaxie/beego \
	&& go get github.com/tools/godep \
	&& godep update -goversion \
	&& godep get \
	&& godep save \
	&& go build

EXPOSE 80

EXTRYPOINT /go/src/app/app
```

本例使用 golang:1.7.1-alpine 作为基础镜像。golang的所有镜像见[这里](https://hub.docker.com/_/golang/)。

#### 构建
```
docker build -t app .
```

#### 运行 

```
docker run -d -p 8080:80 app
```

#### 访问

使用nginx反向代理访问docker中的go应用。

```
server {
    listen       80;
    server_name  app.com;

    charset utf-8;
    access_log  logs/app.access.log;

    location / {
        try_files /_not_exists_ @backend;
    }
    if (!-e $request_filename) {
        return 404;
    }

    location @backend {
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Host            $http_host;

        proxy_pass http://192.168.99.100:8080; // 192.168.99.100为docker machine的ip,8080为 docker run 时指定的本地端口。
    }
}
```

#### 相关资料

[如何使用Docker快速部署go-web应用程序](https://yq.aliyun.com/articles/57247)
[Deploying Go servers with Docker](https://blog.golang.org/docker)
[如何使用Docker部署Go Web应用程序](http://www.infoq.com/cn/articles/how-to-deploy-a-go-web-application-with-docker?utm_source=infoq&utm_medium=related_content_link&utm_campaign=relatedContent_articles_clk)
[The Easiest Way to Develop with Go — Introducing a Docker Based Go Tool](https://www.iron.io/the-easiest-way-to-develop-with-go%E2%80%8A-%E2%80%8Aintroducing-a-docker-based-go-tool/)
[How To Deploy a Go Web Application with Docker](https://semaphoreci.com/community/tutorials/how-to-deploy-a-go-web-application-with-docker)
[nginx 部署](https://beego.me/docs/deploy/nginx.md)