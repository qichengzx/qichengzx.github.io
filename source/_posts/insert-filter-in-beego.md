title: Beego中使用过滤器
categories: golang
date: 2017-03-06 08:23:47
tags:  [go]

---

为了方便调试和排错，决定在现有的beego程序里加上requestID。

查了些资料发现写的并不是特别清楚和详细，在此总结一下，也算是加深下印象。

astaxie说可以用过滤器实现，就是在Beego运行时在特定的步骤前加入。而由于我的需求比较简单，就选在了BeforeRouter。

在main.go中:

```Go
import "github.com/astaxie/beego/context"
import "github.com/satori/go.uuid"

```

在main函数中加入:

```Go
var FilterRequestID = func(ctx *context.Context) {
	requestId := uuid.NewV4().String()
	ctx.Input.SetData("requestId", requestId)
}

beego.InsertFilter("/*", beego.BeforeRouter, FilterRequestID)

```

在需要使用的地方，如

```Go
// @router /requestid [get]
func (this *MyController) Requestid() {
	//读取requestId
	rid := this.Ctx.Input.GetData("requestId").(string)

	fmt.Println("requestId:",rid)
}
```

或者

```Go
func (m *MyController) Requestid() {
	rid := m.Ctx.Input.GetData("requestId").(string)

	fmt.Println("requestId:",rid)
}
```

其实很简单，但是文档和查到的资料中都没有明确的说需要引用 "github.com/astaxie/beego/context"，导致写的时候浪费了一些时间。

参考资料:

[过滤器](https://beego.me/docs/mvc/controller/filter.md)
[beego log中增加request id的一种方式](https://gocn.io/article/95)