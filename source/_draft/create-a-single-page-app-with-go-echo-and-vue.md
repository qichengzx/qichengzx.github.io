title: Create a Single Page App With Go, Echo and Vue
date: 2016-11-30 17:28:55
tags:  [go,vue]
---

本教程中我们将会创建一个"todo"应用。完成后可以实现创建任务，展示最新创建的任务和删除它们。

此程序后端使用Go语言。Go由Google开发。虽然不是最流行的语言，但是正在逐步得到认可。Go非常轻量级，易于学习，运行快。此教程假设你已经对于这门语言有了一些了解，并且已经安装和配置好了开发环境。

我们将会使用Echo框架，Echo框架相当于PHP语言的 Slim PHP 或 Lumen 框架，你应该有点熟悉使用微框架和使用路由处理http请求的概念。

任务数据将会储存在SQLite数据库中，SQLite是一个轻量级的可替代MySQL或PostgreSQL的数据库。数据会存在一个独立的文件中，与应用在同一目录而不是存在服务器上。

最后，前端使用HTML5和流行的 VueJS JavaScript 框架，需要对 VueJS 有一定的了解。

应用分成4个基本点。
首先设置路由和数据库，
处理路由连接，
Task模型用SQLite做持久化存储，
最后应用会有 HTML5+VueJS 做的一个首页。

## 路由和数据库

在入口文件中会引入几个有用的包。"database/sql"是Go标准包，但是Echo和SQLite需要从Github下载。

```bash
$ go get github.com/labstack/echo
$ go get github.com/mattn/go-sqlite3
```

然后创建目录。

```
$ cd $GOPATH/src
$ mkdir go-echo-vue && cd go-echo-vue
```

开始写路由，在go-echo-vue目录创建一个文件并命名为"todo.go"，然后引入 Echo 框架。

```
// todo.go
package main

import (
	"github.com/labstack/echo"
	"github.com/labstack/echo/engine/standard"
)
```

下一步，创建go程序必需的 "main" 方法。 

```
// todo.go
func main() { }
```

为了使前端的VueJS可以和后端通信，创建任务，需要设置一些基本的路由。第一件事就是实例化一个 Echo 。然后使用內建方法定义几个路由。如果使用过其他框架，应该会熟悉这个概念。

路由使用一个正则作为第一个参数，然后使用一个处理方法作为第二个参数。在此教程中必须使用 Echo.HandlerFunc 接口。

现在可以在 "main" 方法中创建几个给前端通信使用的路由了。

```
// todo.go
func main() {
    // Create a new instance of Echo
    e := echo.New()

    e.GET("/tasks", func(c echo.Context) error { return c.JSON(200, "GET Tasks") })
    e.PUT("/tasks", func(c echo.Context) error { return c.JSON(200, "PUT Tasks") })
    e.DELETE("/tasks/:id", func(c echo.Context) error { return c.JSON(200, "DELETE Task "+c.Param("id")) })

    // Start as a web server
    e.Run(standard.New(":8000"))
}
```

以上路由只输出了固定的文本内容，将会在接下来改进。

可以使用postman测试以上接口。

```
$ go build todo.go
$ ./todo
```

运行后，打开postman，输入localhost:8000，选择GET测试"/tasks"，正常可以看到"GET Tasks"。

然后是配置数据库，指定存储文件未"storage.db"，如果不存在程序会自动创建。数据库创建后需要运行数据迁移。

```
// todo.go

import (
	"database/sql"

	"github.com/labstack/echo"
	"github.com/labstack/echo/engine/standard"
	_ "github.com/mattn/go-sqlite3"
)

```

在 main 方法里增加

```
// todo.go
func main() {
	db := initDB("storage.db")
	migrate(db)
}
```

然后需要定义 initDB 和 migrate 方法。

```
// todo.go
func initDB(filepath string) *sql.DB {
	db, err := sql.Open("sqlite3", filepath)

	//检查错误
	if err != nil {
		panic(err)
	}

	//如果open没有报错，但是仍然没有数据库连接，一样要退出
	if db == nil {
		panic("db nil")
	}

	return db
}

func migrate(db *sql.DB) {
	sql := `
	CREATE TABLE IF NOT EXISTS tasks(
		id INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
		name VARCHAR NOT NULL
	);
	`

	_, err := db.Exec(sql)

	//出错退出
	if err != nil {
		panic(err)
	}
}
```

这两个方法用于连接数据库，建表。"initDB"会打开一个db文件或者创建它。如果失败程序会退出。

"migrate"方法运行创建表的SQL。如果失败程序退出。

然后

```
$ go build todo.go
$ ./todo
```
查看效果。

如果打开另一个终端，列出当前目录内容时会发现已经创建了 "storage.db",执行以下命令来确认它确实是个SQLite文件。

```
$ sqlite3 storage.db
```

需要安装此命令。

此命令会给出提示，输入 ".tables" ，可以列出所有的表，输入 ".quit" 退出。

## 处理请求

之前已经创建了与前端交互的接口，现在需要创建或删除任务时给客户端真实的结果。这需要几个方法去完成。

在 "todo.go" 中需要引入新的包。

```
package main
import (
    "database/sql"
    "go-echo-vue/handlers"

    "github.com/labstack/echo"
    "github.com/labstack/echo/engine/standard"
    _ "github.com/mattn/go-sqlite3"
)
```

然后修改路由，使用刚刚创建的handlers包去处理。

```
// todo.go
    e := echo.New()

    e.File("/", "public/index.html")
    e.GET("/tasks", handlers.GetTasks(db))
    e.PUT("/tasks", handlers.PutTask(db))
    e.DELETE("/tasks/:id", handlers.DeleteTask(db))

    e.Run(standard.New(":8000"))
}
```

创建 "hanlders" 目录，在此目录中创建 "tasks.go" ，然后引入以下包。

```
// handlers/tasks.go
package handlers

import (
    "database/sql"
    "net/http"
    "strconv"

    "github.com/labstack/echo"
)
```

下行用于返回JSON。一个使用string类型作为key，任意类型为值的map。

```
// hanlers/tasks.go
type H map[string]interface{}
```

此文件的主要部分就是处理方法。使用db连接作为一个参数，Echo 路由的处理方法还需要一个 Echo.HandlerFunc 接口。



## 原文链接:[create-a-single-page-app-with-go-echo-and-vue](https://scotch.io/tutorials/create-a-single-page-app-with-go-echo-and-vue)