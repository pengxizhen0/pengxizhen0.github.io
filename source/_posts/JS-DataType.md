---
title: JS学习笔记之数据类型
categories:
  - web前端
tags:
  - JS数据类型
date: 2017-04-13
---

## 1.数据类型
ECMAScript一共有6种数据类型：**5种**基本数据类型和**1种**引用数据类型。
基本数据类型：Number\String\Boolean\Null\Undefined
引用数据类型：Object

<!-- more -->

## 2.tyepof操作符
typeof是操作符，不是函数。typeof返回**字符串**。使用typeof返回的数据类型跟第1节是不一样的。typeof一共返回6种，也是6种，只不过是**4种**基本数据类型和**2种**引用数据类型。Null被归为Object，原本属于Object的Function被单独拎出来。

基本数据类型：Number\String\Boolean\Undefined
引用数据类型：Object\Function

## 3.注意事项

### 3.1 声明变量时显式初始化
已声明未赋值变量`myTest`与未声明变量`test`的值都是`undefined`。这是两个本质不一样的变量，typeof之后的值却是一样的，我们应该避免这种情况的出现：声明变量的时候显式地初始化变量，让上述`typeof(test)`的情况不要出现，所以只要出现undefined，我们就可以认为这是一个未声明的变量。

```
//var test;
var myTest;
console.log(typeof(test));//undefined
console.log(typeof(myTest));//undefined
```


### 3.2 对象变量显式初始化为null
如果声明的变量将来要用来保存对象的，应该初始化成`null`(之前我一直初始化成`{}`，这样是不好的)

### 3.3 undefined\null是关键字 可当变量用
我们可以显式地把变量初始化为undefined`var test = undefined`，表示把test赋值成基本数据类型中的undefined。这里不是把undefined字符串赋值给变量，`undefined`是关键字，所以可以当做变量来用，如果写`var test = myundefined`是会报错的。js会认为myundefined是变量，然后去找，发现未定义。如果myundefined加上引号就是String，赋值就不会报错了。

