title: Docker多阶段构建
categories: golang
date: 2017-11-27 20:38:41
tags:  [docker]

---

![](/images/docker_twitter_share.png)

Docker在[17.05](https://github.com/moby/moby/releases/tag/v17.05.0-ce)引入了[多阶段构建](https://docs.docker.com/engine/userguide/eng-image/multistage-build/)的功能，就是将之前需要多次运行build的Dockerfile，现在可以写到一个里面，只build一次，就可以达到同样的效果。

另外，实际应用中，还可以通过这样，非常简单的将最后可执行文件放入极小的镜像中使用。

比如之前说过的，做一个Beego的Docker镜像，如果使用官方的镜像运行，占用空间稍微有点大了，虽然比起Ubuntu，CentOS动辄5-600M还好一些，但是再跟只有10几M，甚至几M的相比，还是太大了。

下面通过一个Dockerfile来了解一下：

```shell
#指定构建镜像
FROM golang:1.9.2 as builder

#指定工作目录
WORKDIR /go/src/app
#将当前项目文件copy到够姜镜像的工作目录中
COPY . .

#在构建镜像中执行go build，这里指定了构建的目标平台，具体的构建命令针对具体情况修改即可
#也可简单的 go build 即可
#另外需要注意依赖包的问题
RUN CGO_ENABLED=0 GOOS=linux go build -x -v -ldflags '-w -s' -a -o app .

#使用alpine作为运行的镜像
FROM alpine:latest

#指定工作目录
WORKDIR /go/src/app

#如果程序中涉及到需要连接DB，并且需要指定时区，需要copy时区文件，为了省事，直接从构建镜像中复制了，可以正常使用
COPY --from=builder /usr/local/go/lib/time/zoneinfo.zip /usr/local/go/lib/time/zoneinfo.zip
#将构建镜像中build完成的可执行文件copy到工作目录
COPY --from=builder /go/src/app/app ./app

EXPOSE 8080

CMD ["./app"]

```

以上就是使用多阶段构建的Dockerfile了，非常明了，只需要docker build就可以了。

然而目前公司的测试环境还没升级到17.05，并不能使用此功能。