---
title: JS学习笔记之对象属性判断
categories:
  - web前端
tags:
  - JS对象判断
date: 2017-04-12
---

## 1. 判断对象是否为空
我们可以使用`fon-in`语句来枚举对象的属性，属性被枚举是没有顺序的。使用for-in语句就可以判断对象是否为空，`for-in`语句还会枚举对象**原型**上的属性。当对象是`null`或者`undefined`时，函数`isEptObj()`也返回true，表示对象是空的。

<!-- more -->

```javascript
function isEptObj(o) {
    for(var t in o) {
        return !1;
    }
    return !0;
}
s1 = null;
s2 = undefined;
s3 = [];
s4 = {};
isEptObj(s1);//true
isEptObj(s2);//true
isEptObj(s3);//true
isEptObj(s4);//true

Object.prototype.age = 24;
isEptObj(s1);//true
isEptObj(s2);//true
isEptObj(s3);//flase
isEptObj(s4);//flase
```

## 2. 判断对象是否包含某属性
使用对象的`hasOwnProperty()`方法可以判断属性是否在实例上。如果该属性不在实例上，会有两种情况：1.该属性在原型上； 2.该属性不在原型上。所以我们还要配合`in`语句（`for-in`的非循环版本）继续判断该属性，进而可以得出该属性在实例上，在原型上，还是都不在。
```javascript
s1 = {};
s1.name = "abc";
Object.prototype.age = 24;
console.log(s1.hasOwnProperty('age')); // flase
console.log(s1.hasOwnProperty('name')); // true
console.log(s1.hasOwnProperty('salary')); // flase
```

## 3. 总结
配合使用`in`语句和`hasOwnProperty()`函数，可以判断属性在实例上，在原型上，还是都不在。


