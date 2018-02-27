title: Nginx HTTP/2 Server Push
categories: nginx
date: 2018-02-27 19:41:32
tags:  [nginx]

---

Nginx 在最新的 [1.13.9](http://nginx.org/en/download.html) 版本中，增加了对 HTTP/2 Server Push 的支持，以下就简单介绍下如何使用。

以下内容主要来自 [Introducing HTTP/2 Server Push with NGINX 1.13.9](https://www.nginx.com/blog/nginx-1-13-9-http2-server-push/)，进行了简单的整理。

### 推送指定资源

首先，在 Nginx 中进行配置：

```nginx
server {
    #开启HTTP/2
    listen 443 ssl http2;

    ssl_certificate ssl/certificate.pem;
    ssl_certificate_key ssl/key.pem;

    root /var/www/html;

    # 当请求 demo.html 时，推送 /style.css, /image1.jpg, /image2.jpg
    location = /demo.html {
        http2_push /style.css;
        http2_push /image1.jpg;
        http2_push /image2.jpg;
    }
}
```

可以通过 Chrome Developer Tools 查看效果。在 Network 中，可以看到 demo.html 的请求和推送到客户端的内容。

如下图所示，可以看到 style.css，image1.jpg，image2.jpg。它们都是 demo.html 请求的一部分。

![](/images/http2-server-push-chrome-dev-tools.png)

### 自动推送

以上是在页面加载时推送指定资源的例子，但是很多情况下，并不能指定要推送哪些资源，因此，Nginx 还支持通过 http2_push_preload 指令，自动分析响应头中的 Link header，来自动推送这些资源。

```nginx
server {
    #开启HTTP/2
    listen 443 ssl http2;

    ssl_certificate ssl/certificate.pem;
    ssl_certificate_key ssl/key.pem;

    root /var/www/html;

    location = /myapp {
        proxy_pass http://upstream;
        http2_push_preload on;
    }
}
```

通过如上的配置，当 upstream 返回的响应头中包含 Link header 时，

```
Link: </style.css>; as=style; rel=preload
```

Nginx 即会开启一个推送， 内容则是 /style.css ，Link header 的路径必须是绝对路径。

如果想要推送多个资源，可以添加多个 Link header ， 或更直接一些，把所有资源添加到一个 Link header 中。

```
Link: </style.css>; as=style; rel=preload, </favicon.ico>; as=image; rel=preload
```

如果不想让 Nginx 推送某个资源，为该 header 添加一个 nopush 参数即可。

```
Link: </nginx.png>; as=image; rel=preload; nopush
```

### 有选择的推送

由于 HTTP/2 规范并没有解决是否要推送哪些资源的问题，但是比较合理的方式是知道需要推送哪些资源，并且客户端没有缓存过这些资源，才进行推送。

所以 Nginx 支持了添加条件，只在符合条件时才进行推送

当客户端请求时包含了 cookie ，Nginx 将只会推送一次资源。

```nginx
server {
    listen 443 ssl http2 default_server;

    ssl_certificate ssl/certificate.pem;
    ssl_certificate_key ssl/key.pem;


    root /var/www/html;
    http2_push_preload on;

    location = /demo.html {
        add_header Set-Cookie "session=1";
        add_header Link $resources;
    }
}


map $http_cookie $resources {
    "~*session=1" "";
    default "</style.css>; as=style; rel=preload, </image1.jpg>; as=image; rel=preload,
             </image2.jpg>; as=style; rel=preload";
}
```

相关链接:

[在Go中使用 HTTP/2 Server Push](https://www.qichengzx.com/2017/07/02/HTTP2-Server-Push.html)

参考资料:

[Server Push (HTTP/2)](https://w3c.github.io/preload/#server-push-http-2)
