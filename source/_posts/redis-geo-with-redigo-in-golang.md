title: Go语言使用redigo操作GEO
categories: golang
date: 2017-09-17 19:45:29
tags:  [go,redis,redigo,geo]
---

周五的时候群里有人遇到在Go中使用Redis GEO的问题，顺手搜了下解决办法。发现还挺简单的，周末无事，写下来记一下。

GEO是在Redis 3.2加入的功能，手册[见此](http://redisdoc.com/geo/index.html)

本示例中只演示GEOADD和GEOPOS功能。

[redigo](https://github.com/garyburd/redigo/tree/master/redis)这个客户端还挺好用的，但是个人觉得略有不足的是文档不全（指的是中文的文档），但是好在可以看源码解决一些不太清楚的问题。

#### 0.连接Redis

首先习惯性的创建了连接池，嗯，连接池。

```Go
func init() {
	RedisClient = &redis.Pool{
		MaxIdle:   MaxIdle,
		MaxActive: MaxActive,

		IdleTimeout: 60 * time.Second,

		Dial: func() (redis.Conn, error) {
			c, err := redis.Dial("tcp", RedisHost)

			if err != nil {
				return nil, err
			}

			/*if _, err := c.Do("AUTH", RedisPwd); err != nil {
				c.Close()
				return nil, err
			}*/
			c.Do("SELECT", RedisDb)

			return c, nil
		},
	}
}
```

#### 1.添加值

为了简单起见，创建了一个方法用于向传入的key中写入name的坐标点。

直接返回错误。

```Go
func push(key, name string, lat, lng float64) error {
	rc := RedisClient.Get()
	defer rc.Close()

	_, err := rc.Do("GEOADD", key, lng, lat, name)
	return err
}

```

#### 2.取回值

同样，创建一个方法，在Redis中取回key中name的坐标点的值。

但是注意，这里在执行完 ``GEOPOS`` 后，调用 redigo 包中的 ``Positions`` 方法把返回结果转成 ``float64`` 的数组。

然后返回这个数组和错误

```Go
func get(key, name string) ([]*[2]float64, error) {
	rc := RedisClient.Get()
	defer rc.Close()

	res, err := redis.Positions(rc.Do("GEOPOS", key, name))

	return res, err
}
```

#### 3.main方法中的简单代码

```Go 
func main() {
	key := "citylist"
	name := "beijing"
	lat := 39.9329
	lng := 116.280316
	err := push(key, name, lat, lng)
	if err != nil {
		panic(err)
	}
	fmt.Println("坐标写入完成")
	fmt.Println("获取刚刚写入的值")

	res, err := get(key, name)
	if err != nil {
		panic(err)
	}
	fmt.Println(res[0][0], res[0][1])
}
```

就这么简单。

import部分和定义的几个全局变量就不用写了，
