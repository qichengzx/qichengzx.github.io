title: Go struct 转 map 使用自定义标签
categories: golang
date: 2018-02-11 21:06:00
tags:  [go]

---

今天工作遇到一个问题，之前将 struct 转 map 的时候，没有注意 field 大小写的问题，具体的说，是没有注意 field name 与实际需要的 name 的区别，其实就是需要自定义转为 map 之后的name，今天发现问题后，看了下引用包的源码，发现是可以[自定义标签](https://godoc.org/github.com/fatih/structs#Struct.Map)的，就跟 struct 转 JSON 一样。

代码也很简单，加上 \``structs:"name"`\`  即可。

```go
package main

import (
	"fmt"
	"github.com/fatih/structs"
)

type Server struct {
	Name string `structs:"server_name"`
	ID   int    `structs:"server_id"`
}

func main() {
	server := &Server{
		Name: "gopher",
		ID:   123456,
	}

	fmt.Printf("struct : %v\n", server) //struct : &{gopher 123456}

	serverMap := structs.Map(server)

	fmt.Printf("map : %v\n", serverMap) //map : map[server_name:gopher server_id:123456]

```

这样就可以拿到自定义key的map了。