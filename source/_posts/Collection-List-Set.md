---
title: Collection_List_Set
categories:
  - Java知识
tags:
  - JDK源码
date: 2019-12-19 21:52:02
---

# Collection、List与Set接口

Collection代表一群元素的集合，至于这群元素怎么排列、是否可重复、是否有序等特性，Collection接口没有作定义。

List和Set继承了Collection接口，他们都具体描述了这群集合元素的特性。

List描述了这群元素是有序且允许重复元素的，用户可通过序号精准地访问到元素。所以List比Collection多的方法，大多跟index有关，比如`void add(int index, E e)`。

Set描述了这群元素是无序且没有重复元素的，Set没有扩展Collection的方法，方法与Collection一样。