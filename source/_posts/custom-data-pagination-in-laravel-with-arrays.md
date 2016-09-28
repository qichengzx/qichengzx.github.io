title: Laravel 手动创建分页
categories: php
date: 2016-09-28 15:43:13
tags:  [php,laravel]

---

有些情况下会从接口读取数据，数据较多时会用到分页，Laravel为这种需求提供了很方便的方法。

[官方文档](https://laravel.com/docs/5.2/pagination)里几句略过，并没有详细说明，经过查找资料，发现如下方法可行。

### 首先use LengthAwarePaginator

```php
use Illuminate\Pagination\LengthAwarePaginator;
use Illuminate\Support\Collection;
```

假设原内容是：

```php
$result = [
    'item1',
    'item2',
    'item3',
    'item4',
    'item5',
    'item6',
];
```

对于一个列表来说，item一般会是个array，这里忽略。

### 情况1，已知总数，只有部分数据
由于本人所使用的接口有页码和每页数量的参数，所以每次查询返回的其实就是每一页的内容了，而接口又返回了符合条件的总数count，所以使用如下方式即可：

```php
$perPage = 10;
$count = 100;//假设这里是接口返回的总数
//创建collection
$collection = new Collection($data);

$currentPageResults = $collection->all();

//生成分页
$data = new LengthAwarePaginator($currentPageResults, $count, $perPage);
//设置分页的链接
$data->setPath($request->url());
```

### 情况2，未知总数，有全部数据
而如果$data是全部数据呢，比如100条数据全部返回，然后要生成一个每页10条记录的分页，可以这样做：

```php
//获取当前页码
$currentPage = LengthAwarePaginator::resolveCurrentPage();

//从数组创建一个laravel collection
$collection = new Collection($searchResults);

//设置每页数量
$perPage = 10;

//从collection分割数据
$currentPageSearchResults = $collection->slice($currentPage * $perPage, $perPage)->all();

//生成分页
$paginatedSearchResults= new LengthAwarePaginator($currentPageSearchResults, count($collection), $perPage);
//设置分页的链接
$data->setPath($request->url());
```

### 区别

其实两者只是相差了一次分割数据。

### 最后

在视图里依然使用

```
{!! $data->render() !!}
```

输出分页组件。

看起来还挺简单的。

### 参考链接：

[官方文档pagination](https://laravel.com/docs/5.2/pagination)

[Custom data pagination with Laravel 5](http://psampaz.github.io/custom-data-pagination-with-laravel-5/)

[CUSTOM PAGINATION VIEW IN LARAVEL 5 WITH ARRAYS](http://www.acekyd.com/2016/02/28/custom-pagination-view-in-laravel-5-with-arrays/)