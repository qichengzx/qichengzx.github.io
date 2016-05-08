title: 解决Discuz无法发布爱奇艺视频的问题
categories: javascript
date: 2016-05-08 18:57:24
tags:  [js,discuz,爱奇艺]
---


![](/images/discuz/201510148054_136.jpg)

最近碰到需要在Discuz论坛中插入爱奇艺视频的问题，之前没关注过，搜索后有些答案说DZ不支持爱奇艺，有些说爱奇艺不支持DZ，并没有真正能解决问题的。

下午突然想到也许是DZ根据粘贴进来的flash地址生成的标签代码不对，试验后发现果然是这个原因。

打卡/static/js/editor.js 文件第1299行查看这段代码：

```js
case 'vid':
	var mediaUrl = $(ctrlid + '_param_1').value;
	var auto = '';
	var posque = mediaUrl.lastIndexOf('?');
	posque = posque === -1 ? mb_strlen(mediaUrl) : posque;
	var ext = mediaUrl.lastIndexOf('.') === -1 ? '' : mediaUrl.substring(mediaUrl.lastIndexOf('.') + 1, posque).toLowerCase();
	ext = in_array(ext, ['mp3', 'wma', 'ra', 'rm', 'ram', 'mid', 'asx', 'wmv', 'avi', 'mpg', 'mpeg', 'rmvb', 'asf', 'mov', 'flv', 'swf']) ? ext : 'x';
	if(ext == 'x') {
		if(/^mms:\/\//.test(mediaUrl)) {
			ext = 'mms';
		} else if(/^(rtsp|pnm):\/\//.test(mediaUrl)) {
			ext = 'rtsp';
		}
	}
	var str = '[media=' + ext + ',' + $(ctrlid + '_param_2').value + ',' + $(ctrlid + '_param_3').value + ']' + squarestrip(mediaUrl) + '[/media]';
	insertText(str, str.length, 0, false, sel);
	break;
```


Discuz根据主流的视频网站的视频地址格式写的规则，生成discuz专用的[media]标签，在前台输出的时候再解析成embed这样的HTML代码。

解析之后的差不多就是这样：

```html
<embed src="http://player.video.qiyi.com/7b42a1a27ff121c201ee5e6c6d757817/0/0/v_19rrklq2bs.swf-albumId=406283300-tvId=406283300-isPurchase=2-cnId=1" allowFullScreen="true" quality="high" width="480" height="350" align="middle" allowScriptAccess="always" type="application/x-shockwave-flash"></embed>
```

而不那么主流的爱奇艺的flash地址则是：”http://player.video.qiyi.com/7b42a1a27ff121c201ee5e6c6d757817/0/0/v_19rrklq2bs.swf-albumId=406283300-tvId=406283300-isPurchase=2-cnId=1”，查看上一段代码可以看到，discuz会用正则去看粘贴的地址最后一个"."后的后缀，如果这个后缀不在自己的已知flash格式数组中，就把类型设为"x"，也就是生成的标签成了[media=x,500,350]。

再到了前台解析，不认识，直接生成一个a标签。

所以，我的解决办法是：

```js
if(ext == 'x') {
	if(/^mms:\/\//.test(mediaUrl)) {
		ext = 'mms';
	} else if(/^(rtsp|pnm):\/\//.test(mediaUrl)) {
		ext = 'rtsp';
	} else if (mediaUrl.indexOf('player.video.qiyi.com')) {
  		ext = 'swf';	
	}
}
```

增加一个else if 判断是否包含爱奇艺播放器的域名，当用户粘贴的flash地址包含这个字符串时，就认为是粘贴了一个爱奇艺的视频，存储的格式也就成了[media=swf]。

但是这里其实不是十分严谨，如果一个非法的flash地址很巧合的包含了这个字符串，也会认为是flash了，那这种情况下就会出错了。