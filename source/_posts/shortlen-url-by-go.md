title: Go语言写的一个短网址服务
categories: golang
date: 2016-12-27 21:27:34
tags:  [go,短网址]
---

![](/images/go/best-url-shortener-to-make-earn-money.png)
<center>题图来自[http://www.dwtricks.com/](http://www.dwtricks.com/2015/05/best-10-url-shortener-networks-to-earn-money-2015.html/)</center>

"缩址，又称短址、短网址、网址缩短、缩短网址、URL缩短等，指的是一种互联网上的技术与服务。此服务可以提供一个非常短小的URL以代替原来的可能较长的URL，将长的URL地址缩短。
用户访问缩短后的URL时，通常将会重定向到原来的URL。"

-- Wikipedia


虽然短网址早已不再那么受广泛关注。但是不妨拿来练手。

根据公开可以搜索到的资料，短网址一般是将一个ID转换到一串字母，生成短的网址用于传播，实际访问会重定向到原网址。如上所述。

那么使用Go来写这个有什么优势呢，优势之一当然是，Go部署简单，只需要copy执行文件即可。执行速度也快，甚至连HTTP服务器都不需要。

##### 下边就边写边说明。

```go
package main

import (
	"fmt"
	"strings"
	"time"
	"net/http"
	"database/sql"

	"github.com/gin-gonic/gin"

	"github.com/garyburd/redigo/redis"
	_ "github.com/go-sql-driver/mysql"

	"github.com/speps/go-hashids"
)
```

###### 定义hashid包需要的salt，即生成字符串的最短位数。

```go
const (
	hdSalt        = "mysalt"
	hdMinLength   = 5
	defaultDomain = "http://localhost:8000/"
)
```

###### 定义redis和MySQL的配置信息

```go
var (
	RedisClient *redis.Pool
	RedisHost   = "127.0.0.1:6379"
	RedisDb     = 0
	RedisPwd    = ""

	db      *sql.DB
	DB_HOST = "tcp(127.0.0.1:3306)"
	DB_NAME = "short"
	DB_USER = "root"
	DB_PASS = ""
)
```

##### main函数，首先连接redis和MySQL。定义如下路由：

- 访问首页
- 访问hash
- 访问短网址信息页
- 生成短网址接口

熟悉的朋友应该都知道，访问短网址服务的首页一般会跳转到一个固定的网址，比如渣浪微博会跳转到微博首页，Twitter则是给出“Twitter uses the t.co domain as part of a service to protect users from harmful activity”的提示。这里我们也让它跳转到一个指定的网页。

最后，以8080端口运行，实际线上会使用80端口，可以自行修改。

```go
func main() {
	initRedis()
	initMysql()

	gin.SetMode(gin.DebugMode)
	r := gin.Default()

	r.GET("/", func(c *gin.Context) {
		//http code can be StatusFound or StatusMovedPermanently 
		c.Redirect(http.StatusFound, defaultDomain)
	})
	r.GET("/:hash", expandUrl)
	r.GET("/:hash/info", expandUrlApi)
	r.POST("/short", shortUrl)

	r.Run(":8000")
}
```

##### 连接redis和MySQL

```go
func initRedis() {
	// 建立连接池
	RedisClient = &redis.Pool{
		MaxIdle:     1,
		MaxActive:   10,
		IdleTimeout: 180 * time.Second,
		Dial: func() (redis.Conn, error) {
			c, err := redis.Dial("tcp", RedisHost)
			if err != nil {
				return nil, err
			}
			if _, err := c.Do("AUTH", RedisPwd); err != nil {
				c.Close()
				return nil, err
			}
			c.Do("SELECT", RedisDb)
			return c, nil
		},
	}
}

func initMysql() {
	dsn := DB_USER + ":" + DB_PASS + "@" + DB_HOST + "/" + DB_NAME + "?charset=utf8"
	db, _ = sql.Open("mysql", dsn)
	db.SetMaxOpenConns(5)
	db.SetMaxIdleConns(20)
	db.Ping()
}
```

##### 生成短网址的接口函数。

根据传入的URL参数，进行简单的验证后，写入数据库。根据写入后生成的ID，再生成一个字符串，然后返回给调用方。

```go
func shortUrl(c *gin.Context) {
	longUrl := c.PostForm("url")

	if longUrl == "" {
		c.JSON(200, gin.H{
			"status":  500,
			"message": "请传入网址",
		})
		return
	}

	if !strings.HasPrefix(longUrl, "http") {
		longUrl = "http://" + longUrl
	}

	if hash, ok := insert(longUrl); ok {
		c.JSON(200, gin.H{
			"status":  200,
			"message": "ok",
			"short":   defaultDomain + hash,
		})
	}
}
```

##### 根据HASH解析并跳转到对应的长URL，不存在则跳转到默认地址

```go
func expandUrl(c *gin.Context) {
	hash := c.Param("hash")

	if url, ok := findByHash(hash); ok {
		c.Redirect(http.StatusMovedPermanently, url)
	}
	// 注意:
	// 	实际中，此应用的运行域名可能与默认域名不同，如a.com运行此程序，默认域名为b.com
	// 	当访问一个不存在的HASH或a.com时，可以跳转到任意域名，即defaultDomain
	c.Redirect(http.StatusMovedPermanently, defaultDomain)
}
```

##### 根据HASH在redis中查找并返回结果，不存在则返回404状态

```go
func expandUrlApi(c *gin.Context) {
	hash := c.Param("hash")

	if url, ok := findByHash(hash); ok {
		c.JSON(200, gin.H{
			"status":  200,
			"message": "ok",
			"data":    url,
		})
		return
	}

	// 此处可以尝试在MySQL中再次查询
	c.JSON(200, gin.H{
		"status":  404,
		"message": "url of hash is not exist",
	})
}
```

##### 将ID转换成对应的HASH值，hdSalt与hdMinLength 会影响生成结果，确定后不要改动

```go
func shortenURL(id int) string {
	hd := hashids.NewData()
	hd.Salt = hdSalt
	hd.MinLength = hdMinLength

	h := hashids.NewWithData(hd)
	e, _ := h.Encode([]int{id})

	return e
}
```

##### 根据HASH解析出对应的ID值, hdSalt与hdMinLength 会影响生成结果，确定后不要改动

```go
func expand(hash string) int {
	hd := hashids.NewData()
	hd.Salt = hdSalt
	hd.MinLength = hdMinLength

	h := hashids.NewWithData(hd)
	d, _ := h.DecodeWithError(hash)

	return d[0]
}
```

##### 数据库中根据ID查找

```go
func find(id int) (string, bool) {
	var url string
	err := db.QueryRow("SELECT url FROM url WHERE id = ?", id).Scan(&url)
	if err == nil {
		return url, true
	} else {
		return "", false
	}
}
```

##### 在redis中根据HASH查找

```go
func findByHash(h string) (string, bool) {
	rc := RedisClient.Get()

	defer rc.Close()
	url, _ := redis.String(rc.Do("GET", "URL:"+h))

	if url != "" {
		return url, true
	}

	id := expand(h)
	if urldb, ok := find(id); ok {
		return urldb, true
	}

	return "", false
}
```

##### 将长网址插入到数据库中，并把返回的ID生成HASH和长网址存入redis

```go
func insert(url string) (string, bool) {
	stmt, _ := db.Prepare(`INSERT INTO url (url) values (?)`)
	res, err := stmt.Exec(url)
	checkErr(err)

	id, _ := res.LastInsertId()

	rc := RedisClient.Get()
	defer rc.Close()

	hash := shortenURL(int(id))
	rc.Do("SET", "URL:"+hash, url)

	return hash, true
}
```

#### 打印方法，和检查错误的方法

```go
func Log(v ...interface{}) {
	fmt.Println(v...)
}

func checkErr(err error) {
	if err != nil {
		panic(err)
	}
}
```

有些地方还需修改，就算是抛砖引玉吧。

感谢[hashids](http://hashids.org/)

#### Github地址 ： [shortme](https://github.com/qichengzx/shortme)

相关资料：

[URL Toolbox: 90+ URL Shortening Services](http://mashable.com/2008/01/08/url-shortening-services/#CgEzOrfnzPqb)
[TinyURL](http://tinyurl.com/)
