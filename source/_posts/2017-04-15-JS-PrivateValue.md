---
layout: post
title: JS学习笔记——私有变量
description: 私有变量
date: 2017-04-15
---

## 0.本章提要
### 使用立即执行函数实现块级作用域
### 使用`构造函数`实现**实例**私有变量：私有变量函数和公有特权方法每个实例都有独立的一份
### 使用`原型模式+块级作用域`实现**静态**私有变量：私有变量函数和公有特权方法每个实例都共享一份

## 1.块级作用域
javascript不像其他语言有块级作用域，比如C、java的块级作用域。
```java
//C、java块级作用域
for(int i = 0; i < N; i++){...}
//i无效
```
javascript是以函数进行区分作用域的（具体可参考JS学习笔记——作用域链）。如果需要使用到像java一样的块级作用域，可以用函数的形式实现。
```javascript
(function(){
    var i;
    ...
})();
```
这个函数是立即执行函数，变量i的作用域仅限于匿名函数内，就不会对全局造成污染。而且该立即执行函数没有引用，一经执行完，作用域链就可以立即销毁。

## 2.实例私有变量
javascript没有像java一样的对象成员访问权限的设置（private、public等）。如果要实现私有变量，可以通过构造函数的方式。下面是一种实现实例私有变量的方法。
```javascript
function Person(name) {//name私有变量
    var privateVar;//私有变量
    function privateFunc(){...}//私有函数
    this.getName() {//特权方法
        return name;
    }
    this.setName(value) {//特权方法
        name = value;
    }
}

var person = new Person("Season");
```
将需要私有的变量和函数放到函数作用域中，外界就认为是私有的了，但我们需要一种手段去访问它们，于是`特权方法(privileged method)`就出现了，它是有权访问私有变量和函数的公有方法。
私有变量或函数使用表达式定义时，一定要加`var`，不然这个变量或函数就会被加到全局作用域中，就不是私有变量了。其中函数也可以使用函数声明，因为函数声明只会提升到当前作用域，不会到全局作用域中。
我们使用`this`把公有的特权方法定义成对象的成员，可供函数作用域外的变量访问。
任何函数都可以被用作构造函数，只要使用了new操作符。`Person()`函数中的私有变量没有赋值给this对象，那么这些私有变量就是局部变量，通过闭包的方式被特权方法访问，被放入到其作用域链中。所以每次new之后，特权方法和私有变量都会被再复制一份，前者属于this对象，当然会复制一份；后者通过闭包被加入到特权方法的作用域链中，也会复制一份。

## 3.静态私有变量
使用`原型模式+块级作用域`实现**静态**私有变量
```
(function(){
    var privateVar = 0;
    function privateFunc(){return false;}
    MyObject = function(){};
    MyObject.prototype.publicMethod = function(){
        privateVar++;
        return privateFunc();
    }
})();

var Obj = new MyObject();
```
这个立即执行的匿名函数不会随着new的执行而执行，因此私有变量和函数只有一份，它们的作用域仅限于匿名函数内。`MyObject`定义的时候不能加`var`，否则匿名函数外就访问不到了。特权方法定义在`MyObject`的原型上，因此特权方法也只有一个。
