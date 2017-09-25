title: Go语言使用redigo操作GEO(3) - 获取范围集合
categories: golang
date: 2017-09-24 18:09:18
tags:  [go,redis,redigo,geo]
---

[上一篇](/2017/09/17/redis-geo-with-redigo-in-golang-part-2.html)记录了使用redigo操作Redis的获取两点距离的功能。

今天继续来写一下，获取一个坐标点指定距离范围内地理位置的集合。

完成此功能，需要使用Redis的 GEORADIUS 命令。

语法：

```shell
GEORADIUS key longitude latitude radius [m|km|ft|mi] [WITHCOORD] [WITHDIST] [ASC|DESC] [WITHHASH] [COUNT count]
```

其中，longitude latitude 表示地理位置的坐标，radius表示范围的距离，单位可以是m，km，ft，mi，依次为米，千米，英里，英尺。

后边的可选参数中：

- WITHCOORD：传入WITHCOORD参数，返回结果会带上匹配位置的经纬度。
- WITHDIST：传入WITHDIST参数，返回结果会带上匹配位置与给定地理位置的距离，距离的单位与 GEORADIUS 命令传入的单位一致。
- ASC|DESC：默认结果是未排序的，ASC表示从近到远排序，DESC表示从远到近排序。
- WITHHASH：传入WITHHASH参数，返回结果会带上匹配位置的hash值。hash为52位有符号整数。
- COUNT count：传入COUNT参数，返回指定数量的结果。

参照文档，可以发现，默认情况下 GEORADIUS 会返回全部匹配的元素，而传入 COUNT 参数会截取指定的部分元素返回，但是由于此命令还需要对返回结果进行排序，所以如果元素较多的情况下，即使使用了 COUNT 参数，查找速度也会很慢。所以此参数仅适用于减小带宽占用，一次性返回少数数据，多次查询。

GEORADIUS 的返回值是一个数组:

- 如果传入了WITHCOORD，WITHDIST，WITHHASH参数，会返回二维的数组，其中每个子数组表示一个元素
- 如果没有传入上述参数，会返回一个一维的数组，值为元素名，如["tianjin","baoding"]

如果返回的是二维数组，子数组的第一个元素是对应位置的名字，其他的会根据传入的参数当做数组的元素返回，其中：

- 距离依然是一个双精度浮点数，单位与传入的单位参数一致。
- GEOHASH 是一个整数。
- 坐标分别为经度，纬度，其中经度在前。

如：

```shell
 GEORADIUS citylist 116.280316 39.9329 200 km WITHCOORD
```
会返回

```shell
1) 1) "beijing"
   2) 1) "116.40528291463851929"
      2) "39.9049884229125027"
2) 1) "baoding"
   2) 1) "115.33530682325363159"
      2) "38.87121760640306434"
3) 1) "tangshan"
   2) 1) "116.94219142198562622"
      2) "39.05078232277295314"
4) 1) "tianjin"
   2) 1) "117.0153459906578064"
      2) "39.12522961794389431"
```

那么，使用redigo如何实现呢？

#### 有WITHCOORD，WITHDIST，WITHHASH参数



#### 无WITHCOORD，WITHDIST，WITHHASH参数

```Go
func radius(key string, lng, lat float64, radius int, unit string) []string {
	rc := RedisClient.Get()
	defer rc.Close()

	pos, _ := redis.Strings(rc.Do("GEORADIUS", key, lng, lat, radius, unit))
	return pos
}

```