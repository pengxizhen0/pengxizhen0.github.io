---
title: JS学习笔记之构造函数
categories:
  - web前端
tags:
  - JS构造函数
date: 2017-04-21
---

我们选择new一个函数的时候，会经历以下3个步骤，以`new Foo()`为例：

<!-- more -->

1. 创建一个新对象，继承自`Foo.prototype`
2. 执行构造函数，并将this指向新对象
3. 返回新对象（若构造函数没有返回值，也照样返回新对象）