title: JS标识符
categories: javascript
date: 2015-11-17 22:07:21
tags:  [javascript,标识符]

---


标识符由一个字母开头，其后可以选择性的加上一个或多个字母、数字或下划线。但是，不能使用下面这些保留字：

```
abstract
boolean break byte
case catch char class const continue
debugger default delete do double
else enum export extends
false final finally float for function
goto
if implements import in instanceof int interface 
long
native new null
package private protected public
return
short static super switch synchronized
this throw throws transient true try typeof
var volatile void
while with
```

这个列表里的大部分并未用在JS语言里，有一些已经逐渐在新标准中出现，但是这个列表也不包括一些本应该保留的字，如undefined,NaN,Infinity。

JS不允许使用保留字来命名变量或参数，也不允许在对象字面量中，或者用 点运算符(.)提取对象属性时，使用保留字作为对象的属性名。


标识符被用于语句，变量，参数，属性名，运算符和标记。

注:实际中，JS也允许 (_),($)开头。