##### Mac 安装 GO

本人使用了Brew来安装。

##### 安装前首先更新Brew。

```
brew update && brew upgrade
```

##### 安装Go

使用brew安装go

```
brew install go
```

##### 设置$GOPATH

Go从1.1版本开始必须设置这个变量，也就是说通过以上方式安装后就必须要设置$GOPATH了。

这个目录用来存放Go源码，可运行文件，和编译之后的包文件。

Mac系统设置

在bash中加入：

```shell
export GOPATH=/your/path
```

以上目录需要存在，不存在就自己手动创建一个。

然后运行

```shell
source ~/.zshrc
```

替换成你自己使用的shell，如.bashrc等。

然后，需要在/your/path目录下，新建 pkg，bin，src目录。

​	src目录存放源代码，如hello.go

​	pkg目录存放编译后的文件，如hello.a

​	bin目录存放生成后的可执行文件

src目录就是主要开发目录。如hello项目，则在src下新建hello目录。

到这里，就可以开始Go之旅了。

##### Hello Go

需要注意的是，在Go中，包名和文件名是可以不同的，包名为main的时候即为可以独立运行的包。

在src目录新建hello目录，创建hello.go文件。

```go
package main //包名

import "fmt" //导入一个系统级别的fmt包

//使用func定义函数
//main函数没有参数，没有返回值
func main() {
  fmt.Println("Hello Go") //调用包函数的方法为 pkgName.funcName
}
```

一个hello.go就写好了，在命令行输入

```go
go run hello.go
```

就可以看到输出Hello Go了。