title: 在Go中使用 HTTP/2 Server Push
categories: golang
date: 2017-07-02 00:09:13
tags:  [go]

---

#### 写在前面

本文来自[Golang的官方博客](https://blog.golang.org/h2push)，由 Jaana Burcu Dogan, Tom Bergan 发表于2017年3月24日。

近来看到此篇，觉得不错，非常适合用来学习Go中的 Server Push ，于是决定翻译一下，水平有限，如有不足不恰当的地方还请提出宝贵意见。

#### 序

HTTP/2旨在解决HTTP/1.x的很多问题。现代网页通常包含很多资源：HTML，css，js，图片等等。在 HTTP/1.x 中，必须明确地请求这些资源。这是一个缓慢的过程。浏览器必须从获取HTML开始，然后在解析页面时获取更多资源。由于服务器必须等待浏览器发起请求，网络通常处于空闲，没有充分利用。

为了改善延迟，HTTP/2引入了 Server Push ，这允许服务器在明确的请求之前将资源推送到浏览器。服务器通常会知道一个页面所需要的额外的资源，并且可以在响应初始请求时开始推送这些资源。这就允许服务器充分利用空闲的网络来改善加载时间。

![server push示意](/images/go/serverpush.svg)

在协议层，HTTP/2 Server Push 由 PUSH_PROMISE 帧发起。PUSH_PROMISE 表明了服务器向客户端推送资源的意图。一旦浏览器接收到PUSH_PROMISE，它就会知道服务器会推送资源。如果浏览器后来发现需要这个资源，它会等待推送完成，而不是发起一个新的请求。这减少了浏览器在网络上等待的时间。

#### net/http 包中的 Server Push

Go1.8 引入了 http.Server 对 push 响应的支持，如果运行的服务器是 HTTP/2 服务器，并且请求连接使用了 HTTP/2，则可以使用此功能。在任何HTTP处理程序中，可以通过检查 http.ResponseWriter 是否实现了 http.Pusher 接口来判断是否支持 Server Push。

例如，如果服务器知道一个页面中包含 app.js，处理程序可以初始化一个 http.Pusher。

```go
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
  if push, ok := w.(http.Pusher); ok {
    //支持Push
    if err := pusher.Push("/app.js", nil); err != nil {
      log.Printf("Failed to push: %v", err)
    }
  }
  // ...
})
```

Push会为 app.js 创建一个 “合成请求”，将该请求合成到 PUSH_PROMISE 帧中，然后将请求转发到服务器的请求处理程序，请求处理程序会生成响应。Push的第二个参数指定了包含在 PUSH_PROMISE 中的附加header头，例如，如果对 app.js 的响应在 Accept-Encoding 上不同，则 PUSH_PROMISE 应包含 Accept-Encoding 的值。

```go
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
  if pusher, ok := w.(http.Pusher); ok {
    //支持Push
    options := &http.PushOptions{
      Header: http.Header{
        "Accept-Encoding": r.Header["Accept-Encoding"],
      },
    }
    if err := pusher.Pusher("/app.js", options); err != nil {
      log.Printf("Failed to push: %v", err)
    }
  }
  // ...
})
```

完整示例见：

```shell
$ go get golang.org/x/blog/content/h2push/server
```

启动服务，然后打开 [https://localhost:8080](https://localhost:8080)，浏览器开发工具应该会显示服务器推送了 app.js 和 style.css 。

![网络请求耗时](/images/go/networktimeline.png)

#### 在响应前开始Push

在发送响应之前调用Push是个好主意，否则可能会意外产生重复的响应，例如，假设下边是HTML响应的一部分：

```html
<html>
<head>
	<link rel="stylesheet" href="a.cs">...	
```

然后调用 Push("a.css", nil) ，浏览器可能会在接收到 PUSH_PROMISE 之前就开始解析这段 HTM L了，这种情况下，浏览器除了接收到 PUSH_PROMISE 之外，还会发起一个 a.css 的请求，那么服务器就会为 a.css 生成两个请求。而在写入响应之前调用PUSH则避免了这种可能性。

#### 何时使用Server Push

应该考虑在任何网络连接空闲时使用 Server Push 。刚完成为 web app 发送 HTML ？不要浪费时间等待请求，开始推送客户端需要用到的资源，你有没有过将资源嵌入到 HTML 文件中以减少延迟？替换掉内联，尝试使用推送。重定向是另一个使用推送的好时机，因为客户端在这个过程中几乎把时间全浪费在请求的往返上。有很多情况适合使用 Push ，我们才刚刚开始。

如果没有提到以下几个注意事项，将是我们的失职。

​	首先，只能推送当前服务器上的资源-这意味着无法推送托管在第三方服务器或CDN上的资源。

​	第二，除非能确定客户端确实需要，否则不要推送，这样会浪费带宽。当浏览器已经缓存了某些资源时， 必须要避免推送。

​	第三，天真的把所有资源推送到页面会使性能更糟糕。

以下链接可以作为补充阅读：

- [HTTP/2 Push: The Details](https://calendar.perfplanet.com/2016/http2-push-the-details/)
- [Innovating with HTTP/2 Server Push](https://www.igvita.com/2013/06/12/innovating-with-http-2.0-server-push/)
- [Cache-Aware Server Push in H2O](https://github.com/h2o/h2o/issues/421)
- [The PRPL Pattern](https://developers.google.com/web/fundamentals/performance/prpl-pattern/)
- [Rules of Thumb for HTTP/2 Push](https://docs.google.com/document/d/1K0NykTXBbbbTlv60t5MyJvXjqKGsCVNYHyLEXIxYMv0)
- [Server Push in the HTTP/2 spec](https://tools.ietf.org/html/rfc7540#section-8.2)

#### 结尾

Go1.8 标准库为 HTTP/2 Server Push 提供了开箱即用的支持，为优化 Web 应用程序提供更多灵活性。

转到[HTTP/2 Server Push演示页面](https://http2.golang.org/serverpush)查看实际效果。

#### 一些资料

[HTTP/2 简介](https://developers.google.com/web/fundamentals/performance/http2/?hl=zh-cn)

HTTP/2 Server Push 详解：

原文：

[A Comprehensive Guide To HTTP/2 Server Push](https://www.smashingmagazine.com/2017/04/guide-http2-server-push/)

译文：

[HTTP/2 Server Push 详解（上）](http://www.alloyteam.com/2017/04/guide-http2-server-push-part1/)

[HTTP/2 Server Push 详解（下）](http://www.alloyteam.com/2017/04/guide-http2-server-push-part2/)