title: Go语言使用redigo操作GEO(2) - 获取两点距离
categories: golang
date: 2017-09-23 18:22:01
tags:  [go,redis,redigo,geo]
---

[上一篇](/2017/09/17/redis-geo-with-redigo-in-golang-part-1.html)简单记录了在Go中使用redigo这个包操作Redis，简单的使用了添加和查询。

今天来试一下计算距离。之前也写到过在[MySQL中计算坐标的距离，及排序](/2017/06/27/order-by-distance-in-mysql.html)。

在上一篇中疏忽了一个问题，使用的几个坐标点是用高德地图获取到的，但是Redis中GEO是使用了 WGS84 坐标系的。实际使用中需要注意一下，这里有一篇文章详细介绍了不同坐标系的区别。

#### 添加测试数据

先添加几条测试用的数据。

```Go
push("citylist", "tianjin", "39.1252291", "117.0153461")
push("citylist", "tangshan", "39.0507819", "116.9421939")
push("citylist", "baoding", "38.8712164", "115.3353061")
```

添加了三个城市。

#### 查询距离

使用GEODIST方法查询两点的距离。手册[见此](http://redisdoc.com/geo/geodist.html)。

语法： GEODIST key member1 member2 [unit]

第四个参数可选,可选值为：

- m 表示单位为米。
- km 表示单位为千米。
- mi 表示单位为英里。
- ft 表示单位为英尺。

默认是米。

简单写一个方法，传入key和要查询的两点，及距离单位。

```Go
func dist(key, m1, m2, unit string) float64 {
	rc := RedisClient.Get()
	defer rc.Close()

	rs1, _ := redis.Float64(rc.Do("GEODIST", key, m1, m2, unit))

	return rs
}
```

GEODIST返回值是双精度浮点数，这里直接使用redigo包的Float64方法转换返回了。

#### 使用
```Go
rs1 := dist("citylist", "beijing", "tianjin", "km")
rs2 := dist("citylist", "beijing", "tangshan", "km")
rs3 := dist("citylist", "beijing", "baoding", "km")

fmt.Println("北京-天津", rs1)
fmt.Println("北京-唐山", rs2)
fmt.Println("北京-保定", rs3)
```