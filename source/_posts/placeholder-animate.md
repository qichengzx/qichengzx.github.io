title: 一个简单的input获得焦点时的小动画效果
categories: javascript
date: 2016-01-11 21:42:58
tags:  [javascript,input]

---

以前经常见到的一种页面表单效果，但是并没有在项目中用过，前几天在sf上看到相关问题，顺手写了一个，还挺简单，没什么含量。

此处省略HTML头信息等内容。

### HTML

```
	<div>
	  <input type="text" id="uname"/>
	  <label>Name</label>
	</div>

```

### CSS

```
div{
  position:relative;
  top:20px;
}
input{
  width:200px;
  height:30px;
  line-height:30px;
  padding:0 3px;
  border:none;
  border-bottom:1px solod #666;

}
label{
  position:absolute;
  top:0;
  left:5px;
  color:#ccc;
  height:30px;
  line-height:30px;
}
```

### JS

```
$(document).on('focus','input',function(){
  $(this).siblings('label').animate({top:'-20px'});
})
.on('focusout','input',function(){
  $(this).siblings('label').animate({top:'0'});
})
```

### RESULT

<iframe width="100%" height="300" src="//jsfiddle.net/qichengzx/zc91Lyx7/8/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

### 不足

没有考虑label的鼠标点击事件。