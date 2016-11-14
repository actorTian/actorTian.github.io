---
title: 破而后立之js对象的创建
date: 2016-11-14 21:17:25
tags: javascript
category: 技术
---
### javascript对象的创建方法
1. 使用内置对象创建
2. 使用json符号
3. 自定义对象构造

#### 使用内置对象创建
1. JavaScript语言原生对象（语言级对象），如String、Object、Function等； 
2. JavaScript运行期的宿主对象（环境宿主级对象），如window、document、body等。 

```javascript
var str   = new String("实例初始化String");
var str_o = "直接赋值的String";
var func  = new Function("x","alert(x)");//示例初始化func
var o     = new Object();//示例初始化一个Object 
```
#### 使用json符号
```javascript
var _obj = {name:'2', init: function(){
    console.log('debug');
}};
```
#### 自定义构造对象
创建高级对象有两种方式: this关键字和原型prototype
```javascript
function Girl(){
    this.name = '二';
}
Girl.prototype.init = function(){
    console.log("i'm coming");
}
```




