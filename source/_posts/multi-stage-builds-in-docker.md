
Docker在17.05引入了[多阶段构建](https://docs.docker.com/engine/userguide/eng-image/multistage-build/)的功能，就是将之前需要多次运行build的Dockerfile，现在可以写到一个里面，只build一次，就可以达到实际需要的目的。

比如之前说过的，做一个Beego的Docker镜像，如果使用官方的镜像运行，占用空间稍微有点大了，虽然比起Ubuntu，CentOS动辄5-600M还好一些，但是再跟只有10几M，甚至几M的相比，还是太大了。

比如，可以这样写：

```shell
FROM golang:1.9.2 as builder

WORKDIR /go/src/app
COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build -x -v -ldflags '-w -s' -a -o app .

FROM alpine:latest

WORKDIR /go/src/app


#fix DB连接缺失timezone文件的问题
COPY --from=builder /usr/local/go/lib/time/zoneinfo.zip /usr/local/go/lib/time/zoneinfo.zip
COPY --from=builder /go/src/app/app ./app

EXPOSE 8080

CMD ["./app"]

```

