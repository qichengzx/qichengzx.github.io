title: MySQL实现按经纬度做距离排序
categories: mysql
date: 2017-06-27 22:18:19
tags: [mysql]

---

![](/images/geolocation.png)
题图来自网络

工作中某些业务需要用到按距离排序返回结果，之前的方式是根据前端传过来来的经纬度，和指定范围的距离，算出一个坐标区间，再用这个区间的值去MySQL中查找，类似“where lat between (lat1, lat2) and lng between (lng1,lng2)”，查出数据后，再遍历数据计算每一条数据到这个经纬度的距离，然后根据得出的距离排序返回。低效，麻烦，不方便分页。

于是决定直接从MySQL中算出距离后返回，省事，方便，还可以直接分页了。

查资料后发现还挺简单的，下方的示例是从[Google官方的文档](https://developers.google.com/maps/articles/phpsqlsearch_v3)中摘取出来。

创建如下数据表：

```mysql
CREATE TABLE `markers` (
  `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY ,
  `name` VARCHAR( 60 ) NOT NULL ,
  `address` VARCHAR( 80 ) NOT NULL ,
  `lat` FLOAT( 10, 6 ) NOT NULL ,
  `lng` FLOAT( 10, 6 ) NOT NULL
) ENGINE = MYISAM ;
```

填充数据：

```mysql
INSERT INTO `markers` (`name`, `address`, `lat`, `lng`) VALUES ('Frankie Johnnie & Luigo Too','939 W El Camino Real, Mountain View, CA','37.386339','-122.085823');
INSERT INTO `markers` (`name`, `address`, `lat`, `lng`) VALUES ('Amici\'s East Coast Pizzeria','790 Castro St, Mountain View, CA','37.38714','-122.083235');
INSERT INTO `markers` (`name`, `address`, `lat`, `lng`) VALUES ('Kapp\'s Pizza Bar & Grill','191 Castro St, Mountain View, CA','37.393885','-122.078916');
INSERT INTO `markers` (`name`, `address`, `lat`, `lng`) VALUES ('Round Table Pizza: Mountain View','570 N Shoreline Blvd, Mountain View, CA','37.402653','-122.079354');
INSERT INTO `markers` (`name`, `address`, `lat`, `lng`) VALUES ('Tony & Alba\'s Pizza & Pasta','619 Escuela Ave, Mountain View, CA','37.394011','-122.095528');
INSERT INTO `markers` (`name`, `address`, `lat`, `lng`) VALUES ('Oregano\'s Wood-Fired Pizza','4546 El Camino Real, Los Altos, CA','37.401724','-122.114646');
```

下面，开始从表中查询数据。

根据latitude，longitude值，基于[Haversine公式](http://en.wikipedia.org/wiki/Haversine_formula)从表中查询数据。

假设我们要查询latitude=37.38714,longitude=-122.083235，范围在25英里内的前20条数据，可以这样：

```mysql
SELECT id, ( 3959 * acos( cos( radians('37.38714') ) * cos( radians( lat ) ) * cos( radians( lng ) - radians('-122.083235') ) + sin( radians('37.38714') ) * sin( radians( lat ) ) ) ) AS distance FROM markers HAVING distance < 25 ORDER BY distance LIMIT 0, 20;
```

如果想使用“公里”代替“英里”，将3959换成6371即可。

特别简单。

参考资料：

[Creating a Store Locator with PHP, MySQL & Google Maps](https://developers.google.com/maps/articles/phpsqlsearch_v3)

[Geo/Spatial Search with MySQL](https://zh.scribd.com/presentation/2569355/Geo-Distance-Search-with-MySQL)