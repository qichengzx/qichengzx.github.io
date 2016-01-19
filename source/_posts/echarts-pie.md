title: ECharts饼图示例
categories: javascript
date: 2016-01-19 21:27:00
tags:  [javascript,php,ECharts,php]

---

![](/images/BEEC6560-EC03-4027-969D-DBB584A47E69.png)

今天写一个做地区分布的饼图的东西。

由于数据库里本身没有存储地区数据，只有IP，所以也用到了[高春辉老师目前的项目](http://www.ipip.net/)里提供的IP地址数据库。可以在数据库下载页面下载到免费的数据库文件和一个PHP处理类。

还用到了百度团队开源的[ECharts](http://echarts.baidu.com/index.html)，很简单很好用。

其实事情本身并没有难度。

### 约定

默认为：此时已有一个静态页面，并且已引入echarts的js文件。

### 先读取数据
```

	$sql = "SELECT id,ips FROM table order by id desc limit 100000";
	
	$pdo = new PDO('mysql:host=127.0.0.1;dbname=db1','root','root');
	
	$query = $pdo -> query($sql);
	$query->setFetchMode(PDO::FETCH_ASSOC);
	$rs = $query->fetchAll();	
	
```

### 根据IP得出所在地信息

```

	include './IP.class.php';//ipip.net提供的IP处理类
	$IP = new IP();
	foreach ($rs as $key => $value) {
		$a = $IP->find($value["ips"]);
		$arr[$key] = $a[1].$a[2]; //正确的IP返回的数据为:Array ( [0] => 中国 [1] => 黑龙江 [2] => 鹤岗 [3] => ) ,这里根据实际情况取对应字段即可。
	}
	
```

### 生成echarts所需的数据

```
	$data = array_count_values($arr);
```

嗯，一个非常简单粗暴的把同名地区数量算出来的方法。这样就生成了如array('北京'=>2,'天津'=>3)这样的数据。正是echarts所需的。

### 饼图区域

```
	<div id="main" style="width: 1400px;height:1400px;"></div>
```

### 正经的JS来了

```
	var myChart = echarts.init(document.getElementById('main'));
	
	var option = {
       
        title : {
	        text: 'IP分布地区',
	        x:'center'
	    },
	    tooltip : {
	        trigger: 'item',
	        formatter: "{a} <br/>{b} : {c} ({d}%)"
	    },
	    legend: {
	        orient: 'vertical',
	        left: 'left',
	        //简单起见此处直接foreach了
	        data: [<?php foreach ($keys as $key => $value) { echo '"'.$value.'",'; }?>]
	    },
	    series : [
	        {
	            name: 'IP所在地区',
	            type: 'pie',
	            radius : '40%',
	            center: ['50%', '60%'],
	            data:[
	            	//同上，
	            	<?php 
	            		//reset($data);
						while (list($key, $val) = each($data)){
							echo '{value:'.$val.',name:"'.$key.'"},';
						}
	            	?>
	            ],
	            itemStyle: {
	                emphasis: {
	                    shadowBlur: 10,
	                    shadowOffsetX: 0,
	                    shadowColor: 'rgba(0, 0, 0, 0.5)'
	                }
	            }
	        }
	    ]
    };
    
    // 使用刚指定的配置项和数据显示图表。
    myChart.setOption(option);

```

打开页面看一下吧。



