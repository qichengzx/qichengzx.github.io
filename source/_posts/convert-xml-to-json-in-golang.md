title: Golang XMl 转 JSON
categories: golang
date: 2016-09-16 11:28:20
tags:  [go]

---

![](/images/golang.png)

#### 起因

某个上古时代的API，依然在返回XML格式的数据，更奇葩的是，GBK格式的。

用Go顺利的写到了发送数据，接收数据，然后取值有点麻烦啊。。。。

各种Google后，终于解决，但是不保证是唯一，正确，最合适的答案。

#### 说在前边

本文假设要解析的XMl数据为：

```xml
<?xml version="1.0" encoding="GBK" ?>
<response>
    <status>200</status>
</response>
```

要解决的问题是取出“200”这个状态值。

#### 导入包

解析XML使用了"encoding/xml"这个包。

所以先导入这个包。

```go
import "encoding/xml"
```

#### 定义struct

定义一个自定义类型的Response

```go
type Response struct {
    Status int `xml:"status" json:"status"`
}
```

定义一个Response类型的变量

```go
var result Response
```

#### 偷懒转格式

因为"encoding/xml"不支持GBK格式的XML，而返回的内容又固定标明了编码是GBK，所以这里偷懒，直接把GBK替换成UTF-8，本例中不影响结果。

```go
xmlstr := `?xml version="1.0" encoding="GBK" ?>
<response>
    <status>200</status>
</response>
`
xmlstr = strings.Replace(xmlstr, "GBK", "UTF-8", -1)
```

使用strings包，替换“GBK”，相信根据参数顺序能看出各个参数的意义，最后一个参数：-1，为替换全部，即字符串中所有出现的第二个参数全部替换。

#### 解析，转换，取值

使用encoding/jon，go-simplejson包

```go
//解析XML
err := xml.Unmarshal([]byte(xmlstr), &result)

if nil != err {
  log.Fatal(err)
}
log.Printf("XML:%v \n", result) 

//转换成JSON
res, err := json.Marshal(result)

if nil != err {
  log.Fatal(err)
}
log.Printf("JSON:%s \n", res)

js, err := simplejson.NewJson([]byte(res))

if nil != err {
  log.Fatal(err)
}
status, err := js.Get("status").Int()

log.Printf("STATUS:%v \n", status)
```

以上是本人在处理XML 转 JSON 时的解决办法，应该还有更简单更合适的方案。仅供参考。

#### 完整代码：

```go
package main
import (
    "encoding/xml"
    "encoding/json"
    "log"
    "strings"
    simplejson "github.com/bitly/go-simplejson"
)
type Response struct {
    Status int `xml:"status" json:"status"`
}

func main() {
    
    var result Response

    //多行字符串，使用反引号`
    xmlstr := `<?xml version="1.0" encoding="GBK" ?>
<response>
    <status>200</status>
</response>`

    xmlstr = strings.Replace(xmlstr, "GBK", "UTF-8", -1)

    err := xml.Unmarshal([]byte(xmlstr), &result)
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("XML:%v",result)

    r, err := json.Marshal(result)
    if nil != err {
        log.Fatal(err)
    }

    log.Printf("JSON:%s", r)

    js, err := simplejson.NewJson([]byte(r))

    if nil != err {
        log.Fatal(err)
    }
    status, err := js.Get("status").Int()
    log.Printf("VALUE:%v",status)

}
```



参考文章：

[标准库—XML处理（一）](http://blog.studygolang.com/2012/12/%e6%a0%87%e5%87%86%e5%ba%93-xml%e5%a4%84%e7%90%86%ef%bc%88%e4%b8%80%ef%bc%89/)

[https://play.golang.org/p/7HNLEUnX-m](https://play.golang.org/p/7HNLEUnX-m)

[https://play.golang.org/p/m99B12RaLe](https://play.golang.org/p/m99B12RaLe)

[XML处理](https://astaxie.gitbooks.io/build-web-application-with-golang/content/zh/07.1.html)