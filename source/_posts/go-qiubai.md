title: Go语言写爬取糗百热门帖子
date: 2016-12-04 15:49:04
tags:  [go,糗百,goquery,gin]
---

闲来无事，想着也用Go来写个爬虫之类的东西，我并不知道这算不算严格意义上的爬虫。

思前想后，觉得写个爬糗百热门的脚本吧，一来足够简单，二来大概熟悉下流程。

首先，选了[goquery](https://github.com/PuerkitoBio/goquery)这个包来解析HTML，声称与jquery相似的用法，事实上也确实是这样，非常方便。

定个目标，只爬取列表页的帖子内容，作者和回帖都不管。

```
package main

import (
	"github.com/PuerkitoBio/goquery"
	"log"
)

//定义结构体
type Qb struct {
	Id int `json:"id"`
	Content string `json:"content"`
}

func main() {
	var url = "http://www.qiushibaike.com/hot"

	doc, err := goquery.NewDocument(url)
	if err != nil {
		log.Fatal(err)
	}

	var qb []Qb
	doc.Find("#content-left .article").Each(func(i int, s *goquery.Selection) {
		//s即为当前的 .article 元素，查找下级中的span元素的内容。
		content := s.Find(".content span").Text()
		qb = append(qb, Qb{Id: i, Content: content})
	})

	log.Println(qb)
}
```

"#content-left .article" 即每一条帖子作为元素的class。

将会输出：
```
[
	{0 结婚十三周年那天，老婆望着一大桌子菜不禁泪流满面。我帮她拭去泪水:瞧你，都激动的哭了!老婆却说:我激动个屁!想想这十三年跟着你受的罪，我实在忍不住啊!} 
	{1 前几天天冷，就给妹妹买了条围巾，然后她说谢谢哥，本人本着组织精神说你应该谢谢你嫂子，她惊讶的对我说:哥，你谈女朋友了。我说:没有，你应该感谢她一直到现在都没出现，哥才有钱给你买东西} 
	{2 跟哥们去理发，剪头的是个妹纸。。妹纸:“你有女朋友么？”哥们一听，突然兴奋的说:“没有！”妹纸:“我是个实习生，本来想给你换大工的，看你没有女朋友，我就随意剪了！”哥们你别看我，我就是一口水没忍住，喷你脸上了而已！} 
	{3 老妈比较胖，小时候每次打我我都是撒腿就跑，老妈没一次抓到我的。直到老妈学会骑自行车以后，那鞭子挥得………真像套马杆的汉子，威武雄壮……}
]
```

那么如何展示到页面中呢。

我选择了 [gin](https://github.com/gin-gonic/gin) 框架。

修改一下代码。

```
func main() {
	r := gin.Default()
	r.LoadHTMLGlob("public/*")
	r.GET("/", Index)
	r.Run()
}

func Index(c *gin.Context) {
	var url = "http://www.qiushibaike.com/hot"

	doc, err := goquery.NewDocument(url)
	if err != nil {
		log.Fatal(err)
	}

	var result []Qb
	doc.Find("#content-left .article").Each(func(i int, s *goquery.Selection) {
		content := s.Find(".content span").Text()
		result = append(result, Qb{Id: i, Content: content})
	})

	c.HTML(http.StatusOK, "index.html", gin.H{
		"items": result,
		"title": "糗百热门"
	})
}
```

可以看到，
```
r := gin.Default()
r.LoadHTMLGlob("public/*")
r.GET("/", Index)
```

这里加载了public目录中的模板，然后下一行，表示，接收到 "/" 的请求时，调用Index方法去处理。

到这里，文档的抓取，解析，构造数据就已经完成，下一步，看一下怎么显示到页面中。

```
{% raw %}
<div class="col-md-12">
    <h2>{{ .title }}</h2>
    <table class="table table-striped table-bordered table-hover">
        {{ range $item := .items }}
        <tr>
            <td>{{ $item.Content }}</td>
        </tr>
        {{ end }}
    </table>
</div>
{% endraw %}
```

使用 "{% raw %}{{ }}{% endraw %}" 输出后端发送过来的数据。使用 range 迭代数据。与

```
for pos, char := range str {
...
}
```

一样。

完整的模板代码：

```
{% raw %}
<!-- public/index.html -->

<html>
    <head>
        <meta http-equiv="content-type" content="text/html; charset=utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
        <title>糗百</title>

        <link rel="stylesheet" href="https://cdn.bootcss.com/bootstrap/3.3.6/css/bootstrap.min.css">
        <link rel="stylesheet"  href="https://cdn.bootcss.com/font-awesome/4.6.3/css/font-awesome.min.css">

    </head>
    <body>
        <div class="container">
            <div class="row">
                <div class="col-md-12">
                    <h2>{{ .title }}</h2>
                    <table class="table table-striped table-bordered table-hover">
                        {{ range $item := .items }}
                        <tr>
                            <td>{{ $item.Content }}</td>
                        </tr>
                        {{ end }}
                    </table>
                </div>
            </div>
        </div>
    </body>
</html>
{% endraw %}
```

这样，运行一下，就可以了。

gin框架默认使用8080端口，打开http://localhost:8080就可以看到一个极简版的糗百热门了。

问题来了，怎么增加一个分页呢？

完整代码见:

[Github地址](https://github.com/qichengzx/goqiubai)

#### 后记

其实早就写完了这篇，但是hexo生成的时候由于 ["{% raw %}{{{% endraw %}"的问题](https://hexo.io/docs/troubleshooting.html#Escape-Contents)，生成一直失败，一直拖到现在。

实际代码中需要去掉 "{ % raw % }" 相关部分。