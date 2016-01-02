title: 为hexo加一个时钟小挂件
date: 2015-11-11 14:16:29
tags:  [hexo,time,widget]
---


之前在 [HTML+CSS3再加一点点JS做的一个小时钟](http://segmentfault.com/a/1190000003055672) 看到的这个问题，觉得很好把代码存下来了，今天突发奇想把它放到hexo的新博客上。

### STEP 0

把HTML内容放到新建的模板里，我命名为time.ejs。

		<div class="widget-wrap">
		<h3 class="widget-title">Time</h3>
		<div class="widget time">
			<style id="clock-animations"></style>
			<div class="clock-wrapper">
				<div class="clock-base">
						<div class="clock-indicator">
							<div></div>
							<div></div>
							<div></div>
							<div></div>
							<div></div>
							<div></div>
							<div></div>
							<div></div>
							<div></div>
							<div></div>
							<div></div>
							<div></div>
						</div>
						<div class="clock-hour"></div>
						<div class="clock-minute"></div>
						<div class="clock-second"></div>
						<div class="clock-center"></div>
				</div>
			</div>
		</div>
	</div>

原文中`<style id="clock-animations"></style>`是放在head区域的，为了减小对全局的影响，我拿到这个挂件中了，实际上并没有影响。

这里要注意挂件的代码结构要尽量与主题自带的结构一致。（仅仅是为了好看，和好整理）

当然了，这个文件要放在`layout/_widget`下。

### STEP 1
js脚本

由于脚本内容较少，我就直接放在了script.js中，文件位置再source/js。

	var now = new Date();
  	var second = now.getSeconds();
  	var minute = now.getMinutes();
  	var hour = now.getHours();
  	if (hour > 12) {
      	hour = hour - 12;
  	}
  	hourDeg   = hour * 30 + now.getMinutes() / 60 * 30;
  	minuteDeg = now.getMinutes() * 6;
  	secondDeg = now.getSeconds() * 6;
  	stylesDeg = [
      	"@keyframes rotate-hour{ from{transform:rotate(" + hourDeg + "deg);}to{transform:rotate(" + (hourDeg + 360) + "deg);}}",
      	"@keyframes rotate-minute{from{transform:rotate(" + minuteDeg + "deg);}to{transform:rotate(" + (minuteDeg + 360) + "deg);}}",
      	"@keyframes rotate-second{from{transform:rotate(" + secondDeg + "deg);}to{transform:rotate(" + (secondDeg + 360) + "deg);}}",
      	"@-moz-keyframes rotate-hour{ from{transform:rotate(" + hourDeg + "deg);}to{transform:rotate(" + (hourDeg + 360) + "deg);}}",
      	"@-moz-keyframes rotate-minute{from{transform:rotate(" + minuteDeg + "deg);}to{transform:rotate(" + (minuteDeg + 360) + "deg);}}",
      	"@-moz-keyframes rotate-second{from{transform:rotate(" + secondDeg + "deg);}to{transform:rotate(" + (secondDeg + 360) + "deg);}}",
      	"@-webkit-keyframes rotate-hour{from{transform:rotate(" + hourDeg + "deg);}to{transform:rotate(" + (hourDeg + 360) + "deg);}}",
      	"@-webkit-keyframes rotate-minute{from{transform:rotate(" + minuteDeg + "deg);}to{transform:rotate(" + (minuteDeg + 360) + "deg);}}",
      	"@-webkit-keyframes rotate-second{from{transform:rotate(" + secondDeg + "deg);}to{transform:rotate(" + (secondDeg + 360) + "deg);}}"
  	].join("");
  	$('#clock-animations').html(stylesDeg);
  
 
### STEP 2
CSS样式

新建文件time.styl，路径为`/source/css/_partial/time.styl`

内容略多，主要是给多浏览器写前缀。

	.clock-wrapper {
		position: relative;
		height: 250px;
		width: 250px;
		background-image: linear-gradient(#f7f7f7,#e0e0e0);
		border-radius: 50%;
		box-shadow: 0 10px 15px rgba(0,0,0,.15),0 2px 2px rgba(0,0,0,.2);
	}
	.clock-base {
		width: 250px;
		height: 250px;
		background-color: #eee;
		border-radius: 50%;
		box-shadow: 0 0 5px #eee;
	}
	.clock-indicator {
		z-index: 1;
		position: absolute;
		top: 15px;
		left: 15px;
		width: 230px;
		height: 230px;
	}
	.clock-indicator div {
		position: absolute;
		width: 2px;
		height: 4px;
		margin: 113px 114px;
		background-color: #a4a4a4;
	}
	.clock-indicator div:nth-child(1) {
		transform:rotate(30deg) translateY(-113px);
		-ms-transform:rotate(30deg) translateY(-113px);     /* IE 9 */
		-moz-transform:rotate(30deg) translateY(-113px);    /* Firefox */
		-webkit-transform:rotate(30deg) translateY(-113px); /* Safari 和 Chrome */
		-o-transform:rotate(30deg) translateY(-113px);  /* Opera */        }
		.clock-indicator div:nth-child(2) {
		transform:rotate(60deg) translateY(-113px);
		-ms-transform:rotate(60deg) translateY(-113px);     /* IE 9 */
		-moz-transform:rotate(60deg) translateY(-113px);    /* Firefox */
		-webkit-transform:rotate(60deg) translateY(-113px); /* Safari 和 Chrome */
		-o-transform:rotate(60deg) translateY(-113px);  /* Opera */
	}
	.clock-indicator div:nth-child(3) {
		transform:rotate(90deg) translateY(-113px);
		-ms-transform:rotate(90deg) translateY(-113px);     /* IE 9 */
		-moz-transform:rotate(90deg) translateY(-113px);    /* Firefox */
		-webkit-transform:rotate(90deg) translateY(-113px); /* Safari 和 Chrome */
		-o-transform:rotate(90deg) translateY(-113px);  /* Opera */
		background-color: #5a5a5a;
	}
	.clock-indicator div:nth-child(4) {
		transform:rotate(120deg) translateY(-113px);
		-ms-transform:rotate(120deg) translateY(-113px);    /* IE 9 */
		-moz-transform:rotate(120deg) translateY(-113px);   /* Firefox */
		-webkit-transform:rotate(120deg) translateY(-113px); /* Safari 和 Chrome */
		-o-transform:rotate(120deg) translateY(-113px);     /* Opera */
	}
	.clock-indicator div:nth-child(5) {
		transform:rotate(150deg) translateY(-113px);
		-ms-transform:rotate(150deg) translateY(-113px);    /* IE 9 */
		-moz-transform:rotate(150deg) translateY(-113px);   /* Firefox */
		-webkit-transform:rotate(150deg) translateY(-113px); /* Safari 和 Chrome */
		-o-transform:rotate(150deg) translateY(-113px);     /* Opera */
	}
	.clock-indicator div:nth-child(6) {
		transform:rotate(180deg) translateY(-113px);
		-ms-transform:rotate(180deg) translateY(-113px);    /* IE 9 */
		-moz-transform:rotate(180deg) translateY(-113px);   /* Firefox */
		-webkit-transform:rotate(180deg) translateY(-113px); /* Safari 和 Chrome */
		-o-transform:rotate(180deg) translateY(-113px);     /* Opera */
		background-color: #5a5a5a;
	}
	.clock-indicator div:nth-child(7) {
		transform:rotate(210deg) translateY(-113px);
		-ms-transform:rotate(210deg) translateY(-113px);    /* IE 9 */
		-moz-transform:rotate(210deg) translateY(-113px);   /* Firefox */
		-webkit-transform:rotate(210deg) translateY(-113px); /* Safari 和 Chrome */
		-o-transform:rotate(210deg) translateY(-113px);     /* Opera */
	}
	.clock-indicator div:nth-child(8) {
		transform:rotate(240deg) translateY(-113px);
		-ms-transform:rotate(240deg) translateY(-113px);    /* IE 9 */
		-moz-transform:rotate(240deg) translateY(-113px);   /* Firefox */
		-webkit-transform:rotate(240deg) translateY(-113px); /* Safari 和 Chrome */
		-o-transform:rotate(240deg) translateY(-113px);     /* Opera */
	}
	.clock-indicator div:nth-child(9) {
		transform:rotate(270deg) translateY(-113px);
		-ms-transform:rotate(270deg) translateY(-113px);    /* IE 9 */
		-moz-transform:rotate(270deg) translateY(-113px);   /* Firefox */
		-webkit-transform:rotate(270deg) translateY(-113px); /* Safari 和 Chrome */
		-o-transform:rotate(270deg) translateY(-113px);     /* Opera */
		background-color: #5a5a5a;
	}
	.clock-indicator div:nth-child(10) {
		transform:rotate(300deg) translateY(-113px);
		-ms-transform:rotate(300deg) translateY(-113px);    /* IE 9 */
		-moz-transform:rotate(300deg) translateY(-113px);   /* Firefox */
		-webkit-transform:rotate(300deg) translateY(-113px); /* Safari 和 Chrome */
		-o-transform:rotate(300deg) translateY(-113px);     /* Opera */
	}
	.clock-indicator div:nth-child(11) {
		transform:rotate(330deg) translateY(-113px);
		-ms-transform:rotate(330deg) translateY(-113px);    /* IE 9 */
		-moz-transform:rotate(330deg) translateY(-113px);   /* Firefox */
		-webkit-transform:rotate(330deg) translateY(-113px); /* Safari 和 Chrome */
		-o-transform:rotate(330deg) translateY(-113px);     /* Opera */
	}
	.clock-indicator div:nth-child(12) {
		transform:rotate(360deg) translateY(-113px);
		-ms-transform:rotate(360deg) translateY(-113px);    /* IE 9 */
		-moz-transform:rotate(360deg) translateY(-113px);   /* Firefox */
		-webkit-transform:rotate(360deg) translateY(-113px); /* Safari 和 Chrome */
		-o-transform:rotate(360deg) translateY(-113px);     /* Opera */
		background-color: #5a5a5a;
	}
	.clock-hour {
		z-index: 2;
		position: absolute;
		left: 128px;
		top: 80px;
		width: 4px;
		height: 65px;
		border-radius: 2px;
		background-color: #555;
		box-shadow: 0 0 2px rgba(0,0,0,0.2); 
		transform-origin: 2px 50px;
		-moz-transform-origin: 2px 50px;
		-ms-transform-origin: 2px 50px;
		-o-transform-origin: 2px 50px;
		-webkit-transform-origin: 2px 50px;
		transition: 1s;
		-moz-transition: 1s; /* Firefox 4 */
		-webkit-transition: 1s; /* Safari 和 Chrome */
		-o-transition: 1s; /* Opera */
		animation:rotate-hour 43200s linear infinite;
		-webkit-animation: rotate-hour 43200s linear infinite; /* Safari 和 Chrome */
	}
	.clock-minute {
		z-index: 3;
		position: absolute;
		left: 128px;
		top: 60px;
		width: 4px;
		height: 85px;
		border-radius: 2px;
		background-color: #555;
		box-shadow: 0 0 2px rgba(0,0,0,0.2); 
		transition: 1s;
		-moz-transition: 1s; /* Firefox 4 */
		-webkit-transition: 1s; /* Safari 和 Chrome */
		-o-transition: 1s; /* Opera */
		transform-origin: 2px 70px;
		-moz-transform-origin: 2px 70px;
		-ms-transform-origin: 2px 70px;
		-o-transform-origin: 2px 70px;
		-webkit-transform-origin: 2px 70px;
		animation:rotate-minute 3600s linear infinite;
		-webkit-animation: rotate-minute 3600s linear infinite; /* Safari 和 Chrome */
	}
	.clock-second {
		z-index: 4;
		position: absolute;
		left: 129px;
		top: 15px;
		width: 2px;
		height: 140px;
		border-radius: 2px;
		background-color: #a00;
		box-shadow: 0 0 1px rgba(0,0,0,0.2); 
		transform-origin: 1px 115px;
		-moz-transform-origin: 1px 115px;
		-ms-transform-origin: 1px 115px;
		-o-transform-origin: 1px 115px;
		-webkit-transform-origin: 1px 115px;
		transition: 1s;
		-moz-transition: 1s; /* Firefox 4 */
		-webkit-transition: 1s; /* Safari 和 Chrome */
		-o-transition: 1s; /* Opera */
		animation:rotate-hour 60s linear infinite;
		 -webkit-animation: rotate-second 60s linear infinite;  /* Safari 和 Chrome */
	}
	.clock-second:after {
		content: "";
		display: block;
		position: absolute;
		width: 8px;
		height: 8px;
		border-radius: 50%;
		left: -3px;
		top:110px;
		background-color: #a00;
		box-shadow: 0 0 3px rgba(0,0,0,0.2);
	}
	.clock-center {
		z-index: 1;
		position: absolute;
		left: 55px;
		top: 55px;
		background-image: linear-gradient(#e3e3e3,#f7f7f7); 
		border-radius: 50%;
		width: 150px;
		height: 150px;
		box-shadow: inset 0 -1px 0 #fafafa, inset 0 1px 0 #e3e3e3;
	}
	.clock-center:after{
		content: "";
		display: block;
		width: 20px;
		height: 20px;
		margin: 65px;
		background-color: #ddd;
		border-radius: 50%;
	}
 
 之后，在sidebar.styl 末尾加入 `@import "time"`,这是styl的语法，表示引入一个文件，因为两个文件是同级，所以可以直接这么写，加`./`也可以。
 
### STEP 3
最后，在themes的config文件中注册这个挂件。

在_config.yml中，找到`widgets`，在你想要加的位置中加入 `- time`，即模板文件名。
比如我的widgets变成了这样：

	widgets:
	- clock
	- category
	- tag
	- tagcloud
	- archive
	- recent_posts
	
### LAST 
至此，就全部完事，可以去`hexo generate`或`hexo server`了。