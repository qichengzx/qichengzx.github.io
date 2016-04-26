title: Discuz无法发布爱奇艺视频的bug
date: 2016-04-26 18:53:01
tags:  [js,discuz,爱奇艺]
---



/static/js/editor.js 文件第1299行代码段：
```
`case 'vid':
```
`
中，Discuz默认根据主流的视频网站的flash地址写的规则，生成discuz专用的[media]标签，在前台输出的时候再解析成embed这样的HTML代码。

解析之后的差不多就是这样：

```
`\<embed src="http://player.video.qiyi.com/9088e1bbcf9521211bef6cc499d18793/0/0/v_19rrlotgoc.swf-albumId=475960000-tvId=475960000-isPurchase=0-cnId=25" allowFullScreen="true" quality="high" width="480" height="350" align="middle" allowScriptAccess="always" type="application/x-shockwave-flash"\>\</embed\>
```
`
而不那么主流的爱奇艺的flash地址则是：”http://player.video.qiyi.com/9088e1bbcf9521211bef6cc499d18793/0/0/v_19rrlotgoc.swf-albumId=475960000-tvId=475960000-isPurchase=0-cnId=25”，discuz的编辑器一看最后一个”.”之后的名字，不在自己的已知flash格式数组中，也就是下边这个方法：
```
`ext = in_array(ext, ['mp3', 'wma', 'ra', 'rm', 'ram', 'mid', 'asx', 'wmv', 'avi', 'mpg', 'mpeg', 'rmvb', 'asf', 'mov', 'flv', 'swf'](#)) ? ext : 'x';
```
`
那么，返回’x’，所以也就是前台在插入爱奇艺flash时，生成的标签是[media=x,500,350]，这样了。

再到了前台解析，不认识，直接生成一个a标签。

所以，我的解决办法是，在
```
`else if(/^(rtsp|pnm):///.test(mediaUrl)) 
```
`
后增加一个判断：

```
`} else if (mediaUrl.indexOf('player.video.qiyi.com')) 
ext = 'swf';
}
```
`
这里用了个取巧不严谨的办法，当用户粘贴的flash地址包含这个字符串时，就认为是粘贴了一个爱奇艺的视频，存储的格式也就成了[media=swf]。