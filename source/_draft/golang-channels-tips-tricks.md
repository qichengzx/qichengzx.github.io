title: Go 的缓冲 CHANNEL
date: 2018-02-07 17:43:49
tags:  [Go]
---

Channels 和 goroutines 是 Go 基于 CSP-based 并发机制的核心。特别是当 CHANNEL 有缓冲区的时候，通常用作 “生产者-消费者” 队列使用。

#### 有缓冲 CHANNELS = 队列 

缓冲CHANNEL是有固定容量的先进先出（FIFO）队列，容量在创建时队列时指定，队列运行时不能修改大小。

```
queue := make(chan Item, 10) // 容量为10的队列
```

队列中的

https://www.rapidloop.com/blog/golang-channels-tips-tricks.html